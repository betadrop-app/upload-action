# Security

## Reporting a vulnerability

Email **security@betadrop.app** — please do not open a public issue for anything exploitable.
We aim to acknowledge within two working days.

## What this action does with your token

`token` is passed to the CLI through the step's environment, never interpolated into a shell
command and never written to a file. The step registers it as a mask on the runner, so a token
that is not a `secrets.*` reference is still redacted from the log.

Two consequences worth knowing:

- **The token is only as scoped as you make it.** A BetaDrop API token can publish builds to your
  account. Create a separate token for CI at *Settings → Developer → API tokens* so you can revoke
  it without touching anyone's local CLI.
- **`cli-version: latest` resolves at run time.** The action installs `@betadrop/cli` from npm on
  every run, so a compromised release of that package would run in your job. Pin `cli-version` for
  pipelines where that matters — see the README.

## Scope

This repository is one YAML file. Reports about the BetaDrop service itself, the install pages, or
`@betadrop/cli` are welcome at the same address; say which one you mean.
