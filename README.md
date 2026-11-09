# Makerchip Tiny Tapeout views

[Makerchip](https://makerchip.com) third-party views that show a [Tiny Tapeout](https://tinytapeout.com) (TT)
project's GitHub Actions results inside the Makerchip IDE. They are served from this repo's GitHub Pages
(`https://stevehoover.github.io/makerchip-tt-views/<view>.html`) and built on the
[Makerchip view client](https://www.npmjs.com/package/@rweda/makerchip-view-client).

The Makerchip VS Code extension opens them for a project made from
[ttsky-makerchip-template](https://github.com/stevehoover/ttsky-makerchip-template) when its "Run TT Flow"
button pushes a commit. They need no sign-in: the project is public, its run status comes from the GitHub API
anonymously, and its results from its GitHub Pages.

## Views

| View | Page | Shows |
|---|---|---|
| TT Progress | `progress.html` | The commit's workflow runs, their jobs, and the steps in progress. |
| TT Layout | `layout.html` | The layout from the project's Pages: the 2D render, TT's 3D viewer, Tiny Tapeout Explorer. |

Each sets its tab status: working while its results are pending, then success or fail.

## Contract

- **Channel:** both views need `rpc: true` (for `getContext` and `setStatus`; without it the IDE refuses every
  call).
- **`params`:** `{repo: "<owner>/<name>", sha?}`. Without `sha`, TT Progress follows the latest commit with runs.
- **`tt/run-start` `{repo, sha}`** (subscribed by both): a commit was pushed; follow it.
- **`tt/run-status` `{repo, sha, done, runs: [{name, status, conclusion, url, jobs: [{name, status,
  conclusion}]}]}`** (produced by TT Progress, subscribed by TT Layout): emitted on each poll. TT Progress is the
  only poller of the GitHub API, whose anonymous limit is 60 requests/hour; it polls each minute until the runs
  complete.
- **RPC `getRunStatus()`** on TT Progress (with `rpc: true`) returns the last `tt/run-status` payload.

## Development

Views loaded from local files (e.g. by the extension's `makerchip_open_third_party_view` `filePath`) become
`data:` URIs, which inherit the IDE's CSP, so their `<script src>` from jsDelivr is blocked. To test locally,
inline the view client in a scratch copy.
