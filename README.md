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
| TT Progress | `progress.html` | The commit's workflow runs, their jobs, and the steps in progress; then its published results at a glance (`summary.json`): why a job failed, tests, precheck, layout metrics, a stale `.sv`. Opens TT Results and TT Datasheet, and TT Results by itself on a failure. |
| TT Results | `results.html` | Every file the commit's runs published, each shown according to its type (logs with their errors marked, reports, JSON, CSV, Markdown, test results, images; a waveform via Surfer). Opens on a failed job's log. Also works in a browser tab: `results.html?repo=<owner>/<name>`. |
| TT Datasheet | (the project's Pages) | The project's `datasheet.pdf`, from TT's `docs` run, in Makerchip's PDF view. |
| TT Layout | `layout.html` | The layout from the project's Pages: the 2D render, TT's 3D viewer, Tiny Tapeout Explorer. |
| TT VGA | `vga.html` | Each Makerchip compile's Verilog running live in [VGA Playground](https://github.com/stevehoover/vga-playground) (a fork, for its embedding API): the Tiny VGA PMOD display, inputs and audio. |

Each sets its tab status: working while its results are pending, then success, warning (e.g. a stale `.sv`) or fail.

## Contract

- **Channel:** every view needs `rpc: true` (for `getContext` and `setStatus`; without it the IDE refuses every
  call). Their links open through the `openUrl` RPC, since a webview may block a view's own (e.g. VS Code's).
- **`params`:** `{repo: "<owner>/<name>", sha?}`. Without `sha`, TT Progress follows the latest commit with runs.
- **`tt/run-start` `{repo, sha}`** (subscribed by both): a commit was pushed; follow it.
- **`tt/run-status` `{repo, sha, done, runs: [{name, status, conclusion, url, jobs: [{name, status,
  conclusion}]}]}`** (produced by TT Progress, subscribed by TT Layout): emitted on each poll. TT Progress is the
  only poller of the GitHub API, whose anonymous limit is 60 requests/hour; it polls each minute until the runs
  complete.
- **RPC `getRunStatus()`** on TT Progress (with `rpc: true`) returns the last `tt/run-status` payload.
- **`companions`** of TT Progress (see the Third-Party View API): `"TT Results"` (`results.html`, with `params`
  `{repo}`, subscribing `tt/run-start`, `tt/run-status` and the IDE's `theme`, and itself with the companion
  `"TT Datasheet"`) and `"TT Datasheet"` (a `pdf` view of `<pages>/runs/docs/PDF/datasheet.pdf`). Without them
  (e.g. in a browser), the views open these pages in a new tab.
- **TT VGA** subscribes to the IDE's `compile-result` and `theme` (`sources: ["ide"]`), and needs no `repo`. It
  loads the generated Verilog and the files under `sv_url_inc/` (as listed in `artifacts`) from each successful
  compile's `resultsUrl`. The page's `?playground=<url>` overrides the VGA Playground URL (e.g. a dev server's
  `http://localhost:5173/?embed`).

## Designs for TT VGA

[tt-makerchip-template](https://github.com/stevehoover/tt-makerchip-template)'s `vga` setting meets all of this,
using `tt_vga()` in Virtual FPGA Lab's `tiny_tapeout_lib.tlv`.

**The marker.** The comment `// makerchip-tt: vga` in the Verilog marks a VGA design; `tt_vga()` emits it. For a
marked design in a TT project, the Makerchip extension opens TT VGA when it compiles, and its "Run TT Flow" button
checks `info.yaml` for the VGA clock and pins; TT VGA notes a compile that isn't marked (it still runs it).

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
| GitHub Pages | for a project made from tt-makerchip-template, everything below (see "GitHub Pages"); otherwise only what TT's `viewer` job publishes: the redirect to TT's 3D viewer (with the PDK), `tinytapeout.{gds,oas}`, `gds_render.png` | yes, with CORS; `max-age=600` |
| Run summary page | lint, Yosys warnings, routing stats, cell categories, precheck `results.md`, test summaries | HTML page only |
| Artifact zips | `tt_submission`, `GDS_logs`, `precheck_reports`, `gatelevel_test_results`, `test-results`, `PDF`, `fpga_bitstream` | no |
| Job logs | full output, including TT's check messages | no (repo admin) |
| Actions REST API | run, job and step status | yes, for public repos (60 requests/hour) |

### GitHub Pages

tt-makerchip-template's `makerchip.yaml` workflow runs as each of `gds`, `test` and `docs` completes. Once `gds`
has completed (its `viewer` job also deploys Pages), it republishes Pages for the branch's latest commit:

- TT's viewer files, as its `viewer` job lays them out.
- `runs/<workflow>/<artifact>/...`: every artifact of the commit's latest `gds`, `test` and `docs` runs, unzipped,
  except the tools' binary databases (`.odb`, `.gds`, `.oas`, `.mag`, `.def`, `.sdf`, `.spef`, `.lib`), which only
  EDA tools read; the final layout is TT's `tinytapeout.{gds,oas}`.
- `runs/<workflow>/logs/...`: each completed run's job logs (as GitHub's log archive lays them out).
- `manifest.json`: `{sha, published, pdk, runs, sv, omitted, files}`, where `pdk` is `tt_submission/pdk.json`
  (PDK and tool versions), `runs.<workflow>` is `{id, url, status, conclusion, jobs: [{name, status, conclusion,
  url, failed_steps}], artifacts, logs}`, `sv` lists each `src/*.sv` as `{sv, tlv, fresh, vga}`, `omitted` counts
  the files left out, by extension (`{files, bytes}`), and `files` lists every file.
- `summary.json`: the results at a glance, `{sha, pdk, tiles, runs: {<workflow>: {conclusion, url}}, metrics,
  tests: {rtl, gl}, precheck, failures, sv, datasheet}`. `metrics` are key LibreLane metrics of the final layout,
  by their LibreLane names. `tests.rtl`, `tests.gl` and `precheck` are JUnit results as `{tests, failed, skipped,
  cases: [{name, status, message}]}`, or `null`. Each of `failures` is a failed job, `{run, job, steps, url, log,
  errors, context}`: its log's path, its error lines, and the lines leading to its first error. `datasheet` is the
  PDF's path, or `null`.

A run that completes before `gds` publishes nothing; `gds`'s own completion then does. Poll `manifest.json` or
`summary.json` (with a cache-busting query) until its `sha` is the commit you want.

**`fresh`** says whether a generated `.sv` was generated from the committed `.tlv`: "Run TT Flow" stamps the
`.sv` with `// makerchip-tt: tlv-sha256 <hash>`, the SHA-256 of the `.tlv` with line endings normalized to LF.
`null` if there is no stamp (e.g. a `.sv` generated by hand). It doesn't cover included libraries. **`vga`** says
whether the `.sv` has the VGA marker (see "Designs for TT VGA").

## Development

Views loaded from local files (e.g. by the extension's `makerchip_open_third_party_view` `filePath`) become
`data:` URIs, which inherit the IDE's CSP, so their `<script src>` from jsDelivr is blocked. To test locally,
inline the view client in a scratch copy.
