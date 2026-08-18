# BetaDrop Upload Action

Upload an iOS `.ipa` or Android `.apk` from your CI pipeline to [BetaDrop](https://betadrop.app) and get an instant over-the-air install link — no TestFlight review wait, no Play track, no tester accounts. Testers open the link on their phone and install from the browser.

The install link is exposed as a step output and added to the workflow run's summary page, so every green build ends with a tester-ready link.

## Usage

```yaml
- name: Publish to BetaDrop
  id: betadrop
  uses: BetaDropApp/upload-action@v1
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
        run: bundle exec fastlane gym --export_method ad-hoc
      - name: Publish to BetaDrop
        uses: BetaDropApp/upload-action@v1
        with:
          file: "*.ipa"
          token: ${{ secrets.BETADROP_TOKEN }}
```

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
        uses: BetaDropApp/upload-action@v1
        with:
          file: app/build/outputs/apk/release/*.apk
          token: ${{ secrets.BETADROP_TOKEN }}
          notes: ${{ github.event.head_commit.message }}
```

### Post the link on a pull request

```yaml
- name: Publish to BetaDrop
  id: betadrop
  uses: BetaDropApp/upload-action@v1
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
| `cli-version` | no | `latest` | Version of [`@betadrop/cli`](https://www.npmjs.com/package/@betadrop/cli) to use. Pin for reproducible pipelines. |

## Outputs

| Output | Description |
| --- | --- |
| `install-url` | The OTA install link for the uploaded build. |

## Notes

- Works on `ubuntu`, `macos`, and `windows` runners (Node is preinstalled on all GitHub-hosted images).
- A glob matching **more than one** file fails the step rather than uploading an arbitrary build — narrow the pattern.
- Build links follow your account's retention settings; see [pricing](https://betadrop.app/pricing/).
- This action is a thin wrapper over the same [`@betadrop/cli`](https://www.npmjs.com/package/@betadrop/cli) you'd use locally, so CI behaves exactly like your machine.

## License

MIT
