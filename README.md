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
- [Per-Project Adoption](#per-project-adoption)
- [Secrets Reference](#secrets-reference)
- [Variables Reference](#variables-reference)
- [GPG Key Rotation](#gpg-key-rotation)
- [User Install Instructions](#user-install-instructions)
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
2. Note the **account name** (e.g., `my-signing-account`). This becomes `AZURE_CODE_SIGNING_ACCOUNT`.

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
| `AZURE_CODE_SIGNING_ACCOUNT` | Trusted Signing account name |
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
   The clipboard contents become `APPLE_CERT_BASE64`. The password you chose becomes `APPLE_CERT_PASSWORD`.

#### Get your Team ID

1. Log in to [developer.apple.com](https://developer.apple.com/account).
2. Go to **Membership details**.
3. Copy the **Team ID** (10-character alphanumeric string) — this is `APPLE_TEAM_ID`.

#### Create an app-specific password for notarization

1. Go to [appleid.apple.com](https://appleid.apple.com) > **Sign-In and Security** > **App-Specific Passwords**.
2. Generate a new password (label it e.g., `GitHub Actions Notarization`).
3. Copy it — this is `APPLE_ID_PASSWORD`.

> **Important:** Do not use your actual Apple ID password. `notarytool` requires an app-specific password.

#### Secrets to configure

| Secret | Value |
|--------|-------|
| `APPLE_CERT_BASE64` | Base64-encoded .p12 certificate + private key |
| `APPLE_CERT_PASSWORD` | Password used when exporting the .p12 |
| `APPLE_TEAM_ID` | 10-character Team ID from developer.apple.com |
| `APPLE_ID` | Apple ID email address enrolled in the Developer Program |
| `APPLE_ID_PASSWORD` | App-specific password (not your Apple ID password) |

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

Chocolatey and winget both require a signed MSI installer. The `package-msi.yml` workflow builds this using `cargo-wix`, which requires a `dist/wix/main.wxs` configuration file in your project.

---

### 8. winget

**Required for:** `publish-winget.yml`
**Cost:** Free

1. You need a GitHub account (used to fork and submit PRs to `microsoft/winget-pkgs`).
2. Go to GitHub > **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)**.
3. Create a token with `public_repo` scope (required to fork and PR to the public `winget-pkgs` repo).
4. Copy the token — this is `WINGET_PAT`.

The workflow uses `wingetcreate` to automatically generate a manifest and submit a PR to `microsoft/winget-pkgs`. The PR goes through automated validation before merging.

Like Chocolatey, winget requires a signed MSI installer (`dist/wix/main.wxs`).

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
      bundle-id: com.example.myproject # macOS bundle ID (if signing macOS)
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
└── wix/                   # Required for MSI (Chocolatey/winget)
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
| `AZURE_CODE_SIGNING_ACCOUNT` | Windows targets | Azure Portal > Trusted Signing account name |
| `AZURE_CERT_PROFILE` | Windows targets | Certificate profile name in Trusted Signing |
| `APPLE_CERT_BASE64` | macOS targets | Exported .p12 certificate, base64-encoded |
| `APPLE_CERT_PASSWORD` | macOS targets | Password for the .p12 export |
| `APPLE_TEAM_ID` | macOS targets | developer.apple.com > Membership |
| `APPLE_ID` | macOS targets | Apple ID email address |
| `APPLE_ID_PASSWORD` | macOS targets | App-specific password from appleid.apple.com |
| `CARGO_REGISTRY_TOKEN` | `project-type: rust` | crates.io > Account > API Tokens |
| `NPM_TOKEN` | `project-type: node` | npmjs.com > Account > Access Tokens |
| `HOMEBREW_TAP_TOKEN` | `enable-homebrew` | GitHub fine-grained PAT (Contents: RW on tap repo) |
| `SCOOP_BUCKET_TOKEN` | `enable-scoop` | GitHub fine-grained PAT (Contents: RW on bucket repo) |
| `CHOCO_API_KEY` | `enable-chocolatey` | community.chocolatey.org > Account > API Keys |
| `WINGET_PAT` | `enable-winget` | GitHub classic PAT with `public_repo` scope |
| `PACKAGES_REPO_TOKEN` | `enable-apt` or `enable-rpm` | GitHub fine-grained PAT (Contents: RW, Actions: Write on packages repo) |
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
| `PACKAGES_DOMAIN` | Domain for self-hosted apt/rpm repos | `packages.example.com` | packages repo workflow |
| `INSTALL_DOMAIN` | Domain for installer script hosting | `install.example.com` | Installer templates |
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

Replace `<packages-domain>`, `<install-domain>`, `<your-org>`, and `<project>` with your configured values.

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
