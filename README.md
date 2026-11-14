# Makerchip Tiny Tapeout views

[Makerchip](https://makerchip.com) third-party views that show a [Tiny Tapeout](https://tinytapeout.com) (TT)
project's GitHub Actions results inside the Makerchip IDE. They are served from this repo's GitHub Pages
(`https://stevehoover.github.io/makerchip-tt-views/<view>.html`) and built on the
[Makerchip view client](https://www.npmjs.com/package/@rweda/makerchip-view-client).

The Makerchip VS Code extension opens them for a project made from
[tt-makerchip-template](https://github.com/stevehoover/tt-makerchip-template) when its "Run TT Flow"
button pushes a commit. They need no sign-in: the project is public, its run status comes from the GitHub API
anonymously, and its results from its GitHub Pages.

## Views

| View | Page | Shows |
|---|---|---|
| TT Progress | `progress.html` | The commit's workflow runs, their jobs, and the steps in progress. |
| TT Layout | `layout.html` | The layout from the project's Pages: the 2D render, TT's 3D viewer, Tiny Tapeout Explorer. |
| TT VGA | `vga.html` | Each Makerchip compile's Verilog running live in [VGA Playground](https://github.com/stevehoover/vga-playground) (a fork, for its embedding API): the Tiny VGA PMOD display, inputs and audio. |

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
- **TT VGA** subscribes to the IDE's `compile-result` and `theme` (`sources: ["ide"]`), and needs no `repo`. It
  loads `top.sv` and the files under `sv_url_inc/` from each successful compile's `resultsUrl`. An optional
  `params.playground` overrides the VGA Playground URL (e.g. a dev server's `http://localhost:5173/?embed`).

## Designs for TT VGA

The view runs the compile's Verilog as is, so the design source must suit both Makerchip and VGA Playground:

- `--inlineGen` on the `\m5_TLV_version` line, so the Verilog is one file.
- Makerchip-only Verilog, such as the `top` test bench that instantiates the `tt_um_*` module, inside
  `` `ifdef MAKERCHIP`` (defined only by Makerchip's simulation).
- Other modules, e.g. `hvsync_generator`, fetched with `m4_sv_get_url(<url>, <module>.sv)`, not
  `m4_sv_include_url`: Makerchip then finds the module by name, and the view passes the file along.

## Where a TT run's results land

What a new view can read. TT's workflows (`gds`, `test`, `docs`) all run on push.

| Where | What | Readable without auth |
|---|---|---|
| GitHub Pages (`gds` workflow's `viewer` job) | redirect to TT's 3D viewer (with the PDK), `tinytapeout.{gds,oas}`, `gds_render.png` | yes, with CORS; `max-age=600` |
| Run summary page | lint, Yosys warnings, routing stats, cell categories, precheck `results.md`, test summaries | HTML page only |
| Artifact zips | `tt_submission`, `GDS_logs`, `precheck_reports`, `gatelevel_test_results`, `test-results`, `PDF`, `fpga_bitstream` | no |
| Job logs | full output, including TT's check messages | no (repo admin) |
| Actions REST API | run, job and step status | yes, for public repos (60 requests/hour) |

## Development

Views loaded from local files (e.g. by the extension's `makerchip_open_third_party_view` `filePath`) become
`data:` URIs, which inherit the IDE's CSP, so their `<script src>` from jsDelivr is blocked. To test locally,
inline the view client in a scratch copy.
