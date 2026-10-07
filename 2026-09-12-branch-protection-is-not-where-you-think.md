---
layout: post
title: "Your branch protection is not where you think it is"
date: 2026-09-12 01:58:00 -0400
categories: DevOps
---

# Your branch protection is not where you think it is

Let me start by conceding the thing that makes most posts like this wrong.

Platform vendors are, on the whole, careful about this. GitLab maintains nearly
380 machine-readable deprecation files, over a hundred of them recording impact,
scope and the role that has to act, gave 27 months of runway on its CI job-token
removal, and publishes a breaking-changes guide about a month before each major
release. GitHub says so when a new default is not retroactive — its February 2023
token change reads *"This change will not impact any existing enterprises,
organizations or repositories."* Azure DevOps scopes its new defaults the same
way, to organisations created after the change. When GitHub actually retired a
protection primitive, tag protections, it gave three months' notice, had shipped
a migration tool seven months before that notice, ran three escalating API
brownouts, and auto-migrated anything left behind. Rule Insights went GA in
August 2026 and shows ruleset evaluations across an organisation, with the most
active bypassers ranked.

Most platform changes make things stricter, not looser. And most posture loss in
real organisations is self-inflicted: somebody renamed a job, somebody left a
ruleset in Evaluate mode, somebody added themselves to a bypass list.

So this is not a post about vendors being careless.

It is a post about something narrower and, I think, more awkward: **the thing that
is actually enforced on your repositories is a function of several inputs, and
your configuration is only one of them.** The others are not recorded in the
config you export as evidence, and several of them are not visible in any single
place at all.

I went looking for examples, found about fifty, and then tested a handful myself
because I did not believe the API could be as unhelpful as it looked.

## Nothing in your config changed, and the control changed anyway

Three from the corpus, because they are the cleanest.

**GitHub, 6 June 2023.** The changelog is subtitled, by GitHub,
*"Announcing important changes to what it means for a pull request to be
'approved'."* Two quotes:

> Previously it was possible for a branch protected by "required pull request
> review" to be merged without an approved PR. This was possible because approvals
> were gathered across multiple independent pull requests if the feature branches
> pointed to the same commit as well as targeting the same branch.

> Previously, it was possible for users to make unreviewed changes to a protected
> branch, by creating the merge commit locally and pushing it to the server. The
> merge would be accepted, so long as the parents commits were set correctly.

The fix is good. The interesting part is the implication for anything you have
already asserted. If you told an auditor, a customer, or a questionnaire that
changes to `main` have required peer review since 2019, that claim is weaker than
you thought for every period before June 2023 — and GitHub has said so in writing.
Your `required_pull_request_reviews` JSON is byte-identical before and after.
Rollout was gradual, so there is no date you can put on your own repository. The
only user-visible signal was approvals quietly disappearing.

**Azure DevOps, 14 September 2023.** The cleanest pure loosening I found:

> Previously in, the code coverage policy status was overridden to 'Failed' if
> your build in PR was failing. This was a blocker for some of you who had the
> build as an optional check and the code coverage policy as a required check for
> PRs resulting in PRs being blocked. With this sprint, the code coverage policy
> won't be overridden to 'Failed' if the build fails. **This feature will be
> enabled for all customers.**

(The "Previously in," is Microsoft's, not mine.) Retroactive, all existing
repositories, no opt-out. A repo configured build-optional plus coverage-required
blocked merges on a failing build before this, and did not after. An
`az repos policy` dump shows no change. A policy-as-code diff shows no change. A
REST snapshot shows no change. The only thing that would have caught it is opening
a pull request with a failing build and checking whether you could still merge.

**GitLab 15.0, May 2022.** From GitLab's own deprecations documentation, flagged
as a breaking change and announced in 14.8:

> Upon its removal, push rules will supersede Code Owners. **Even if Code Owner
> approval is required, a push rule that explicitly allows a specific user to push
> code supersedes the Code Owners setting.**

CODEOWNERS is identical. The protected-branch configuration is identical. The push
rule is identical. Only the *interaction between them* changed, and no single
artifact in your repository records an interaction.

## Then I tried to read the current state, and that was worse

Here is where I stopped collecting changelog entries and started testing, because
I assumed I was missing an endpoint.

I created a repo, put classic branch protection on `main` requiring one approval,
and added a repository ruleset on the same branch requiring two. Both active. Then
I asked GitHub what protects that branch.

```
GET /repos/{owner}/{repo}/branches/main/protection
  → required_approving_review_count: 1

GET /repos/{owner}/{repo}/rules/branches/main
  → pull_request, required_approving_review_count: 2
```

Two endpoints, two different answers, and neither mentions the other. The classic
endpoint does not know rulesets exist. The rules endpoint returns only the
ruleset's rule.

So I deleted the ruleset, leaving a branch genuinely protected by classic rules
requiring one approval, and asked the "rules for this branch" endpoint again:

```
GET /repos/{owner}/{repo}/rules/branches/main
  → []
```

An empty array. On a protected branch.

That is the finding I would most like someone to tell me I have wrong. As far as I
can establish, **there is no single GitHub endpoint that reports a branch's
effective protection.** You have to call both, know that they are disjoint, and
merge them yourself using GitHub's documented precedence — which is
most-restrictive-wins, and which I could not test properly because it needs two
approving identities.

A tool reading only the classic endpoint 404s on a ruleset-only repository. A tool
reading only the rules endpoint reports "unprotected" on a branch protected the
old way. Both failure modes are silent, and they point in opposite directions.
(Renovate, for one, gets this right: it checks rulesets first and falls back to
classic protection.)

In August 2026 GitHub added a "Convert to ruleset" button that maps a classic rule
into an equivalent ruleset. It is per rule and opt-in, and the announcement does
not retire classic protection, so until every rule on every branch is converted,
both sources still have to be read.

This also explains something I had been puzzled by. Compliance automation
platforms are inconsistent here in ways that look careless until you try it
yourself: one reads approval counts from both sources and takes the maximum,
another takes the stricter of the two, one supports rulesets only on certain plans,
one has no documented ruleset support at all. That is not laziness. The API makes
the correct answer require two calls and a precedence rule.

## Three more things I confirmed by hand

**`bypass_mode: exempt` does not appear in the rules view.** GitHub's own OpenAPI
spec says of it: *"rules will not be run for that actor and a bypass audit entry
will not be created."* Enforcement off, and no audit record when it happens. I
added an exempt bypass for the repository admin role and re-read the branch's
rules. The response still reports `required_approving_review_count: 2`, and
contains no occurrence of `bypass` or `exempt` anywhere. To discover it you need a
separate, admin-scoped call to the ruleset object.

I had expected the rules endpoint to filter by the caller's own bypass status.
It does not — exempt, always, and no-bypass-at-all all report identically. That
makes it cleaner than I thought, and worse: the view that looks like "what
protects this branch" omits who does not have to obey it.

For what it is worth, across the six rulesets articles in GitHub's documentation
(as of October 2026), `exempt` never appears as a bypass mode — its one occurrence
is about file paths — while `bypass` appears up to 22 times in a single article.

**On a free organisation, making a repository private makes its protections
unreadable.** Both endpoints return HTTP 403:
*"Upgrade to GitHub Pro or make this repository public to enable this feature."*
The configuration still exists. It is neither readable nor enforced. If your
scanner treats 403 as "skip", it reports nothing wrong. If it treats 403 as "not
protected", it reports a failure that is not the one that is actually happening.

**On a free organisation, changing repository visibility destroys classic branch
protection.** This one I reproduced twice and still find slightly hard to
believe. Before: one required approval, `enforce_admins` on, one required status
check. Toggle the repo to private and back to public, touching nothing else.
After: `404 Branch not protected`. Gone. Rulesets survive the same toggle
untouched.

In fairness: a free-tier private repository cannot hold branch protection, so
GitHub arguably has to drop it on the way in. But it does not restore it on the
way out, it does not warn you, and the action that triggered it — changing
visibility — has nothing to do with branch protection.

*Update, 2026-10-07:* on a paid plan it does not reproduce. On a GitHub Team
organisation, the same protection (one approval, `enforce_admins` on, one
required check) survived the private-and-back round trip twice, and stayed
readable while the repository was private. So this is the free tier's forced
removal on the way to private, never restored on the way back. It is not a
general property of visibility changes.

## So what is the actual claim

Not "vendors silently break your posture." I opened by conceding why that is
unfair, and it is.

The claim is narrower:

> **What is actually enforced on your repositories is a joint function of vendor
> semantics, tenant vintage, plan and licence state, and bypass configuration.
> None of those four is recorded in the configuration you publish as evidence, and
> the platform's own readout is not guaranteed to show you when any of them
> changes.**

Tenant vintage is the one people miss. GitHub's `GITHUB_TOKEN` read-only default
arrived in February 2023 and explicitly did not apply to existing organisations —
so organisations created before then remain read/write unless someone changed the
setting, while every document says the default is read-only. Azure DevOps scoped pipeline repository access by
default for *organisations created after May 2020*. GitLab 18.0 turned on CI
job-token allowlists and, on GitLab.com, populated project allowlists from
observed traffic — an allowlist nobody wrote. In all three cases two organisations
with identical exported configuration have different enforcement, and the only
thing that distinguishes them is a signup date.

Plan state is the one that is easiest to trip over by accident. Rulesets are not
available on private repositories under GitHub Free. GitHub Advanced Security
controls stop applying when licences lapse. Bitbucket's enforced merge checks are
Premium-only, and below that *"we'll warn users when they have unresolved merge
checks, but they'll still be able to merge."* Identical configuration, enforcement
off, no event anywhere.

And one more that is not a change at all, just a permanent property worth knowing:
GitHub's documentation says a job that is skipped *"will report its status as
"Success". It will not prevent a pull request from merging, even if it is a
required check."* Adding an `if:` to a
job — in a pull request, not in settings — makes a required check pass vacuously,
while branch protection still shows it as required. Skipping the whole *workflow*
blocks. Skipping the *job* passes.

## What I would actually do about it

Less than you might expect, and none of it is novel.

**Read both endpoints.** If you have anything that asserts branch protection —
a compliance control, an internal scorecard, a script — make sure it reads classic
protection and rulesets and merges them. If it reads one, it is wrong on some of
your repositories right now.

**Check your bypass lists, and look specifically for `exempt`.** It is not in the
rules view and it suppresses its own audit trail.

**Treat "could not read" as a finding, not a pass.** This is the one that changed
how I think. A 403 from a plan restriction, an admin-gated field your token cannot
see, a ruleset endpoint that returns empty — these are all "I do not know", and
the honest report is `unknown`, not `compliant`. I had written a little fleet
scanner that scored repositories out of 1.0, and it turned out that a repository
where *nothing could be observed* scored a perfect 1.0, because I had excluded
unreadable checks from the denominator. Absence of evidence came out as compliance.
I have since split the number in two: a score over what was conclusively observed,
and a separate coverage figure for how much could be observed at all. If you have
built anything similar, go and check what yours does when it cannot see.

**Probe behaviour, not settings, for the things that matter most.** The Azure
DevOps coverage change is invisible to every configuration read and obvious to one
pull request with a failing build.

The small tool I keep poking at this with is
[baseliner](https://github.com/baselinerhq/baseliner) — a single binary that
scores a fleet of repositories against a policy you write. It does not yet check
any of the four things above, which is rather the point of this post: I went
looking for what a repository-posture tool *should* check and discovered it was
measuring the easy half. The throwaway repository I used for the API tests, with
the exact before/after states, is
[here](https://github.com/baseliner-sandbox/test-ruleset-precedence).

If you know that any of this is wrong — particularly the claim that no single
endpoint gives effective branch protection — I would genuinely like to be
corrected. The easiest way is an issue on the
[evidence repository](https://github.com/baseliner-sandbox/test-ruleset-precedence/issues).

## Corrections

**2026-10-07.** The visibility-toggle finding is free-plan only. On a GitHub
Team organisation classic protection survived the round trip, 2 out of 2. The
section now says so, and the closing request for a paid-plan data point is
gone, since this answers it.

**2026-10-06.** A source check before sharing this more widely turned up errors,
now fixed in place:

- I wrote that Renovate reads only classic protection and silently misses
  ruleset-only repositories. That is wrong: it has checked rulesets first since
  September 2025 ([renovatebot/renovate#38072](https://github.com/renovatebot/renovate/pull/38072)).
- Rulesets are unavailable on private repositories under GitHub Free, not
  "below GitHub Team"; GitHub Pro has them.
- Several supporting details were stated more strongly than their sources
  allow: the GitLab deprecation-file fields and release-post timing, GitHub's
  non-retroactive wording, the Azure DevOps sprint-note count, the
  tag-protection timeline, the docs word counts, Rule Insights being free, and
  the `GITHUB_TOKEN` default for older organisations. Each now says what the
  source says.
- Added GitHub's August 2026 "Convert to ruleset" option, which the original
  did not mention.

None of these changes the main claim.
