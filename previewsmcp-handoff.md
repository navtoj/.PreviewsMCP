# Handoff: PreviewsMCP on macOS 27 (continuation 2)

Written 2026-11-20. About a month will pass before work resumes, so check for newer Xcode, macOS and upstream releases before trusting anything below.

## Goal and rules
Build and run `//previewsmcp/cli:previewsmcp` from `navtoj/.PreviewsMCP` (clone: `/Users/navtoj/Developer/PreviewsMCP`) on Apple Silicon, macOS 27.2.1, Bazel 9.3.3, Xcode 27.2 (build 27B52). Only macOS matters.

- Never use `obj-p/tap` or anything under `github.com/obj-p`. Verify upstream sources against official repos.
- Give exact commands with filtered output. For builds, use:
  `bazel build --curses=no --color=no //previewsmcp/cli:previewsmcp 2>&1 | tee /tmp/bazel-build.log | grep --line-buffered -E "error:|ERROR:|FAILED|CMake Error|fatal error"`
- Fix one error at a time. Treat hypotheses as unverified until checked in source.
- zsh expands unquoted globs, so quote `--include='*.swift'`.
- After rebuilding, kill the old daemon or it keeps running old code:
  `pkill -f 'previewsmcp.*serve'; rm -f ~/.previewsmcp/serve.sock ~/.previewsmcp/serve.pid`
- The daemon log is `~/.previewsmcp/serve.log`. Look for `JIT session error:` lines first.
- User preferences: tabs (width 4), TypeScript over JavaScript, Bun/Deno over Node, Swift over Objective-C, SwiftUI hosted in AppKit via NSHostingView. Existing repo files such as `Compiler.swift` use 4 spaces, so match the file you are editing.

## Captured state
(Not captured yet. Run this once, then paste `/tmp/handoff-state.md` here.)

```bash
cd /Users/navtoj/Developer/PreviewsMCP
{
echo '## Captured state'; date
echo '### Versions'
sw_vers; xcodebuild -version; xcrun swiftc --version 2>&1 | head -2; bazel --version; cat .bazelversion
echo '### Git'
git remote -v; git branch -a -vv; git status --short
git rev-parse HEAD wip/macos27-live-preview
git diff --stat main wip/macos27-live-preview
echo '### LLVM archive'
ls -l ~/.local/llvm-src/; shasum -a 256 ~/.local/llvm-src/*.tar.gz
grep -n -B2 -A14 'llvm_repository' MODULE.bazel
echo '### Frame'
git -C ../Frame status --short
} > /tmp/handoff-state.md 2>&1
```

## Verified working
- The binary builds. `list` and `snapshot` work: `snapshot` of `Frame/App/Sources/ContentView.swift` produced a correct 400x600 RGBA PNG showing "Hello, world!".
- `previewsmcp run <file>` opens a live window. A text edit hot-reloads in 1-2 s, and a structural edit (new `@State` plus a `Button`) hot-reloads through the incremental-compile path.
- Clicking the button in the window updates the state.

## Root cause and fix for rendering
**Root cause (verified):** swiftc on this target emits `objc_msgSend$<selector>` stubs by default. ld-prime synthesizes them at static link time, and the preview JIT has no link step, so JIT link failed with unresolved symbols (`_objc_msgSend$mainBundle`, `$bundleForClass:`, `$initWithPath:`, `$init`, `$colorNamed:bundle:`). These likely came from Xcode's `GeneratedAssetSymbols.swift` (inferred from the selector names and the `rewroteResourceBundle` log line, not confirmed in an object file). Confirmed by compiling a trivial `Bundle.main` file with and without the flag: 1 stub vs 0.

**Fix:** `previewsmcp/PreviewsCore/Compiler.swift` gained:

```swift
/// ld-prime synthesizes `objc_msgSend$<selector>` stubs at static link time; the preview
/// JIT has no such step, so emit plain `objc_msgSend` calls instead.
private static let jitSafeFrontendFlags: [String] = [
    "-Xfrontend", "-disable-objc-msgsend-selector-stubs",
    "-Xfrontend", "-disable-objc-msgsend-class-selector-stubs",
]
```

It is appended after `extraFlags` in four argv builders: the one-shot compile, `emitStableModule`, `compileModuleIncremental`, and `captureOverlayFrontendJob` (the `-###` capture that feeds the cached frontend-bypass template). After this, no JIT errors appeared in any run.

**Not checked:** clang-compiled Objective-C/C targets would need `-fno-objc-msgsend-selector-stubs`. Nothing has hit that yet.

## Git state
- `wip/macos27-live-preview` is published to the remote. It is one WIP commit holding everything below. **Never rewrite or force-push it.** It is the backup, and it contains the machine-local `file://` LLVM URL.
- `macos27-support` was proposed as the clean branch off `main`, with one commit per group below. **It is unknown whether it was created.** Check `git branch -a`.
- After the 0.11.0 build experiment, `Package.swift` and `Package.resolved` should be back at 0.12.1 (`git checkout HEAD -- Package.swift Package.resolved`). Confirm with `git status --short`.
- Build clean commits by pulling files over from the WIP branch, for example `git checkout wip/macos27-live-preview -- previewsmcp/PreviewsCore/Compiler.swift`.

## Changes, grouped for separate commits
1. **Toolchain:** `.bazelrc` (`--xcode_version=27.2`), `MODULE.bazel` (llvm `sha256` set to `e6f6fd5d…`, `url` set to `file:///Users/navtoj/.local/llvm-src/llvm-src-swift-6.2.3-RELEASE.tar.gz`, and `//patches:rdf-register-ref-less.patch` added as the third entry in `llvm_repository.patches`), and `patches/rdf-register-ref-less.patch`. The patch replaces the `std::less<RegisterRef>` specialization with `llvm::rdf::RegisterRefLess` to fix a libc++ error in `RDFGraph.cpp`. It was verified to apply cleanly after the other two patches. Keep the `file://` URL and sha256 hunk out of any PR.
2. **Stub flags:** `Compiler.swift`, as above. Suggested message:
   ```
   Disable objc_msgSend selector stubs in preview compiles

   swiftc on recent toolchains emits objc_msgSend$<selector> stubs that
   ld-prime synthesizes at static link time. The preview JIT has no such
   step, so JIT link failed with unresolved symbols. Pass
   -disable-objc-msgsend-selector-stubs (and the class-selector variant)
   to every swiftc invocation in Compiler.swift.
   ```
3. **`Task.detached`:** `PreviewsMCPApp.swift:69`, `Task {` became `Task.detached {`, to fix `sending 'mutable' risks causing data races`. Check that the file has no other changes mixed in (`git diff main wip/macos27-live-preview -- <file>`).
4. **swift-sdk 0.12.1 migration:**
   - `Package.swift` and `Package.resolved` (0.11.0 to 0.12.1, which also drops the `swift-async-algorithms` pin).
   - `PreviewsCore.ProgressReporter` qualifications in 14 files (including `PreviewStartHandler.swift` and `MCPServerSupport.swift`).
   - `.text(t)` became `.text(t, _, _)` in two places in `MCPContentHelpers.swift`.
   - A fourth `_` in `.image(...)` patterns in `VariantsCommand.swift`, `SnapshotAllCommand.swift` (x2) and `SnapshotCommand.swift`.
   - `PreviewsMCPClient.swift:183`: `fail(id:…)` wrapped in `if let requestId = cancelled.params.requestId`.
   - `MODULE.bazel.lock` grew by about 500 lines. Check that this is expected.

   Suggested message (verify against the real diff, especially the `requestId` claim, which was inferred):
   ```
   Bump swift-sdk to 0.12.1 for Xcode 27

   0.11.0 fails to compile under Xcode 27.2 (NetworkTransport.swift:
   "sending ... risks causing data races"). Move to 0.12.1 and adapt call
   sites: Content enum cases gained associated values, and
   cancelled.params.requestId is now optional.
   ```

## Decisions and rationale
- **Branches:** the WIP commit is the published backup because it contains the machine-local LLVM URL and was meant to be split. Clean history goes on `macos27-support`.
- **Keep the 0.12.1 bump (verified):** with 0.11.0 restored, the Bazel build fails inside the SDK (`NetworkTransport.swift:532` and `:763`, `sending '...ContinuationResumed' risks causing data races`). Commit `c8f0261` pinned 0.11.0 deliberately, for the `SerializedStdioTransport` workaround (#320). That reason is superseded: `FramedTransport.swift` and `PreviewsMCPClient` (#334) now replace the SDK's transports and client. The SDK still supplies protocol types and the server side, so test `serve` before committing (see below).
- **Fix at the compiler flag, not the JIT:** one argument per swiftc call. Defining `objc_msgSend$...` symbols on the JIT side would be a far bigger change and was never needed.
- **Race left unpatched:** the JIT teardown race never reproduced after the stub fix, so no LLVM patch was written.

## Open: test suite does not build
`bazel test //previewsmcp/Tests/...` fails because the test sources still use the 0.11.0 shapes (`.text` is now a 3-tuple, `.image` needs a fourth pattern). The other 12 targets showed `NO STATUS` only because Bazel stopped at the first build failure. A `--keep_going` run was started but never finished, so **there are no real pass/fail results yet.**

Known compile errors:
- `previewsmcp/Tests/MCPIntegrationTests/MCPTestServer.swift`: 342, 359, 421
- `previewsmcp/Tests/PreviewsCLITests/MCPHandlerContractTests.swift`: 126, 630, 637
- `previewsmcp/Tests/PreviewsCLITests/MCPWireTestSupport.swift`: 27
- `previewsmcp/Tests/PreviewsCLITests/PhaseFailureCarrierTests.swift`: 25-27, 80, 85

Likely fixes are `case .text(let t, _, _)` and the extra `_`, but the `==` comparisons and `.hasPrefix`/`.contains` calls on the tuple may need different treatment. Read the lines before editing. Then:

```bash
bazel test --keep_going --curses=no --color=no //previewsmcp/Tests/... 2>&1 | tee /tmp/bazel-test.log | grep -E "error:|PASSED|FAILED|Executed"
```

Simulator-dependent suites (`IOSPreviewE2ETests`, `PreviewsIOSTests`) may fail for environmental reasons. Separate those from real regressions.

## Not verified
- `serve` over MCP stdio with a real MCP client. Do this before committing the SDK bump, since `snapshot` and `run` don't exercise that path.
- A view that loads an asset-catalog color or image. The original stub errors likely came from `GeneratedAssetSymbols.swift`, so this is the best test of the rewritten resource bundle.
- The original segfault: `EXC_BAD_ACCESS` at address 0 in a JIT worker (`JITDylib::getDefaultResourceTracker()` called from `EPCDynamicLibrarySearchGenerator::tryToGenerate`) while thread 1 ran `~LLJIT` → `endSession` → `SimpleRemoteEPC::disconnect`. Verified in LLVM source: `endSession` removes JITDylibs (`Core.cpp:1631`) before `EPC->disconnect()` (`:1633`), and the generator lambda captures `&JD` raw (`EPCDynamicLibrarySearchGenerator.cpp:53`). That suggests a use-after-free, but it is a hypothesis. It is probably latent: the segfault was most likely the unresolved-symbol error going through the teardown path. Before writing any LLVM patch, check `swiftlang/llvm-project` and upstream `llvm/llvm-project` for an existing fix.
- Whether `snapshot` ever worked on Xcode 26.2 with the original LLVM archive. If it did, this is a regression worth reporting.
- A text edit logged `Refreshed (native rebuild + reload)` before `Structural change, recompiling...`, while the structural edit logged only the latter. The watcher code hasn't been read. It works either way.
- Whether Bazel uses `Package.resolved` or its own SDK pins (`rules_swift_package_manager`). The 0.11.0 experiment did change the build, so it appears to read it, but `MODULE.bazel` was not checked.

## Reference: healthy vs broken log
Healthy (snapshot, Xcode 27.2, after the stub fix):
```
jit_latency: stable-module non-leaf 221ms
jit_latency: incremental-compile 1010ms
jit_latency: agent-session force-fresh=true 24ms
jit_latency: add-deps dylibs=0 archives=1 0ms
jit_latency: add-objects 5 1ms
jit_latency: render-entry 88ms
snap: captured 8102 bytes
```
Warm hot-reload: incremental-compile about 350-650 ms, render-entry about 105-115 ms.

Broken (before the fix): `JIT session error: Symbols not found: [ _objc_msgSend$mainBundle ... ]` right after `add-objects 5`, then `Error: Internal error: Server disconnected` from a daemon segfault. Crash reports go to `~/Library/Logs/DiagnosticReports/previewsmcp-*.ips`.

If `serve.log` is missing: `DaemonClient.spawnDaemon` (lines 206-245) falls back to `/dev/null` when it can't open the log, probably because `~/.previewsmcp` didn't exist at first spawn. The directory exists now, and the log has been captured since.

## Test state of the Frame project
`/Users/navtoj/Developer/Frame/App/Sources/ContentView.swift` was overwritten with a counter view during testing. The original was:

```swift
import SwiftUI

// MARK: - ContentView

public struct ContentView: View {
	// MARK: Lifecycle

	public init() {}

	// MARK: Content Properties

	public var body: some View {
		Text("Hello, world!")
			.padding()
	}
}

// MARK: - ContentView_Previews

struct ContentView_Previews: PreviewProvider {
	static var previews: some View {
		ContentView()
	}
}
```

Restore it with `git -C /Users/navtoj/Developer/Frame checkout -- App/Sources/ContentView.swift`, assuming the file is tracked there. If not, paste the above.

## Idea parked: live preview in your own app's window
Today the JIT code runs in a separate `PreviewAgent` process (`previewsmcp/PreviewAgent/main.cpp`, which calls `sharedApplication`/`run` via `objc_msgSend`), and that process owns the window and the `NSHostingView`. Every edit starts a fresh agent session (`force-fresh=true`), which is why `@State` resets. The window-hosting code has not been read. Options, from least to most work:
1. Poll snapshots into your own window. Simple, not interactive.
2. Stream frames from the agent. An MJPEG stream exists for iOS (#266). Nothing was checked for macOS. Input would have to be forwarded back.
3. Embed the agent's view with `CAContext`/`CALayerHost`. This is private API and fragile.
4. Run the JIT in-process in your app. This gives real interaction, but a bad preview can crash the app, and the executor is currently built as a remote process.

To decide, read the agent's window creation first:
`git grep -n -E 'NSHostingView|NSWindow|CALayerHost|CAContext|MJPEG' -- 'previewsmcp/PreviewAgent/*' 'previewsmcp/PreviewsMacOS/*'`

Check the repo `LICENSE` before copying code into another repo. A thin menu-bar AppKit app (SwiftUI via `NSHostingView`) that shells out to `previewsmcp run` is the low-effort alternative. Open question: is interaction inside your window or isolation from preview crashes more important?

## Cleanup for later
- Replace the `file://` LLVM URL with a hosted archive, then update `url` and `sha256`. The provenance of the current archive is not recorded here: it was made in the first session, so write down how it was built.
- Deprecation warnings: `.text(_:metadata:)` and the `.image` equivalents. Eventually move to `.text(text:annotations:_meta:)`.
- Change `package.sh`'s default `LLVM_RELEASE_REPO`.
- Update the README (`obj-p/tap`, "Xcode 26.2").
- Fix the `obj-p` references in `.github/workflows/release.yml` and `scripts/generate-formula.sh`.
- Update the CI workflows that pin `Xcode-26.2.0.app`.
- Sort out the `.bazelversion` mismatch (9.2.0 vs installed 9.3.3).
- `LogsCommand.swift` help text mentions `~/.previewsmcp/serve.log`, which was missing in the first run.
- Consider contributing the stub-flag fix upstream. Confirm the canonical repo first.

## Suggested order when resuming
1. `git status`, `git branch -a`, and check for newer Xcode, macOS and upstream changes.
2. Rebuild, restart the daemon, and rerun `snapshot` and `run` to confirm everything still works.
3. Fix the test sources, get the suite running, and triage failures.
4. Test `serve` over stdio with a real MCP client, and test an asset-catalog view.
5. Build the clean commits on `macos27-support`, one per group above.
6. Host the LLVM archive, then clean up the `obj-p` and CI references.
