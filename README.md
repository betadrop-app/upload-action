# BetaDrop Upload Action

[![Listed on the GitHub Marketplace](https://img.shields.io/badge/Marketplace-BetaDrop%20Upload-2ea44f?logo=github)](https://github.com/marketplace/actions/betadrop-upload)

<!-- The Marketplace slug comes from action.yml's `name:` ("BetaDrop Upload"), NOT the repository
     name — /marketplace/actions/betadrop-upload-action is a 404. If the action is ever renamed,
     this link breaks silently, because a broken badge still renders as an image. -->

Upload an iOS `.ipa` or Android `.apk` from your CI pipeline to [BetaDrop](https://betadrop.app) and get an instant over-the-air install link — no TestFlight review wait, no Play track, no tester accounts. Testers open the link on their phone and install from the browser.

The install link is exposed as a step output and added to the workflow run's summary page, so every green build ends with a tester-ready link.

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
      - uses: actions/checkout@v4
      - name: Build the .ipa
        run: bundle exec fastlane gym --export_method release-testing
      - name: Publish to BetaDrop
        uses: betadrop-app/upload-action@v1
        with:
          file: "*.ipa"
          token: ${{ secrets.BETADROP_TOKEN }}
```

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
      - uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
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

### Post the link on a pull request

```yaml
- name: Publish to BetaDrop
  id: betadrop
  uses: betadrop-app/upload-action@v1
  with:
    file: "*.apk"
    token: ${{ secrets.BETADROP_TOKEN }}

- name: Comment the install link
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        ...context.repo,
        issue_number: context.issue.number,
        body: `📲 Beta build ready: ${{ steps.betadrop.outputs.install-url }}`,
      })
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `file` | yes | — | Path to the `.ipa`/`.apk`. Globs allowed; must match exactly one file. |
| `token` | yes | — | A BetaDrop API token (`bd_live_…`). Create one under **Settings → Developer → API tokens** at [betadrop.app](https://betadrop.app), and store it as a repository secret. |
| `name` | no | — | Override the build name shown on the install page. |
| `notes` | no | — | Release notes for this build — e.g. the commit message. |
| `channel` | no | — | A [channel](https://betadrop.app/channels/) slug to point at this build once it is live — e.g. `nightly`. The channel's link, and any "Download from BetaDrop" button embedded with it, then serves this build with nothing to click. Pro and Studio plans; claim the channel first. A channel that cannot be used fails the step **before** anything is uploaded. Needs `@betadrop/cli` 0.3.0+. |
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

## Notes

- Works on `ubuntu`, `macos`, and `windows` runners (Node is preinstalled on all GitHub-hosted images).
- A glob matching **more than one** file fails the step rather than uploading an arbitrary build — narrow the pattern.
- Build links follow your account's retention settings; see [pricing](https://betadrop.app/pricing/).
- An **ad-hoc iOS build uploaded from CI still only installs on registered devices**. If a tester reports "it downloads but won't open", their UDID was not in the provisioning profile when the IPA was signed. Send them the [UDID checker](https://betadrop.app/udid-checker/) — it reads the UDID off the iPhone itself, so they don't need a Mac or a cable — then add the device in the Apple Developer portal and re-sign.
- This action is a thin wrapper over the same [`@betadrop/cli`](https://www.npmjs.com/package/@betadrop/cli) you'd use locally, so CI behaves exactly like your machine.

## License

MIT
