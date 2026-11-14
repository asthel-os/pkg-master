# `pkg-master` — AsthelOS Master Repository & Release Specifications

`pkg-master` is the central repository index and release specification tree for **AsthelOS**, consumed by both **`pkg`** (the native package manager) and **`bootstrap`** (the native system & Live ISO builder).

Thanks to the modular `manifest.json` root descriptor, any repository can adopt a versioned layout (like `asthel-os/pkg-master`), a flat `repository.json`, a direct GitHub Releases `packages.json` channel, or an inline repository list.

---

## Directory Structure

```text
pkg-master/
├── manifest.json                 # Modular root descriptor (entry point for pkg & bootstrap)
├── asthel-os/
│   └── packages.json             # GitHub Releases package channel
└── versions/
    └── 26.04/
        ├── repository.json       # Repositories for AsthelOS 26.04 (deb, ppa, git)
        ├── base.toml             # Base system specification (used by bootstrap)
        ├── live.toml             # Live ISO specification (used by bootstrap)
        └── data/                 # System configs & themes copied into the RootFS
```

---

## 1. Root Descriptor (`manifest.json`)

Every repository can place a `manifest.json` at its root to tell `pkg` and `bootstrap` how the repository is organized.

### Supported Fields

| Field | Type | Description |
| :--- | :--- | :--- |
| `name` | `string` | Namespace prefix for the repository (e.g., `"asthel-os"` -> `asthel-os/ubuntu`). |
| `type` (or `layout`) | `string` | Layout mode: `"versioned"`, `"repository"` (`"flat"` / `"master"`), `"packages"` (`"git"`), or `"inline"`. |
| `default_version` | `string` | Fallback version when `PKG_VERSION` or `/etc/os-release` (`ID=asthel-os`) is not present. |
| `versions_dir` | `string` | Directory containing versioned subfolders (defaults to `"versions"`). |
| `repository_file` | `string` | Filename of the repository manifest (defaults to `"repository.json"`). |
| `packages_file` | `string` | Path to `packages.json` when `type` is `"packages"` / `"git"` (defaults to `"packages.json"`). |
| `repos` | `array` | Inline list of repository entries when `type` is `"inline"`. |

### Layout Examples

#### A. Versioned Layout (`"type": "versioned"`) — used by `pkg-master`
Resolves `<versions_dir>/<version>/<repository_file>` using the following priority:
1. `PKG_VERSION` environment variable (set automatically by `bootstrap`).
2. `VERSION_ID` from `<PKG_INSTALL_ROOT>/etc/os-release` or `/etc/os-release` when `ID=asthel-os`.
3. `default_version` in `manifest.json`.
4. Latest version directory inside `<versions_dir>/`.

```json
{
  "name": "asthel-os",
  "type": "versioned",
  "default_version": "26.04",
  "versions_dir": "versions",
  "repository_file": "repository.json"
}
```

#### B. Flat Repository Layout (`"type": "flat"` or `"repository"`)
Loads a single `repository.json` (or custom path) from the root of the repository:

```json
{
  "name": "my-distro",
  "type": "flat",
  "repository_file": "repository.json"
}
```

#### C. Direct GitHub Releases Channel (`"type": "packages"` or `"git"`)
Exposes a `packages.json` file directly without needing a `repository.json`:

```json
{
  "name": "my-apps",
  "type": "packages",
  "packages_file": "packages.json"
}
```

#### D. Inline Layout (`"type": "inline"`)
Defines all repositories directly inside `manifest.json`:

```json
{
  "name": "custom",
  "type": "inline",
  "repos": [
    {
      "name": "extras",
      "type": "deb",
      "url": "http://archive.ubuntu.com/ubuntu",
      "suite": "resolute",
      "components": ["main", "universe"]
    }
  ]
}
```

---

## 2. Repository Manifest (`repository.json`)

A `repository.json` file defines the `name` namespace and the list of `repos` (`deb`, `ppa`, or `git`):

```json
{
  "name": "asthel-os",
  "repos": [
    {
      "name": "ubuntu",
      "type": "deb",
      "url": "http://archive.ubuntu.com/ubuntu",
      "suite": "resolute",
      "components": ["main", "restricted", "universe", "multiverse"]
    },
    {
      "name": "asthel-os",
      "type": "git",
      "path": "asthel-os/packages.json"
    },
    {
      "name": "my-tool",
      "type": "git",
      "repo": "owner/my-tool"
    },
    {
      "name": "drivers",
      "type": "ppa",
      "suite": "resolute",
      "repo": "ppa:graphics-drivers/ppa"
    }
  ]
}
```

### Repository Entry Types

- **`deb`**: Standard APT/DEB mirror. Requires `url`, `suite`, and `components`.
- **`ppa`**: Launchpad PPA (`ppa:<owner>/<name>`). Uses `repo` and optional `suite` (defaults to the system codename).
- **`git`**: GitHub Releases source.
  - With `path`: points to a `packages.json` channel file inside the same repository (e.g., `asthel-os/packages.json`).
  - With `repo`: points directly to a single GitHub repository (`owner/repo`) whose latest Release provides a `.deb` (or `.tar.gz` / `.tgz` / `.tar.xz`) asset.

---

## 3. GitHub Releases Channel (`packages.json`)

Used when a `git` repository entry specifies `"path"` (such as `asthel-os/packages.json`):

```json
{
  "packages": [
    {
      "name": "pkg",
      "repo": "asthel-os/pkg"
    }
  ]
}
```

During `pkg update`, `pkg` queries the latest GitHub Release of each listed `repo` and indexes matching `.deb` assets for the target architecture (`amd64`, `arm64`, `riscv64`, `all`).

---

## 4. Release Specifications (`versions/<version>/`)

Each release folder under `versions/<version>/` also provides specifications for `bootstrap`:

- **`base.toml`**: Declares release metadata (`name`, `version`, `codename`, `id`, `base_suite`), core components (`bootloader`, `initramfs`, `init`, `package_manager`, `kernel`), `essential` and `boot` package lists, and `[configs]` mappings from `data/`.
- **`live.toml`**: Inherits `base.toml` and configures Live ISO settings (`iso_label`, `iso_filename`, `hostname`, `username`, `autologin`, `squashfs_compression`), `live` packages, and systemd `[services]`.
- **`data/`**: Static configuration files, Plymouth themes, and GRUB themes copied into the target RootFS.

---

## 5. Testing & Using Your Repository

```bash
# Add a repository from GitHub
sudo pkg add github owner/my-repo

# Or test a local repository directory before pushing
sudo pkg add github /path/to/my-repo

# Sync indexes and verify
sudo pkg update
pkg repo list
pkg search <package-name>

# Inspect release specifications with bootstrap
bootstrap inspect -r /path/to/my-repo
```
