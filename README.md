# release-infra

Shared reusable GitHub Actions workflows for cross-platform building, signing, and publishing CLI tools.

## Overview

A tag push (`v*`) on any adopting repository triggers a complete build-sign-package-publish pipeline. Each distribution channel is independent and opt-in.

## Required Variables

Set these as **GitHub Actions repository variables** (Settings > Secrets and variables > Actions > Variables) on each adopting repository, or as **organization variables** to share across repos:

| Variable | Description | Example |
|----------|-------------|--------|
| `PACKAGES_DOMAIN` | Domain for self-hosted apt/rpm repos | `packages.example.com` |
| `INSTALL_DOMAIN` | Domain for installer script hosting | `install.example.com` |
| `MAINTAINER_NAME` | Maintainer display name | `Jane Doe` |
| `MAINTAINER_EMAIL` | Maintainer email address | `jane@example.com` |
| `AUR_USERNAME` | AUR commit username | `janedoe` |
| `WINGET_ID_PREFIX` | winget package ID prefix | `JaneDoe` |

## Workflows

| Workflow | Purpose |
|----------|--------|
| `release.yml` | **Orchestrator** - composes all workflows below |
| `build-rust.yml` | Cross-compile Rust for target matrix |
| `build-node.yml` | Build JS/TS projects |
| `sign-windows.yml` | Azure Trusted Signing |
| `sign-macos.yml` | Apple codesign + notarytool |
| `release-github.yml` | Create GitHub Release with archives + SHA256SUMS |
| `publish-cargo.yml` | `cargo publish` to crates.io |
| `publish-npm.yml` | `npm publish` to npm |
| `publish-homebrew.yml` | Generate formula, push to tap repo |
| `publish-scoop.yml` | Generate manifest, push to bucket repo |
| `publish-chocolatey.yml` | Build nupkg, push to community repo |
| `publish-winget.yml` | Generate manifest, PR to winget-pkgs |
| `package-msi.yml` | Build MSI via cargo-wix, sign |
| `package-dmg.yml` | Build DMG, sign, notarize |
| `package-linux.yml` | nfpm: deb + rpm + apk |
| `publish-apt.yml` | Add deb to apt repo via dispatch |
| `publish-rpm.yml` | Add rpm to rpm repo via dispatch |
| `publish-aur.yml` | Update PKGBUILD, push to AUR |

## Target Matrix

| Target | Runner | Signing |
|--------|--------|--------|
| `x86_64-apple-darwin` | `macos-latest` | Apple codesign + notarize |
| `aarch64-apple-darwin` | `macos-latest` | Apple codesign + notarize |
| `x86_64-pc-windows-msvc` | `windows-latest` | Azure Trusted Signing |
| `aarch64-pc-windows-msvc` | `windows-latest` | Azure Trusted Signing |
| `x86_64-unknown-linux-gnu` | `ubuntu-latest` | None |
| `aarch64-unknown-linux-gnu` | `ubuntu-latest` (cross) | None |
| `x86_64-unknown-linux-musl` | `ubuntu-latest` (cross) | None |
| `aarch64-unknown-linux-musl` | `ubuntu-latest` (cross) | None |

## Per-Project Adoption

### 1. Create caller workflow

Add `.github/workflows/release.yml` to your project:

```yaml
name: Release

on:
  push:
    tags: ["v*"]

jobs:
  release:
    uses: <your-org>/release-infra/.github/workflows/release.yml@v1
    with:
      project-name: myproject
      binary-name: myproject
      project-type: rust        # or "node"
      bundle-id: com.example.myproject
      targets: >-
        x86_64-apple-darwin,
        aarch64-apple-darwin,
        x86_64-pc-windows-msvc,
        x86_64-unknown-linux-gnu,
        aarch64-unknown-linux-gnu
      enable-homebrew: true
      enable-scoop: true
      enable-chocolatey: false
      enable-winget: false
      enable-apt: true
      enable-rpm: true
      enable-apk: false
      enable-dmg: false
      enable-aur: true
    secrets: inherit
```

### 2. Set repository variables

Configure the variables listed in the **Required Variables** section above on your repository (or organization).

### 3. Add packaging configs (`dist/` directory)

Only needed for channels you enable:

```
dist/
├── nfpm.yaml              # Required for apt/rpm/apk
├── chocolatey/            # Required for Chocolatey
│   ├── package.nuspec
│   └── tools/
│       ├── chocolateyInstall.ps1
│       └── chocolateyUninstall.ps1
├── PKGBUILD               # Required for AUR
└── wix/                   # Required for MSI (Chocolatey/winget)
    └── main.wxs
```

### 4. Configure secrets

Required secrets depend on enabled features:

| Secret | Required for |
|--------|-------------|
| `AZURE_TENANT_ID` | Windows signing |
| `AZURE_CLIENT_ID` | Windows signing |
| `AZURE_CLIENT_SECRET` | Windows signing |
| `AZURE_CODE_SIGNING_ACCOUNT` | Windows signing |
| `AZURE_CERT_PROFILE` | Windows signing |
| `APPLE_CERT_BASE64` | macOS signing |
| `APPLE_CERT_PASSWORD` | macOS signing |
| `APPLE_TEAM_ID` | macOS signing |
| `APPLE_ID` | macOS signing |
| `APPLE_ID_PASSWORD` | macOS signing |
| `CARGO_REGISTRY_TOKEN` | cargo publish |
| `NPM_TOKEN` | npm publish |
| `HOMEBREW_TAP_TOKEN` | Homebrew |
| `SCOOP_BUCKET_TOKEN` | Scoop |
| `CHOCO_API_KEY` | Chocolatey |
| `WINGET_PAT` | winget |
| `PACKAGES_REPO_TOKEN` | apt/rpm |
| `AUR_SSH_PRIVATE_KEY` | AUR |

### 5. Release

```bash
git tag v1.0.0
git push origin v1.0.0
```

The pipeline handles everything from there.

## GPG Key Rotation (Package Repository Signing)

1. Generate a new GPG key pair:
   ```bash
   gpg --full-generate-key  # RSA 4096, 2-3 year expiry
   ```
2. Export the private key:
   ```bash
   gpg --armor --export-secret-keys KEY_ID > private.key
   ```
3. Export the public key:
   ```bash
   gpg --armor --export KEY_ID > gpg.key
   ```
4. Update `PKG_GPG_PRIVATE_KEY` and `PKG_GPG_PASSPHRASE` secrets on the `packages` repo
5. Replace `gpg.key` on the `gh-pages` branch of the `packages` repo
6. Users will need to re-import the new public key

## User Install Instructions

Replace `<packages-domain>`, `<install-domain>`, `<your-org>`, and `<project>` below with your configured values.

### Homebrew (macOS/Linux)
```bash
brew tap <your-org>/tap
brew install <your-org>/tap/<project>
```

### Scoop (Windows)
```powershell
scoop bucket add <your-org> https://github.com/<your-org>/scoop-bucket
scoop install <project>
```

### Chocolatey (Windows)
```powershell
choco install <project>
```

### apt (Debian/Ubuntu)
```bash
curl -fsSL https://<packages-domain>/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/packages.gpg
echo "deb [signed-by=/etc/apt/keyrings/packages.gpg] https://<packages-domain>/apt stable main" | sudo tee /etc/apt/sources.list.d/packages.list
sudo apt update
sudo apt install <project>
```

### rpm (Fedora/RHEL/CentOS)
```bash
sudo curl -o /etc/yum.repos.d/packages.repo https://<packages-domain>/rpm/packages.repo
sudo dnf install <project>
```

### AUR (Arch Linux)
```bash
yay -S <project>-bin
```

### Shell installer (macOS/Linux)
```bash
curl -fsSL https://<install-domain>/<project>/install.sh | sh
```

### PowerShell installer (Windows)
```powershell
irm https://<install-domain>/<project>/install.ps1 | iex
```

### cargo
```bash
cargo install <project>
```

## Versioning

Shared workflows are versioned with tags (`@v1`, `@v2`). Breaking changes require a major version bump. Pin to a specific version in your caller workflow.
