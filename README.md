# release-infra

Shared reusable GitHub Actions workflows for cross-platform building, signing, and publishing CLI tools.

A tag push (`v*`) on any adopting repository triggers a complete build-sign-package-publish pipeline. Each distribution channel is independent and opt-in.

## Table of Contents

- [Workflows](#workflows)
- [Target Matrix](#target-matrix)
- [Prerequisites](#prerequisites)
  - [1. Windows Code Signing (Azure Trusted Signing)](#1-windows-code-signing-azure-trusted-signing)
  - [2. macOS Code Signing (Apple Developer)](#2-macos-code-signing-apple-developer)
  - [3. crates.io Publishing](#3-cratesio-publishing)
  - [4. npm Publishing](#4-npm-publishing)
  - [5. Homebrew Tap](#5-homebrew-tap)
  - [6. Scoop Bucket](#6-scoop-bucket)
  - [7. Chocolatey](#7-chocolatey)
  - [8. winget](#8-winget)
  - [9. Self-Hosted apt/rpm Repositories](#9-self-hosted-aptrpm-repositories)
  - [10. AUR (Arch User Repository)](#10-aur-arch-user-repository)
  - [11. Direct Install Scripts](#11-direct-install-scripts)
- [Per-Project Adoption](#per-project-adoption)
- [Secrets Reference](#secrets-reference)
- [Variables Reference](#variables-reference)
- [GPG Key Rotation](#gpg-key-rotation)
- [User Install Instructions](#user-install-instructions)
- [README Badges](#readme-badges)
- [Versioning](#versioning)

## Workflows

| Workflow | Purpose |
|----------|--------|
| `release.yml` | **Orchestrator** — composes all workflows below |
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
| `package-msi.yml` | Build MSI via cargo-wix, sign (requires `enable-msi: true`) |
| `package-dmg.yml` | Build DMG, sign, notarize |
| `package-linux.yml` | nfpm: deb + rpm + apk |
| `publish-apt.yml` | Add deb to apt repo via dispatch |
| `publish-rpm.yml` | Add rpm to rpm repo via dispatch |
| `publish-aur.yml` | Update PKGBUILD, push to AUR |
| `publish-install-scripts.yml` | Render and publish curl/PowerShell install scripts |
| `publish-badges.yml` | Generate per-channel badge SVGs and publish them to the packages repo |

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

## Prerequisites

Only set up the services for the channels you plan to enable. The core build and GitHub Release pipeline requires no external accounts.

### 1. Windows Code Signing (Azure Trusted Signing)

**Required for:** `sign-windows.yml`, `package-msi.yml`
**Cost:** Azure pay-as-you-go (Trusted Signing ~$9.99/month for the Basic SKU). No separate certificate purchase is required — Azure Trusted Signing is a managed service that provides the signing certificate.

#### Create an Azure account and subscription

1. Sign up at [portal.azure.com](https://portal.azure.com).
2. Create a subscription (pay-as-you-go is fine).

#### Create a Trusted Signing account

1. In the Azure Portal, search for **Trusted Signing** and create a new account.
2. Note the **account name** (e.g., `my-signing-account`). This becomes `AZURE_SIGNING_ACCOUNT`.
3. Note the **regional endpoint URL** (e.g., `https://eus.codesigning.azure.net`). This becomes `AZURE_SIGNING_ENDPOINT`.

#### Create a certificate profile

1. In your Trusted Signing account, go to **Certificate profiles** and create a new profile.
2. Choose the appropriate profile type (e.g., **Public Trust** for code that will be distributed publicly).
3. Complete identity validation as prompted.
4. Note the **certificate profile name** — this is `AZURE_CERT_PROFILE`.

#### Register an Azure AD application

1. In the Azure Portal, go to **Entra ID** (Azure Active Directory) > **App registrations** > **New registration**.
2. Name it (e.g., `GitHub Actions Code Signing`), register it.
3. Copy the **Application (client) ID** — this is `AZURE_CLIENT_ID`.
4. Go to **Certificates & secrets** > **New client secret**. Copy the secret value immediately (shown only once) — this is `AZURE_CLIENT_SECRET`.
5. Go to your Trusted Signing account > **Access control (IAM)**. Add a role assignment granting the app registration the **Trusted Signing Certificate Profile Signer** role.

#### Get your tenant ID

1. In the Azure Portal, go to **Entra ID** > **Overview**.
2. Copy the **Tenant ID** — this is `AZURE_TENANT_ID`.

#### Secrets to configure

| Secret | Value |
|--------|-------|
| `AZURE_TENANT_ID` | Azure AD tenant (directory) ID |
| `AZURE_CLIENT_ID` | App registration application (client) ID |
| `AZURE_CLIENT_SECRET` | App registration client secret value |
| `AZURE_SIGNING_ENDPOINT` | Regional endpoint URL (e.g., `https://eus.codesigning.azure.net`) |
| `AZURE_SIGNING_ACCOUNT` | Trusted Signing account name |
| `AZURE_CERT_PROFILE` | Certificate profile name in Trusted Signing |

---

### 2. macOS Code Signing (Apple Developer)

**Required for:** `sign-macos.yml`, `package-dmg.yml`
**Cost:** $99/year (Apple Developer Program)

#### Enroll in the Apple Developer Program

1. Sign up at [developer.apple.com](https://developer.apple.com/programs/).
2. Complete enrollment ($99/year). Requires an Apple ID with two-factor authentication enabled.

#### Create a Developer ID Application certificate

1. On a Mac, open **Keychain Access** > **Certificate Assistant** > **Request a Certificate from a Certificate Authority**. Save the CSR to disk.
2. In the [Apple Developer portal](https://developer.apple.com/account/resources/certificates/add), create a new **Developer ID Application** certificate using the CSR.
3. Download the certificate and double-click to install it in Keychain Access.

#### Export the certificate as a .p12 file

1. In Keychain Access, find the installed certificate under **My Certificates**.
2. Right-click > **Export**. Choose `.p12` format. Set a strong password.
3. Base64-encode the .p12 file:
   ```bash
   base64 -i Certificates.p12 | pbcopy
   ```
   The clipboard contents become `APPLE_CERTIFICATE`. The password you chose becomes `APPLE_CERTIFICATE_PASSWORD`.

#### Get your signing identity string

The signing identity is the full certificate name visible in Keychain Access, e.g.:

```
Developer ID Application: Jane Doe (AB12CD34EF)
```

This exact string (including the Team ID in parentheses) becomes `APPLE_SIGNING_IDENTITY`.

#### Create an App Store Connect API key for notarization

This is the recommended notarization auth method — it does not require your Apple ID or an app-specific password.

1. Log in to [App Store Connect](https://appstoreconnect.apple.com) > **Users and Access** > **Integrations** > **App Store Connect API**.
2. Create a new key with the **Developer** role.
3. Download the generated `.p8` file (it can only be downloaded once).
4. Note the **Key ID** (e.g., `ABC1234567`) — this is `APPLE_API_KEY_ID`.
5. Note the **Issuer ID** shown at the top of the API Keys page — this is `APPLE_API_ISSUER`.
6. The contents of the downloaded `.p8` file become `APPLE_API_KEY_CONTENT`.

#### Secrets to configure

| Secret | Value |
|--------|-------|
| `APPLE_CERTIFICATE` | Base64-encoded .p12 certificate + private key |
| `APPLE_CERTIFICATE_PASSWORD` | Password used when exporting the .p12 |
| `APPLE_SIGNING_IDENTITY` | Full signing identity string, e.g. `Developer ID Application: Name (TEAMID)` |
| `APPLE_API_ISSUER` | Issuer ID UUID from App Store Connect |
| `APPLE_API_KEY_ID` | Key ID from App Store Connect |
| `APPLE_API_KEY_CONTENT` | Contents of the downloaded `.p8` file |

---

### 3. crates.io Publishing

**Required for:** `publish-cargo.yml`
**Cost:** Free

1. Create an account at [crates.io](https://crates.io) (sign in with GitHub).
2. Go to **Account Settings** > **API Tokens**.
3. Create a new token with **publish-update** scope.
4. Copy the token — this is `CARGO_REGISTRY_TOKEN`.

Ensure the crate name in your `Cargo.toml` is available or you already own it.

---

### 4. npm Publishing

**Required for:** `publish-npm.yml`
**Cost:** Free (public packages)

1. Create an account at [npmjs.com](https://www.npmjs.com).
2. Go to **Account** > **Access Tokens**.
3. Generate a new **Automation** token (this type bypasses 2FA for CI use).
4. Copy the token — this is `NPM_TOKEN`.

Ensure the package name (or `@scope/name`) is available or you have publish rights.

---

### 5. Homebrew Tap

**Required for:** `publish-homebrew.yml`
**Cost:** Free

#### Create the tap repository

1. Create a GitHub repository named `homebrew-tap` (the `homebrew-` prefix is a Homebrew convention).
2. Add a `Formula/` directory (can contain a `.gitkeep` initially).

#### Create a Personal Access Token

1. Go to GitHub > **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens**.
2. Create a token scoped to the tap repository with **Contents: Read and write** permission.
3. Copy the token — this is `HOMEBREW_TAP_TOKEN`.

---

### 6. Scoop Bucket

**Required for:** `publish-scoop.yml`
**Cost:** Free

#### Create the bucket repository

1. Create a GitHub repository named `scoop-bucket`.
2. Add a `bucket/` directory (can contain a `.gitkeep` initially).

#### Create a Personal Access Token

1. Go to GitHub > **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens**.
2. Create a token scoped to the bucket repository with **Contents: Read and write** permission.
3. Copy the token — this is `SCOOP_BUCKET_TOKEN`.

---

### 7. Chocolatey

**Required for:** `publish-chocolatey.yml`
**Cost:** Free

1. Create an account at [community.chocolatey.org](https://community.chocolatey.org/account/Register).
2. Go to **Account** > **API Keys**.
3. Copy your API key — this is `CHOCO_API_KEY`.

> **Note:** First-time package submissions go through moderator review, which can take several days. Subsequent version updates are faster but still pass through automated checks.

Your project needs a `dist/chocolatey/` directory with:
- `package.nuspec` — Chocolatey package spec
- `tools/chocolateyInstall.ps1` — install script
- `tools/chocolateyUninstall.ps1` — uninstall script

Chocolatey packages the portable `.exe` ZIP — no MSI is required. `enable-chocolatey` is independent of `enable-msi`.

---

### 8. winget

**Required for:** `publish-winget.yml`
**Cost:** Free

1. You need a GitHub account (used to fork and submit PRs to `microsoft/winget-pkgs`).
2. Go to GitHub > **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)**.
3. Create a token with `public_repo` scope (required to fork and PR to the public `winget-pkgs` repo).
4. Copy the token — this is `WINGET_PAT`.

The workflow uses `wingetcreate` to automatically generate a manifest and submit a PR to `microsoft/winget-pkgs`. The PR goes through automated validation before merging.

winget uses portable `.exe` installers and does not require an MSI — the `publish-winget.yml` workflow submits a manifest directly using `wingetcreate`. No `enable-msi` flag is needed for winget alone.

---

### 9. Self-Hosted apt/rpm Repositories

**Required for:** `publish-apt.yml`, `publish-rpm.yml`
**Cost:** Free (GitHub Pages)

#### Create the packages repository

1. Create a GitHub repository (e.g., `packages`).
2. Create a `gh-pages` branch with the initial directory structure:
   ```
   apt/dists/stable/main/binary-amd64/
   apt/dists/stable/main/binary-arm64/
   apt/pool/main/
   rpm/x86_64/repodata/
   rpm/aarch64/repodata/
   apk/
   CNAME        # your packages domain, e.g., packages.example.com
   ```
3. Enable **GitHub Pages** on the `gh-pages` branch (Settings > Pages).
4. Add a workflow on the `main` branch that handles `repository_dispatch` events (`update-apt`, `update-rpm`). See the companion `packages` repo template for the full workflow.

#### Configure DNS

Add a CNAME record pointing your packages domain (e.g., `packages.example.com`) to `<your-org>.github.io`. GitHub Pages provides HTTPS automatically via Let's Encrypt.

#### Generate a GPG key pair for package signing

```bash
# Generate key (RSA 4096, set 2-3 year expiry)
gpg --full-generate-key

# List keys to get the KEY_ID
gpg --list-secret-keys --keyid-format long

# Export private key (store as PACKAGES repo secret)
gpg --armor --export-secret-keys KEY_ID > private.key

# Export public key (place on gh-pages branch as gpg.key)
gpg --armor --export KEY_ID > gpg.key
```

#### Create a Personal Access Token

1. Go to GitHub > **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens**.
2. Create a token scoped to the packages repository with:
   - **Contents: Read and write**
   - **Actions: Write** (needed for `repository_dispatch`)
3. Copy the token — this is `PACKAGES_REPO_TOKEN`.

#### Secrets to configure on the packages repository

| Secret | Value |
|--------|-------|
| `PKG_GPG_PRIVATE_KEY` | Armored GPG private key (entire contents of `private.key`) |
| `PKG_GPG_PASSPHRASE` | Passphrase for the GPG key |

#### Variables to configure on the packages repository

| Variable | Value |
|----------|-------|
| `PACKAGES_DOMAIN` | Your packages domain (e.g., `packages.example.com`) |
| `REPO_LABEL` | Label for the repository metadata (e.g., your org name) |

#### Secrets to configure on each project repository

| Secret | Value |
|--------|-------|
| `PACKAGES_REPO_TOKEN` | PAT with dispatch access to the packages repo |

Your project needs a `dist/nfpm.yaml` file for nfpm to build deb/rpm/apk packages. Use `${MAINTAINER}` as the maintainer value — the workflow substitutes it from `vars.MAINTAINER_NAME` and `vars.MAINTAINER_EMAIL` at build time.

---

### 10. AUR (Arch User Repository)

**Required for:** `publish-aur.yml`
**Cost:** Free

#### Create an AUR account

1. Register at [aur.archlinux.org](https://aur.archlinux.org/register).

#### Generate an SSH key pair

```bash
ssh-keygen -t ed25519 -C "aur" -f aur_key -N ""
```

1. Add the **public key** (`aur_key.pub`) to your AUR account: [My Account](https://aur.archlinux.org/account) > **SSH Public Key**.
2. The entire contents of the **private key** file (`aur_key`) become `AUR_SSH_PRIVATE_KEY`.

#### Register the package name

Before the first automated publish, register the package on AUR:

```bash
git clone ssh://aur@aur.archlinux.org/<project>-bin.git
cd <project>-bin
# Add an initial PKGBUILD and .SRCINFO, commit, and push
```

Your project needs a `dist/PKGBUILD` file. Use `${MAINTAINER}` as the maintainer value — the workflow substitutes it at build time.

---

### 11. Direct Install Scripts

**Required for:** `publish-install-scripts.yml`
**Cost:** Free (GitHub Pages via the packages repository)

Direct install scripts (`install.sh` for macOS/Linux, `install.ps1` for Windows) are rendered from the templates in `templates/` and served from the packages repository under `https://<PACKAGES_DOMAIN>/<project>/`. No separate hosting setup is required beyond the packages repository already described in [section 9](#9-self-hosted-aptrpm-repositories).

#### How it works

1. On release, `publish-install-scripts.yml` renders `install.sh` and `install.ps1` from the templates in this repository, substituting project-specific values.
2. The rendered scripts are attached to the GitHub Release.
3. A `repository_dispatch` event (`update-install-scripts`) is sent to the packages repository.
4. The packages repository workflow downloads the scripts from the release and commits them to the `gh-pages` branch under `<project>/`.
5. GitHub Pages serves them at `https://<PACKAGES_DOMAIN>/<project>/install.sh` and `https://<PACKAGES_DOMAIN>/<project>/install.ps1`.

#### Prerequisites

- The packages repository must already be configured (see [section 9](#9-self-hosted-aptrpm-repositories)).
- The `PACKAGES_DOMAIN` variable must be set on the packages repository (e.g. `packages.keathmilligan.net`).
- The `PACKAGES_REPO_TOKEN` secret must be configured on the project repository (same token used for apt/rpm dispatch).

No additional secrets or variables are needed beyond those already required for the packages repository.

---

## Per-Project Adoption

### 1. Create the caller workflow

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
      project-type: rust              # or "node"
      targets: >-
        x86_64-apple-darwin,
        aarch64-apple-darwin,
        x86_64-pc-windows-msvc,
        x86_64-unknown-linux-gnu,
        aarch64-unknown-linux-gnu
      tap-repo: <your-org>/homebrew-tap
      bucket-repo: <your-org>/scoop-bucket
      packages-repo: <your-org>/packages
      enable-homebrew: true
      enable-scoop: true
      enable-msi: false
      enable-chocolatey: false
      enable-winget: false
      enable-apt: true
      enable-rpm: true
      enable-apk: false
      enable-dmg: false
      enable-aur: true
      enable-install-scripts: true
    secrets: inherit
```

### 2. Set repository variables

Configure the [variables](#variables-reference) on your repository or organization.

### 3. Add packaging configs

Add a `dist/` directory with configs for the channels you enable:

```
dist/
├── nfpm.yaml              # Required for apt/rpm/apk
├── chocolatey/            # Required for Chocolatey
│   ├── package.nuspec
│   └── tools/
│       ├── chocolateyInstall.ps1
│       └── chocolateyUninstall.ps1
├── PKGBUILD               # Required for AUR
└── wix/                   # Required for MSI (enable-msi: true)
    └── main.wxs
```

### 4. Configure secrets

Add the [secrets](#secrets-reference) for the channels you enable.

### 5. Release

```bash
git tag v1.0.0
git push origin v1.0.0
```

The pipeline handles everything from there.

## Secrets Reference

All secrets are configured on the **project repository** (or organization) and forwarded to the shared workflows via `secrets: inherit`.

| Secret | Required when | Source |
|--------|--------------|--------|
| `AZURE_TENANT_ID` | Windows targets | Azure Portal > Entra ID |
| `AZURE_CLIENT_ID` | Windows targets | Azure Portal > App registration |
| `AZURE_CLIENT_SECRET` | Windows targets | Azure Portal > App registration > Client secrets |
| `AZURE_SIGNING_ENDPOINT` | Windows targets | Azure Portal > Trusted Signing > Regional endpoint URL |
| `AZURE_SIGNING_ACCOUNT` | Windows targets | Azure Portal > Trusted Signing account name |
| `AZURE_CERT_PROFILE` | Windows targets | Certificate profile name in Trusted Signing |
| `APPLE_CERTIFICATE` | macOS targets | Exported .p12 certificate, base64-encoded |
| `APPLE_CERTIFICATE_PASSWORD` | macOS targets | Password for the .p12 export |
| `APPLE_SIGNING_IDENTITY` | macOS targets | Full cert name, e.g. `Developer ID Application: Name (TEAMID)` |
| `APPLE_API_ISSUER` | macOS targets | Issuer ID UUID from App Store Connect |
| `APPLE_API_KEY_ID` | macOS targets | Key ID from App Store Connect |
| `APPLE_API_KEY_CONTENT` | macOS targets | Contents of the `.p8` API key file |
| `CARGO_REGISTRY_TOKEN` | `project-type: rust` | crates.io > Account > API Tokens |
| `NPM_TOKEN` | `project-type: node` | npmjs.com > Account > Access Tokens |
| `HOMEBREW_TAP_TOKEN` | `enable-homebrew` | GitHub fine-grained PAT (Contents: RW on tap repo) |
| `SCOOP_BUCKET_TOKEN` | `enable-scoop` | GitHub fine-grained PAT (Contents: RW on bucket repo) |
| `CHOCO_API_KEY` | `enable-chocolatey` | community.chocolatey.org > Account > API Keys |
| `WINGET_PAT` | `enable-winget` | GitHub classic PAT with `public_repo` scope |
| `PACKAGES_REPO_TOKEN` | `enable-apt`, `enable-rpm`, or `enable-install-scripts` | GitHub fine-grained PAT (Contents: RW, Actions: Write on packages repo) |
| `AUR_SSH_PRIVATE_KEY` | `enable-aur` | SSH private key registered with AUR account |

The **packages repository** also needs its own secrets (configured on that repo, not on the project):

| Secret | Source |
|--------|--------|
| `PKG_GPG_PRIVATE_KEY` | Armored GPG private key for signing apt/rpm metadata |
| `PKG_GPG_PASSPHRASE` | Passphrase for the GPG key |

## Variables Reference

Set as **GitHub Actions repository variables** (Settings > Secrets and variables > Actions > Variables) on each adopting repository, or as **organization variables** to share across repos:

| Variable | Description | Example | Used by |
|----------|-------------|---------|---------|
| `PACKAGES_DOMAIN` | Domain for self-hosted apt/rpm repos and install scripts | `packages.example.com` | packages repo workflow, `publish-install-scripts.yml` |
| `MAINTAINER_NAME` | Maintainer display name | `Jane Doe` | nfpm, PKGBUILD, AUR |
| `MAINTAINER_EMAIL` | Maintainer email address | `jane@example.com` | nfpm, PKGBUILD, AUR |
| `AUR_USERNAME` | AUR git commit username | `janedoe` | publish-aur.yml |
| `WINGET_ID_PREFIX` | winget package ID prefix | `JaneDoe` | publish-winget.yml |
| `REPO_LABEL` | Label for apt/rpm repo metadata | `myorg` | packages repo workflow |

## GPG Key Rotation

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
4. Update `PKG_GPG_PRIVATE_KEY` and `PKG_GPG_PASSPHRASE` secrets on the packages repo.
5. Replace `gpg.key` on the `gh-pages` branch of the packages repo.
6. Users will need to re-import the new public key.

## User Install Instructions

Replace `<packages-domain>`, `<your-org>`, and `<project>` with your configured values.

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
curl -fsSL https://<packages-domain>/<project>/install.sh | sh
```

### PowerShell installer (Windows)
```powershell
irm https://<packages-domain>/<project>/install.ps1 | iex
```

### cargo
```bash
cargo install <project>
```

## README Badges

When `enable-badges: true` is set in your caller workflow, the pipeline generates a static SVG badge file for each enabled distribution channel and publishes them to your packages repository (alongside install scripts) under `<project>/badges/`. They are then served via GitHub Pages.

CI and Release workflow status use standard GitHub Actions badges directly. Per-channel badges are version-stamped static SVGs updated on every release.

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `enable-badges` | No | `false` | Generate and publish per-channel badge SVGs to the packages repo |

The packages repository must already be configured (see [section 9](#9-self-hosted-aptrpm-repositories)) and `PACKAGES_REPO_TOKEN` must be set. The `update-badges` dispatch event must be handled by the packages repo workflow.

### Usage example

```yaml
jobs:
  release:
    uses: <your-org>/release-infra/.github/workflows/release.yml@v1
    with:
      project-name: mytool
      binary-name: mytool
      project-type: rust
      targets: >-
        x86_64-apple-darwin,
        aarch64-apple-darwin,
        x86_64-pc-windows-msvc,
        x86_64-unknown-linux-gnu
      packages-repo: <your-org>/packages
      enable-homebrew: true
      enable-apt: true
      enable-cargo: true
      enable-badges: true
    secrets: inherit
```

### README badge block

Add the following to your project `README.md`, substituting your packages domain, project name, and repo. CI and Release use native GitHub workflow badges; channel badges are served from your packages repo:

```markdown
[![CI](https://github.com/<your-org>/mytool/actions/workflows/ci.yml/badge.svg)](https://github.com/<your-org>/mytool/actions/workflows/ci.yml)
[![Release](https://github.com/<your-org>/mytool/actions/workflows/release.yml/badge.svg)](https://github.com/<your-org>/mytool/actions/workflows/release.yml)
[![Homebrew](https://<packages-domain>/mytool/badges/homebrew.svg)](https://github.com/<your-org>/mytool/releases/latest)
[![apt](https://<packages-domain>/mytool/badges/apt.svg)](https://github.com/<your-org>/mytool/releases/latest)
[![crates.io](https://<packages-domain>/mytool/badges/crates-io.svg)](https://github.com/<your-org>/mytool/releases/latest)
```

Badge files are named after their channel: `macos-dmg.svg`, `homebrew.svg`, `scoop.svg`, `chocolatey.svg`, `winget.svg`, `apt.svg`, `rpm.svg`, `apk.svg`, `aur.svg`, `crates-io.svg`, `npm.svg`, `install-scripts.svg`.

---

## Versioning

Shared workflows are versioned with tags (`@v1`, `@v2`). Breaking changes require a major version bump. Pin to a specific version in your caller workflow.
