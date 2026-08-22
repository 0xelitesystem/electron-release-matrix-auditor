# Electron Release Matrix Auditor

Paste a GitHub Actions workflow. It reports which release artifacts the matrix legs will fight over, and whether two legs can create two draft releases for one tag.

**Live demo:** https://0xelitesystem.github.io/electron-release-matrix-auditor/

An Electron release matrix fails quietly. Every leg exits 0, every step is green, and the release page looks fine right up until someone tries to auto-update. This tool reads the workflow file and derives, from the publisher's own source, the two ways that happens.

## Use

1. Open the live demo, or clone and open `index.html`.
2. Paste the contents of `.github/workflows/release.yml`, or press one of the sample buttons.
3. Read the findings.

Two sample buttons load real workflows from this account: the release workflow that produced two draft releases for a single tag, and the two-job rewrite that fixed it. Same tool, same author, one broken and one clean. Three more synthetic samples cover an arch-split matrix, a dynamic `fromJson` matrix, and a `softprops/action-gh-release` publisher.

### What it finds

**The draft race.** `electron-builder`'s `getOrCreateRelease()` lists the repository's releases, looks for a draft carrying this tag, and POSTs a new one when it does not find it. There is no lock between the read and the create. Two matrix legs that both list before either creates will both create, and GitHub accepts both, because a draft release has no git tag behind it, so duplicate `tag_name` strings on drafts are legal. Nothing returns an error. You get two drafts with the installers split between them.

This finding is publisher-specific by design. `softprops/action-gh-release` catches a 422 `already_exists` on creation, retries into the existing release, and has a function whose entire job is deleting the duplicate draft it just created. Flagging every parallel Electron release workflow would make the tool useless, since that shape is the common one.

**The manifest collision.** Ported straight from `updateInfoBuilder.ts`: the OS suffix is empty for Windows, and the arch prefix is empty unless the platform is Linux. So `latest-mac.yml` is written by every mac architecture, and `latest.yml` is written by both `win/x64` and `win/arm64`. When two legs upload the same name, `overwriteArtifact()` logs a warning, DELETEs the existing asset, and uploads its own. Every installer stays on the release page and one architecture silently loses auto-update. This finding is publisher-independent, because `softprops` resolves the same conflict the same way.

The mac half has a paper trail in electron-builder issue 5592. The Windows half falls out of the same two functions and appears to have no issue filed at all, which matters now that Arm Windows runners are generally available and `[windows-latest, windows-11-arm]` is an ordinary matrix to write.

**The second-order trap.** `gh release view` will not tell you which draft is which. `FetchRelease` in `cli/cli` starts two goroutines, a published lookup and a draft lookup, into a two-slot buffered channel, and returns whichever answers first. And the draft lookup asks GraphQL for `release(tagName:)`, singular, so a duplicate draft is not merely deprioritised, it is invisible to the command line.

### What it will not do

It bails, loudly, on a matrix built from another job's output with `fromJson`, because that matrix does not exist in the file. It does not read `package.json`, so installer file names, product name, version and configured target arches are out of scope; only update manifests are analysed. It does not follow reusable workflows. It parses a practical subset of YAML (nested maps, block sequences, sequences of maps, flow maps, flow sequences, quoted scalars, block scalars) and reports a parse failure rather than guessing when it meets anchors, aliases, merge keys or multi-document streams.

The `runs-on` to platform and arch table is visible in the page and fully editable. It is consulted only when a leg carries no explicit architecture, and an unrecognised label is reported as unknown rather than guessed.

## Why this exists

Every existing answer to "will my release matrix collide" is a GitHub issue thread. Threads go stale, and the one everybody links to was closed by its own reporter two days after filing, which reads as resolved to anyone skimming. This is the derivation instead of the anecdote: the two functions that build the manifest name are ported into the page and run over your workflow, so the answer stays right for matrices nobody anticipated.

It is one HTML file with no dependencies, no build step, no telemetry and no network calls of any kind. Your workflow file is often the shape of your whole release pipeline, and it should not have to be uploaded to anyone to get an answer about it. MIT licensed, so vendor it into a repo or a CI check if that is useful.

## Privacy

Everything runs in your browser. Nothing leaves your machine. There is no analytics, no font loading, no CDN and no fetch of any kind. The only thing stored is your edited runner table and your theme choice, in this browser's `localStorage`, and both stay there. Open the network tab and confirm it.

## Run locally

```
git clone https://github.com/0xelitesystem/electron-release-matrix-auditor.git
cd electron-release-matrix-auditor
```

Open `index.html` in a browser. That is the whole thing. If you would rather serve it:

```
python -m http.server 8000
```

Then open http://localhost:8000/.

## Build

There is no build. One file, inline CSS and JS, no dependencies, no toolchain.

## Sources

Every quoted excerpt on the page cites its file and function and was read from the default branch of these repositories.

- `packages/app-builder-lib/src/publish/updateInfoBuilder.ts` and `packages/electron-publish/src/gitHubPublisher.ts` in https://github.com/electron-userland/electron-builder
- `pkg/cmd/release/shared/fetch.go` in https://github.com/cli/cli
- `src/github.ts` in https://github.com/softprops/action-gh-release

## License

MIT. Copyright (c) 2026 0xelitesystem.

## Related

- https://github.com/0xelitesystem/jwt-inspector
- https://github.com/0xelitesystem/json-workbench
- https://github.com/0xelitesystem/regex-studio
