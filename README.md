# BetaDrop Upload Action

[![Listed on the GitHub Marketplace](https://img.shields.io/badge/Marketplace-BetaDrop%20Upload-2ea44f?logo=github)](https://github.com/marketplace/actions/betadrop-upload)

<!-- The Marketplace slug comes from action.yml's `name:` ("BetaDrop Upload"), NOT the repository
     name — /marketplace/actions/betadrop-upload-action is a 404. If the action is ever renamed,
     this link breaks silently, because a broken badge still renders as an image. -->

Upload an iOS `.ipa` or Android `.apk` from your CI pipeline to [BetaDrop](https://betadrop.app) and get an instant over-the-air install link — no TestFlight review wait, no Play track, no tester accounts. Testers open the link on their phone and install from the browser.

The install link lands in three places, so nobody has to go looking for it: a step **output** for
the rest of your workflow, the **run summary** page where the whole team can see it, and
optionally a **comment on the pull request** that updates itself on every push.

### What lands on the run page

Every green build ends with this on the workflow run's own summary — no step to add, and the whole
team sees it, not just whoever triggered the build. It is rendered below by the same Markdown the
action writes, so this is the panel itself rather than a picture of one (the link is an example):

> <img src="https://betadrop.app/logo.png" width="28" align="left" alt="BetaDrop">
>
> ### Build published to BetaDrop
>
> **[Install on a device](https://betadrop.app/install/?i=EXAMPLE)** · `https://betadrop.app/install/?i=EXAMPLE`
>
> Open it on a phone to install over the air — no TestFlight, no tester accounts.

With `comment-on-pr: true`, the same link also arrives on the pull request, and every later push
edits that comment instead of adding another:

> ### Beta build ready to install
>
> **[Install on a device](https://betadrop.app/install/?i=EXAMPLE)**
>
> `https://betadrop.app/install/?i=EXAMPLE`
>
> <sub>Open it on a phone to install over the air. Built from `0123456` · published by <a href="https://betadrop.app">BetaDrop</a></sub>

## Usage

```yaml
- name: Publish to BetaDrop
  id: betadrop
  uses: betadrop-app/upload-action@v1
  with:
    file: app/build/outputs/apk/release/*.apk
    token: ${{ secrets.BETADROP_TOKEN }}

- name: Use the link
  run: echo "Install at ${{ steps.betadrop.outputs.install-url }}"
```

### iOS example

```yaml
jobs:
  beta:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v7
      - name: Build the .ipa
        run: fastlane gym --export_method release-testing
      - name: Publish to BetaDrop
        uses: betadrop-app/upload-action@v1
        with:
          file: "*.ipa"
          token: ${{ secrets.BETADROP_TOKEN }}
```

fastlane is preinstalled on the `macos` images, so `fastlane gym` runs without a setup step —
use `bundle exec fastlane gym` only if your repository has a Gemfile pinning it.

`release-testing` is Apple's current name for ad-hoc distribution. gym copies this value
straight into the ExportOptions plist's `method` key, where Apple deprecated `ad-hoc` in
Xcode 15.4 — and Xcode 26, which is what `macos-latest` resolves to on GitHub-hosted
runners, rejects it outright. If your fastlane is old enough to reject the new name, pass
it through `export_options: { method: "release-testing" }` in a Fastfile instead. Either
way the export runs before this action does, so getting it wrong fails the build a step
early.

### Android example

```yaml
jobs:
  beta:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Set up JDK
        uses: actions/setup-java@v6
        with: { java-version: "17", distribution: "temurin" }
      - name: Build release APK
        run: ./gradlew assembleRelease
      - name: Publish to BetaDrop
        uses: betadrop-app/upload-action@v1
        with:
          file: app/build/outputs/apk/release/*.apk
          token: ${{ secrets.BETADROP_TOKEN }}
          notes: ${{ github.event.head_commit.message }}
```

### Post the link on the pull request

Set `comment-on-pr` and the reviewer gets the link without opening Actions:

```yaml
jobs:
  beta:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write   # required for the comment
    steps:
      - uses: actions/checkout@v7
      - run: ./gradlew assembleRelease
      - uses: betadrop-app/upload-action@v1
        with:
          file: app/build/outputs/apk/release/*.apk
          token: ${{ secrets.BETADROP_TOKEN }}
          comment-on-pr: true
```

Every later push to the branch **edits that same comment** rather than adding another, so a
twenty-commit PR ends with one comment holding the current build — not twenty holding stale ones.

If the comment cannot be posted (usually a missing `pull-requests: write`), the step says so as a
warning and the run stays green: the build published, and the link is still an output and still on
the run summary. On an event with no pull request — a push to `main`, a manual dispatch — the
comment is skipped silently.

Prefer to build the comment yourself? The link is a normal step output:

```yaml
- name: Publish to BetaDrop
  id: betadrop
  uses: betadrop-app/upload-action@v1
  with:
    file: "*.apk"
    token: ${{ secrets.BETADROP_TOKEN }}

- run: echo "Install at ${{ steps.betadrop.outputs.install-url }}"
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `file` | yes | — | Path to the `.ipa`/`.apk`. Globs allowed; must match exactly one file. |
| `token` | yes | — | A BetaDrop API token (`bd_live_…`). Create one under **Settings → Developer → API tokens** at [betadrop.app](https://betadrop.app), and store it as a repository secret. |
| `name` | no | — | Override the build name shown on the install page. |
| `notes` | no | — | Release notes for this build — e.g. the commit message. |
| `channel` | no | — | A [channel](https://betadrop.app/channels/) slug to point at this build once it is live — e.g. `nightly`. The channel's link, and any "Download from BetaDrop" button embedded with it, then serves this build with nothing to click. Pro and Studio plans; claim the channel first. A channel that cannot be used fails the step **before** anything is uploaded. Needs `@betadrop/cli` 0.3.0+ — the step verifies this before uploading. |
| `comment-on-pr` | no | `false` | Post the install link as a comment on the pull request, editing that same comment on later pushes. Needs `pull-requests: write`. Skipped on events with no PR. |
| `github-token` | no | `${{ github.token }}` | Token used for that comment. Override only if it should come from another account. |
| `cli-version` | no | `latest` | Version of [`@betadrop/cli`](https://www.npmjs.com/package/@betadrop/cli) to use. Pin for reproducible pipelines. |

## Outputs

| Output | Description |
| --- | --- |
| `install-url` | The OTA install link for the uploaded build. |
| `channel-url` | The channel's stable install link when `channel` was set; empty otherwise. |

### One link that follows every build

```yaml
- name: Publish to BetaDrop
  id: betadrop
  uses: betadrop-app/upload-action@v1
  with:
    file: app/build/outputs/apk/release/*.apk
    token: ${{ secrets.BETADROP_TOKEN }}
    channel: nightly

- run: echo "Testers keep using ${{ steps.betadrop.outputs.channel-url }}"
```

If the build uploads but the channel could not be pointed at it, the step fails — a pipeline that
asked for a channel must not go green when the channel did not move.

`channel` needs [`@betadrop/cli`](https://www.npmjs.com/package/@betadrop/cli) **0.3.0 or newer**.
The step checks the CLI it installed before it uploads anything, and names the version it found if
that version is too old — so a run either points the channel or tells you exactly why it cannot.
Every other input works on any published version.

### A download button that follows the channel

Wrap the channel link in the "Download from BetaDrop" badge and paste it in your README — one
Markdown image link, no script. With `channel:` set on the step above, every merge updates what
the button installs and the README never changes.

```markdown
[![Download MyApp from BetaDrop](https://betadrop.app/badge/download-dark.svg)](https://betadrop.app/install/nightly?ref=badge)
```

Copy the exact snippet (dark or light, HTML or Markdown) from the embed panel under any build's
install link. Details and guidelines: <https://betadrop.app/download-button/>.

### Pin the CLI for a reproducible pipeline

`cli-version` defaults to `latest`, which means a green pipeline can change behaviour on a day you
did not touch it. Pin it once you are past setup:

```yaml
- uses: betadrop-app/upload-action@v1
  with:
    file: "*.ipa"
    token: ${{ secrets.BETADROP_TOKEN }}
    cli-version: "0.2.13"
```

## Troubleshooting

**`the installed @betadrop/cli … has no --channel flag`** — the version that resolved is older than
0.3.0, which is where `channel` landed. Set `cli-version` to 0.3.0 or newer; if `latest` itself is
older than that, the flag has not reached npm yet and the rest of the action still works without
`channel`.

**The PR comment never appears** — check the run log for the warning. The usual cause is the job
lacking `permissions: pull-requests: write`; the other is an event with no pull request, which is
reported as a notice. Forked-PR runs get a read-only token by default and cannot comment at all.

**`No file matches '…'`** — the glob ran before the build wrote anything, or wrote somewhere else.
Add a `run: ls -R` step before this one to see what the build actually produced. A pattern that
resolves only to directories counts as no match.

**`'…' matches N files`** — deliberate. Publishing an arbitrary one of several builds is worse than
failing, so narrow the pattern until it names one.

**`Publish did not end with an install URL`** — the CLI printed something after the link, which
breaks the contract this action reads. Report it; the last line of the run log is the useful detail.

**It downloads on the phone but will not open (iOS)** — see the note about UDIDs below.

## Notes

- Works on `ubuntu`, `macos`, and `windows` runners (Node is preinstalled on all GitHub-hosted images) — this repo's own [selftest](.github/workflows/selftest.yml) runs the shipped script on all three.
- A glob matching **more than one** file fails the step rather than uploading an arbitrary build — narrow the pattern.
- Pass `token` as a repository secret. A token pasted literally into a workflow is registered as a mask by the step, but it is still committed to your repository in clear.
- Build links follow your account's retention settings; see [pricing](https://betadrop.app/pricing/).
- An **ad-hoc iOS build uploaded from CI still only installs on registered devices**. If a tester reports "it downloads but won't open", their UDID was not in the provisioning profile when the IPA was signed. Send them the [UDID checker](https://betadrop.app/udid-checker/) — it reads the UDID off the iPhone itself, so they don't need a Mac or a cable — then add the device in the Apple Developer portal and re-sign.
- This action is a thin wrapper over the same [`@betadrop/cli`](https://www.npmjs.com/package/@betadrop/cli) you'd use locally, so CI behaves exactly like your machine.

## License

MIT
