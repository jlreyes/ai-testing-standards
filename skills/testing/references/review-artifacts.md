# Review artifacts

Record the real validation run so a reviewer can see what happened and try the
result. Start recording before exercising the changed behavior, and keep a
useful successful run. A recording of an automated test can show the actual
user flow; passing test output alone does not show the interface.

## Choose the evidence and tools

Use tools already available in the project or environment. Check their local
help and capabilities before installing a tool. The repository's testing
context owns exact build schemes, target machines, capture paths, and upload
destinations. Preview servers are separate work.

| Surface | Evidence to retain | Existing tools to reach for |
| --- | --- | --- |
| Native macOS app | Video of the changed flow, plus screenshots of states needing visual judgment | Cua recording and screenshots where connected; on macOS, its video capture is native. Built-in `screencapture` is another option on a host with a display and capture permission. |
| iOS simulator | Video of the changed flow and selected screenshots | `xcrun simctl io <device> recordVideo --codec=h264 demo.mp4` and `xcrun simctl io <device> screenshot result.png` |
| Web app | Video of actual interaction and screenshots of changed states | The project's Playwright recording and screenshot facilities, or an available browser/Cua capture tool |
| CLI | Commands, output, and exit status; video when timing or a terminal interface matters | Built-in `script` for a transcript; existing screen capture for video. Asciinema or VHS are optional when the task benefits from them. |
| Changed client build | A downloadable package from the tested revision, with launch instructions | The project's build/export tools. For macOS, Xcode tools and `ditto` can produce a zipped `.app`; GitHub release assets can host it. Use the platform's installable format for other clients. |

Cua video is off by default: request `record_video: true` when starting a
recording, then stop it to finalize the MP4. Its per-action screenshots can help
select review states. Check the connected tool's session and recording ownership
rules, use the intended development computer, and keep unrelated desktop content
out of the capture. A headless host without a display cannot supply a screen
recording merely because the CLI is installed.

For Playwright, enable video for the validation run, rather than retaining only
failures or retries. Close the browser context to finalize the file. For
`simctl`, wait for recording to start and send SIGINT to stop and finalize it.

A terminal transcript must contain the invocation as well as its result. On
macOS, `script -q cli.txt /bin/zsh -f` starts a recorded shell; run the intended
commands there and record each relevant exit status immediately afterward.
Do not reconstruct a successful interaction from memory or treat a transcript
wrapper's exit status as the tested command's status.

## Make the evidence reviewable

Keep the clip focused on the changed behavior, with enough setup and result to
understand it. Add screenshots where appearance needs inspection, including a
before view when it makes a visual change easier to judge. Preserve failures
that explain a remaining limitation. Trimming idle time is fine; editing must
not imply that a failed or untested step succeeded.

Before sharing, open the screenshots, play the video, or read the transcript.
Check that the behavior is visible, the output is legible, and the recording
finished.
Use appropriate test data and inspect for secrets or unrelated private content.
Identify the tested commit, relevant environment, scenario, and observed result
beside the links. If the tested tree was dirty, identify that difference instead
of attributing the evidence to the commit alone.

## Package something the reviewer can try

Build from the tested revision using the project's normal export path. For a Mac
app, package the exported bundle with, for example:

```sh
ditto -c -k --sequesterRsrc --keepParent MyApp.app MyApp.zip
```

A routine download-and-open experience requires the project's Developer ID
signing and notarization setup. A simulator bundle is not an installable iPhone
build. State the platform, architecture, minimum OS, and any actual installation
or signing limitation beside the download.

Download, unpack, and launch the shared package in the intended test environment;
launching only the development build does not establish that the download works.
Use a PR-and-commit-specific asset name so later builds do not silently replace
what the evidence describes. Put binaries in the project's artifact storage,
not in source history. If packaging or installation cannot be completed, report
that gap and its cause rather than calling the build ready to try.

## Share with the PR

Attach media to the PR description or a linked validation comment. GitHub CLI
supports image/video `--attach` on PR create, edit, and comment in version 2.99.0
and later; confirm the installed command supports it. For example:

```sh
gh pr comment 123 --body-file validation.md \
  --attach demo.mp4 \
  --attach 'result.png#Result after saving'
```

Keep command transcripts in a readable fenced block, optionally under a
collapsible section. A gist is suitable for shareable output; secret gists are
unlisted, not private. Use the private repository's PR or artifact storage for
private evidence.

Upload a client package to the project's existing artifact destination. GitHub
prerelease assets are a CLI-accessible option without CI: create a prerelease
for the exact PR commit, upload with `gh release upload`, and link the asset
from the PR. Retain repository access controls and use publishing paths
authorized for the task. Verify the published links and reviewer access; a local file path is not
a downloadable PR artifact.

After a change that affects the demonstrated behavior or package, regenerate the
affected evidence. Label older evidence with its revision. Formatting or PR-text
edits do not require another recording. If capture or upload is blocked, keep
the available local evidence and say what is missing and why.

## Tool references

- [Playwright videos](https://playwright.dev/docs/videos)
- [GitHub CLI media attachments](https://github.blog/changelog/2026-09-01-github-cli-media-in-issues-pull-requests-and-comments/)
- [GitHub release uploads](https://cli.github.com/manual/gh_release_upload)
- [Gist visibility](https://docs.github.com/en/get-started/writing-on-github/editing-and-sharing-content-with-gists/creating-gists)
- [Apple notarization](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)
