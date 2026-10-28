# ios-release

A reusable GitHub Actions workflow that builds an iOS app on Linux with
[omarchy-apple-dev](https://github.com/joshuaswarren/omarchy-apple-dev), uploads it
to App Store Connect, and attaches the `.ipa` to the GitHub release that triggered
it. In App Store Connect, the build appears in TestFlight. You can also submit it
for App Store review from there. The workflow never submits for review itself.

## Requirements

- An [xtool](https://github.com/xtool-org/xtool) project: `Package.swift` plus
  `xtool.yml`, with `infoPath` pointing to the app's Info.plist.
- A self-hosted Linux runner with omarchy-apple-dev installed in full mode (the
  toolchain and the iOS SDK), registered with the label `omarchy`.
- An App Store Connect team API key with the App Manager role and access to
  Certificates, Identifiers & Profiles. Store it as three secrets in the app's
  repository: `ASC_KEY_ID`, `ASC_ISSUER_ID` and `ASC_KEY_P8` (the `.p8` file's
  contents).
- The app record created in App Store Connect.

## Usage

`.github/workflows/release.yml` in the app's repository:

```yaml
name: Release

on:
  release:
    types: [published]

permissions:
  contents: write

jobs:
  release:
    uses: breitburg/ios-release/.github/workflows/release.yml@<commit sha> # v1
    secrets:
      ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
      ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
      ASC_KEY_P8: ${{ secrets.ASC_KEY_P8 }}
```

Pin a full commit SHA, not a tag: a tag can be moved, and a commit can't.

Release with a tag `v<version>+<build>`. The version becomes
`CFBundleShortVersionString` and the build becomes `CFBundleVersion`:

```sh
gh release create v1.0+5 --generate-notes
```

The build number must be higher than every build already uploaded for that
version.

### Inputs

| Input | Default | |
|---|---|---|
| `runs-on` | `["self-hosted", "Linux", "omarchy"]` | Runner labels, as a JSON list |
| `info-plist` | `infoPath` from `xtool.yml` | Info.plist to set the version in |
| `omarchy-apple-dev` | `~/.local/share/omarchy-apple-dev` | omarchy-apple-dev on the runner |

## Security

- This repository holds no secrets. The caller passes the API key in, and the
  workflow writes it to a file only the runner user can read, then deletes it
  when the job ends, whether the job passed or failed. Neither the key nor the
  tokens made from it are printed.
- The release tag and the inputs reach scripts only through environment
  variables, so a crafted tag can't inject shell code. The version is set only
  in a file inside the checked-out repository.
- The job asks for one permission, `contents: write`, to attach the `.ipa`.
- `ship.sh` keeps the Apple Distribution private key on the runner. Every
  workflow that runs on that runner runs as the same user and could read it.
  Register the runner only with private repositories you trust, and never with
  a public repository: on a public repository, a pull request from a fork could
  run code on your machine.
