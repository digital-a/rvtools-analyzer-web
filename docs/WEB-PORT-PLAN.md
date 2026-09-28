# Porting RVTools Analyzer to the web (Cloudflare)

This is an analysis of what it would take to turn the native macOS app into a browser app hosted on Cloudflare. It's based on commit `a7c7078`. No code was changed for it.

- [Summary](#summary)
- [1. What's in the repo](#1-whats-in-the-repo)
- [2. Apple-specific code that has to be replaced](#2-apple-specific-code-that-has-to-be-replaced)
- [3. Target architecture](#3-target-architecture)
- [4. Cloudflare service mapping](#4-cloudflare-service-mapping)
- [5. Porting plan](#5-porting-plan)
- [6. Risks and decisions](#6-risks-and-decisions)
- [7. Effort, assuming Claude Code writes the code](#7-effort-assuming-claude-code-writes-the-code)

## Summary

The port is feasible, but it means rewriting the app, not converting it. None of the code can be carried over mechanically:

- **UI (~7.5k lines of SwiftUI, 20 files):** has to be rewritten, since SwiftUI doesn't run in a browser.
- **Core (~12.2k lines, `Sources/RVToolsCore`):** has no UI code, but it relies on Apple-only libraries in 4 places (listed in Section 2).

The recommendation:

- Rewrite the core in TypeScript and run it in the browser.
- Build the front end with React + Vite.
- Use Cloudflare only for hosting, price-list caching and optional project storage.

Customer RVTools data should stay in the browser by default. The README promises "your RVTools data never leaves the Mac"; if the port uploads exports to a server by default, that becomes a data-handling change you'd have to explain to customers.

## 1. What's in the repo

| Part | Location | Size | Portability |
|---|---|---|---|
| Parsing: ZIP, XLSX, CSV, Dataset | `Zip.swift`, `XLSX.swift`, `CSV.swift`, `Dataset.swift` | ~800 lines | Straight port; swap 2 Apple libraries |
| Correlation engine and model | `Correlate.swift` (1076 lines), `Models.swift`, `Relationships.swift`, `StoragePaths.swift`, `Subnets.swift` | ~2.9k | Plain logic; straight port |
| Rules, lifecycle, analysis, data quality | `Rules.swift` (70+ checks, `Thresholds`), `Lifecycle.swift`, `Analysis.swift`, `DataQuality.swift` | ~1.2k | Straight port |
| Trend mode | `Trend.swift` | 728 | Straight port; one concurrency call to replace |
| Built-in solutions | `Solutions/*` (VCF9Sizing 856, VCF9Readiness 522, CloudMigration 350…) | ~3.1k | Plain logic, except price downloading |
| Custom solutions runtime | `Custom/*` | ~1.9k | Scripts are already JavaScript; the sandbox around them must be replaced |
| PowerPoint export | `PPTX*.swift`, `ZipWriter.swift` | ~1.1k | Straight port; swap the compression library |
| Projects | `Project.swift` | 151 | The format needs to change (see Risk 4) |
| SwiftUI app | `Sources/RVToolsAnalyzer` | 7.5k | Full rewrite |
| CLI | `Sources/rvtools-cli/main.swift` | 470 | Keep as the reference during the port, then replace with a Node CLI on the TS core |

The custom solution API (`run(vms, inventory, params, context)`, [SOLUTIONS.md](SOLUTIONS.md)) is already JavaScript. The helper library (`ScriptPrelude.swift`, 319 lines) is JS source embedded in a Swift string. That part ports almost verbatim, and existing `.rvasolution` packs can keep working.

## 2. Apple-specific code that has to be replaced

| Current (file:line) | What it does | Web replacement |
|---|---|---|
| `import Compression`: `Zip.swift:142,153`, `ZipWriter.swift:76` | Unzipping and zipping for `.xlsx` and `.pptx` | `fflate` (small, runs in a Worker, supports streaming) |
| `XMLParser` delegates: `XLSX.swift:92–304` (5 parsers) | Reading workbooks, shared strings, styles and sheets | `saxes`, or a hand-written tokenizer. Keep the SAX-style streaming structure; don't load a full DOM for large sheets. |
| `DispatchQueue.concurrentPerform`: `XLSX.swift:52`, `Trend.swift:80` | Parsing sheets and snapshots in parallel | A pool of Web Workers, or a single Worker (it's fast enough for typical exports) |
| `JavaScriptCore` + the `dlsym` watchdog: `ScriptedSolution.swift:128,403–414` | Runs custom scripts sandboxed, with a time limit | QuickJS compiled to WebAssembly (`quickjs-emscripten`) inside a Worker. It has no network or DOM access, and its interrupt handler enforces `timeoutSeconds`. |
| `URLSession`: `CloudPricing.swift:212,268` | Downloads Azure/AWS prices | A Cloudflare Worker proxy (Section 4). CORS is probably a problem, but it hasn't been tested (see Risk 2). |
| `FileManager`: Caches, Application Support, `trashItem` (`CloudPricing.swift:91`, `SolutionPack.swift:145,281`, `PriceLists.swift:199,263`, `Project.swift`, `Dataset.swift:124–149`) | Price cache, installed packs and price lists, projects, CSV folders | IndexedDB or OPFS in the browser; R2 for anything synced |
| `UserDefaults` / `@AppStorage` (18 uses in `AppModel.swift`, 15 in `ConfigurationView.swift`…) | Settings, saved assumptions, recent projects | IndexedDB locally, with optional sync to D1 |
| Folder watching (automatic reload of solution packs) | Picks up pack edits as you save | No browser equivalent. Use an in-app editor (Monaco) or re-import the pack. |
| Separate `openWindow` windows (relationship maps) | Maps open in their own window | A route or modal |
| `ImageRenderer` / `DebugSnapshot.swift` | PNG export of maps and doc screenshots | Export the map's SVG to PNG in the browser; take doc screenshots with Playwright |
| SF Symbols (44 in the UI, plus the `"symbol"` field in pack manifests, e.g. `"internaldrive.fill"`) | Icons | A lookup table from SF Symbol names to Lucide icons, so existing manifests still load |
| Swift Charts (5 files, light use) | Charts | Recharts or ECharts |

## 3. Target architecture

```
Browser (the SPA)
├── Main thread (UI): React + Vite + TS, TanStack Router/Table/Virtual, Recharts, SVG relationship maps
├── Core Worker: the TS port of RVToolsCore
│     parse (fflate + SAX) → correlate → rules/analysis → solutions → PPTX/CSV export
├── Script Worker: QuickJS-wasm sandbox for custom solutions (ScriptPrelude.js + the rva.* bridge)
└── Local store: IndexedDB/OPFS (recent projects, autosave, settings, packs, price lists, price cache)

Cloudflare
├── Worker + Static Assets: serves the SPA, plus /api/*
├── Cron Trigger → Workflow: weekly price refresh, one step per provider/region
├── R2: normalized price snapshots; optional project bundles, packs, PPTX templates
├── D1: optional users, project index, synced settings/acknowledgements, pack and price-list registry
├── KV: optional index of the latest price snapshot per region (fast reads at the edge)
└── Access (Zero Trust): login in front of everything, no auth code to write
```

Why the core should run in the browser rather than in Workers:

1. **Keeps customer data local.** Parsing in the browser means customer data never has to reach Cloudflare.
2. **Workers limits don't fit.** Workers have a 128 MB memory limit and CPU-time caps. A large RVTools export (thousands of VMs, 25 tabs) plus the correlated model would get close to that. A browser tab won't.
3. **Everything is instant.** Every setting, scope change or assumption change recalculates instantly with no round trip, which is how the Mac app behaves now.

## 4. Cloudflare service mapping

| Need | Service | Notes |
|---|---|---|
| Hosting the SPA | **Workers with Static Assets** | Cloudflare is steering new projects to Workers rather than Pages, and one Worker serves both the SPA and `/api`. |
| Price downloads (`AzurePrices.fetch`, `AWSPrices.fetch`) | **Worker** at `/api/prices/:provider/:region` | Reads from R2. On a miss, fetches upstream, converts to `RegionPrices` JSON and stores it. This fixes CORS and turns ~80 paginated Azure calls per region into one GET for the browser. |
| The 7-day refresh (`PriceStore.maxAge`) | **Cron Trigger** starting a **Workflow** | Pagination is capped at 80 pages (`CloudPricing.swift:132`) and runs twice per region (VMs, then disks), so up to ~160 upstream calls per region. That can exceed per-invocation subrequest limits. Workflows give each region its own retried step. Check the current plan limits. |
| Price snapshot storage | **R2** at `prices/{provider}/{region}/{yyyy-mm-dd}.json` | Dated keys keep old snapshots, so projects can still reproduce their estimates, which is what `Project.swift` does today with `prices/`. |
| Latest-snapshot index | **KV** at `latest:{provider}:{region}` → R2 key | Optional; D1 can do this too. |
| Authentication | **Cloudflare Access** | Free tier covers small teams. The Worker reads the `Cf-Access-Jwt-Assertion` header for the user's identity. No login code to write. |
| Synced settings, acknowledgements, saved assumptions, project list | **D1** | Only if you want work to follow users across devices. |
| Project bundles, shared packs, PPTX templates | **R2** | Encrypt customer exports in the browser with WebCrypto AES-GCM before uploading, so R2 only holds ciphertext. |
| Large uploads | R2 presigned URLs, or multipart upload through the Worker | Keeps big exports out of Worker memory. |
| Team-wide pack and price-list library | R2 files + a D1 index | Replaces the "symlink Application Support to OneDrive" workaround in the README. |

A starting D1 schema, if you add sync:

```sql
CREATE TABLE users (id TEXT PRIMARY KEY, email TEXT UNIQUE NOT NULL, created_at TEXT NOT NULL);
CREATE TABLE projects (
  id TEXT PRIMARY KEY, owner_id TEXT NOT NULL REFERENCES users(id),
  name TEXT NOT NULL, mode TEXT, format_version INTEGER NOT NULL,
  r2_key TEXT NOT NULL, encrypted INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL, modified_at TEXT NOT NULL
);
CREATE TABLE user_settings (user_id TEXT PRIMARY KEY REFERENCES users(id), thresholds_json TEXT, units_json TEXT, solution_params_json TEXT);
CREATE TABLE packs (id TEXT PRIMARY KEY, kind TEXT CHECK (kind IN ('solution','pricelist','template')), title TEXT, version TEXT, r2_key TEXT NOT NULL, owner_id TEXT, shared INTEGER DEFAULT 0, updated_at TEXT);
CREATE TABLE price_snapshots (provider TEXT, region TEXT, fetched_at TEXT, r2_key TEXT, PRIMARY KEY (provider, region, fetched_at));
```

## 5. Porting plan

**Phase 0: Reference outputs from the Mac app (do this first).**

- On a Mac, run the current `rvtools-cli` on `samples/*.xlsx`, the CSV folder and `samples/series` with:
  - `--export`
  - each `--solution`, with fixed `--set` values
  - `--trend`
  - `--map`
- Commit those outputs as test fixtures. The TS core passes when it produces the same outputs.
- This is the only practical way to prove 70+ rules and six sizing engines ported correctly.
- The Swift package doesn't build on Linux (`Compression` and `JavaScriptCore` are Apple-only), so generating the fixtures needs a Mac.

**Phase 1: Core in TypeScript.** Port in dependency order, checking against the fixtures at each step:

1. `Zip`, `XLSX`, `CSV`, `Dataset`
2. `Models`, `Correlate`
3. `Rules`, `Lifecycle`, `Analysis`, `DataQuality`
4. `Relationships`, `StoragePaths`, `Subnets`
5. `Trend`
6. `SolutionKit`, then each solution
7. `Export`
8. `PPTX*` with `ZipWriter`

Package it as `packages/core`. It should be pure TS with no DOM access, so the same code runs in a Worker and in Node for the new CLI.

**Phase 2: Custom solutions runtime.**

- Load `ScriptPrelude.js` into QuickJS.
- Reimplement the bridge functions set up in `ScriptedSolution.swift:313` (`install`).
- Enforce the time limit with the interrupt handler.
- Pass data in and out as JSON, which is how `JSONValue.swift` already works.
- Run `examples/solutions/*` as test fixtures (`scripts/check-solutions.sh` already does the equivalent on the Mac).

**Phase 3: A static web app with no backend.**

- React UI covering Overview, Issues, Compute, VMs, Storage, Network, Configuration, Lifecycle, Correlations, Raw Tabs, Trends and Solutions.
- File input: drag-and-drop plus `<input type=file multiple>`. For CSV folders, use a directory picker (`webkitdirectory`) or read dropped folders.
- Projects as a single zip file (Risk 4).
- Deploy as a Worker with Static Assets behind Access. At this point it's a working product, with customer data still staying on the user's machine.

**Phase 4: Price service.** Worker `/api/prices` + R2 + Cron/Workflow. The "Download Prices" button now calls your API.

**Phase 5 (optional): Sync.** D1 + R2 with client-side encryption. Adds a team pack library and cross-device projects.

**Phase 6: Retire the Swift code, or keep both.** Decide whether the Mac app stays alive. Keeping both means every rule change is made twice, and they will drift apart. Pick one.

## 6. Risks and decisions

1. **Swift→WebAssembly instead of a rewrite: not recommended.**
   - The same four places block it: `Compression`, `JavaScriptCore`, `XMLParser` (FoundationXML depends on libxml2), and threads.
   - The Foundation runtime would add several MB to the download (an estimate, not measured).
   - The UI still has to be rewritten in either case.
   - It would save part of Phase 1, but you'd be debugging Swift Wasm toolchain problems instead. Nobody has tried a build of this repo with the Swift Wasm SDK; verify that before relying on it.
2. **CORS and the size of the AWS price files: unknown.**
   - Whether Azure and AWS allow direct browser requests hasn't been tested; the analysis environment's network proxy blocked both. Either way, the Worker proxy removes the question.
   - The size of the AWS `meteredUnitMaps` files is also unknown. If one region's file is tens of MB, parsing it in a Worker could hit the 128 MB memory limit. In that case, stream-parse it or run that step in a container.
3. **Browser memory on huge exports.** A single-page app holding the full correlated model is probably fine up to about 10k VMs. Test with a real large export early; the synthetic samples are ~600 KB.
4. **The project format has to change.**
   - `.rvaproj` is a folder that Finder shows as one file. Browsers can't reliably read or save folders like that.
   - Move to a single zip file with the same layout: `project.json`, `sources/`, `prices/`.
   - Add an importer for existing folder projects, using the directory picker, so current users can bring their projects over.
   - Bump `ProjectFile.currentFormat` to 2 (`Project.swift:15`).
5. **Features that work differently on the web:**
   - Saving back to the same file ("save in place") only works in Chromium browsers; Safari and Firefox fall back to downloading a copy.
   - Double-click to open from Finder and "Open With" go away.
   - Automatic reload when a pack changes on disk goes away.
6. **Security of the sandbox.** A Web Worker on its own can still make network requests, so running shared packs in a plain Worker would break the "a shared solution can't send customer data anywhere" guarantee. QuickJS-wasm keeps that guarantee.
7. **Wording in the solutions.** The Azure/AWS notes at `CloudMigration.swift:339–340` and the README's privacy claims need rewording to cover prices fetched through your server and optional cloud sync.

## 7. Effort, assuming Claude Code writes the code

Most of the typing effort goes away. What's left is checking the port is correct and making decisions, and those need the person steering. The estimates below are guesses: there's no measured data on how fast agents port this particular codebase. They assume one engineer steering, reviewing, and running Claude Code sessions in parallel where the work can be split.

| Phase | Calendar time | Engineer time | What sets the pace |
|---|---|---|---|
| 0: Reference outputs | ½ day | ½ day, on a Mac | Only a Mac can generate the fixtures. Claude Code can write the script, but a person runs it. |
| 1: Core in TypeScript | 1.5–2.5 weeks | ~30% | Each module is one agent task: port it, then loop until its output matches the fixtures. Parsing, rules and trend can run in parallel sessions. Correlate and VCF9Sizing need the most review. |
| 2: Custom solutions runtime | 3–5 days | ~40% | Small amount of code, but the sandbox needs a real security review by a person. |
| 3: UI | 2.5–4 weeks | ~50% | Taste and layout decisions. An agent can match the screenshots in `docs/images/` using Playwright, but a person will iterate on how it looks and feels. |
| 4: Price service | 2–4 days | ~30% | Cloudflare account setup (a `wrangler` token, R2 bucket, Access policy), plus testing the Workflow against the real Azure and AWS endpoints. |
| 5: Sync + encryption | 1–1.5 weeks | ~40% | Review of the encryption and key handling. Don't hand that to an agent without checking it yourself. |

That's roughly 6–9 weeks end to end, or about 4–6 weeks to a local-only product (phases 0–4), against about 4–5 months done by hand.

**What doesn't get faster with agents:**

- **Correctness where the fixtures run out.** The sample data exercises the rules it was built to show. Edge cases in real exports (RVTools 3.x headers, VMC, multiple vCenters, blank columns) only surface with real data, and customer exports shouldn't go to a cloud agent session. Run those comparisons locally.
- **Fixture coverage decides quality.** An agent loops until the tests pass. Whatever the fixtures don't cover, it will get plausibly wrong. Spend Phase 0 time on breadth:
  - every `--solution` with non-default `--set` values
  - trend mode
  - every `--map` type
  - both unit systems (`--units binary|decimal`, `--rate bits|bytes`)
- **Review volume.** About 20k lines of new TypeScript will arrive faster than anyone can read it. Review by behaviour (fixture diffs, screenshot diffs) rather than line by line. Read closely only in Correlate, the sizing engines, the sandbox bridge and the crypto.

**Setup that makes the agent sessions productive:**

- A `CLAUDE.md` in the repo holding the architecture from Section 3, the rule that the core has no DOM access, and the test command for each package.
- A single command per module (`pnpm test:core -- correlate`) that diffs output against the fixtures. That gives each session a clear finish line.
- One module per session, with the Swift file named as the source of truth. Don't ask for "port the core".
- Playwright screenshot tests against `docs/images/*.png` as the target for the UI phase. Expect to relax them to layout checks, since the web version won't be pixel-identical.
