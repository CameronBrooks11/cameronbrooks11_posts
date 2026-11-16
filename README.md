# cameronbrooks11_posts

The posts for my blog, published at
[cameronbrooks11.github.io/writings](https://cameronbrooks11.github.io/writings/). The site
([cameronbrooks11.github.io](https://github.com/CameronBrooks11/cameronbrooks11.github.io)) builds
them with Jekyll, with this repository as its `_posts` submodule.

## Writing a post

- Name the file `YYYY-MM-DD-slug.md`; Jekyll publishes only files named that way.
- Start it with front matter:

  ```yaml
  ---
  layout: post
  title: "The title"
  date: 2026-09-12 01:58:00 -0400
  categories: DevOps
  ---
  ```

- Add `published: false` to keep a dated draft off the site. A file without a date in its name, such
  as `dump.md` (notes for the rocketry post), is never published.
- A new post goes live when the site's `_posts` submodule is moved to the new commit.

## Licence

The posts are licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); see
[LICENSE](LICENSE). Reuse them with credit and a link.
