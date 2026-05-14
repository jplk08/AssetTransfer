# AssetTransfer

**Asset Transfer Script Library** — Copyright (c) Thinkbox Software Inc.

A Python-based file transfer framework that runs as a **Deadline render farm plugin**. It provides a unified interface for syncing production assets between offices and servers across five different transfer protocols.

> **Status:** Beta software. Do not use in production without thorough testing.

---

## Table of Contents

- [Project Goal](#project-goal)
- [Architecture Overview](#architecture-overview)
- [Supported Protocols](#supported-protocols)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Command-Line Usage](#command-line-usage)
- [Running as a Deadline Plugin](#running-as-a-deadline-plugin)
- [Protocol-Specific Notes](#protocol-specific-notes)

---

## Project Goal

AssetTransfer solves the problem of moving large production asset files (renders, textures, project files) between distributed office locations. Instead of hard-coding a transfer method, it abstracts the protocol layer so studios can swap between FTP, SFTP, rsync, Robocopy, or FileCatalyst without changing their pipeline.

It integrates directly with **Thinkbox Deadline** as a job plugin, meaning asset transfers can be queued, monitored, and managed alongside render jobs on the farm.

---

## Architecture Overview

```
at_client.py           # CLI entry point — parses args, reads config, dispatches to API
AssetTransfer.py       # Deadline plugin wrapper — bridges Deadline job system to at_client.py
api_objects/
  AbstractAPI.py       # Base class defining download_files() / upload_files() interface
  FTPAPI.py            # FTP implementation
  SFTPAPI.py           # SFTP implementation (uses bundled PyCrypto libs)
  rsyncAPI.py          # rsync implementation (requires private key)
  RobocopyAPI.py       # Robocopy implementation (Windows only)
  FileCatalystAPI.py   # FileCatalyst implementation (requires licensed jar)
  sftp_libs/           # Bundled PyCrypto binaries for Linux SFTP support
  rsync/               # rsync private key directory
```

**Flow:**
1. Deadline submits a job referencing `AssetTransfer.py` with a requirements file and config file as auxiliary files.
2. `AssetTransfer.py` invokes `at_client.py` with the appropriate arguments.
3. `at_client.py` reads the config, selects the correct API class dynamically, and runs the transfer.
4. Progress is reported back to Deadline via stdout parsing (`Transfering: N/M`).

---

## Supported Protocols

| Protocol | Class | Notes |
|---|---|---|
| `ftp` | `FtpAPI` | Standard FTP |
| `sftp` | `SftpAPI` | SSH file transfer; requires PyCrypto |
| `rsync` | `RsyncAPI` | Requires a private key in `api_objects/rsync/` |
| `robocopy` | `RobocopyAPI` | Windows only; uses IPC$ connection |
| `filecatalyst` | `FilecatalystAPI` | Requires licensed FC CLI jar in `api_objects/fc_cli/` |

---

## Project Structure

```
AssetTransfer/
├── AssetTransfer.py       # Deadline plugin entry point
├── at_client.py           # Standalone CLI client
├── job_info.txt           # Deadline job metadata (Plugin=AssetTransfer)
├── plugin_info.txt        # Deadline plugin metadata (Version=2.7)
├── README.md              # This file
├── LICENSE                # Apache License 2.0
└── api_objects/
    ├── __init__.py
    ├── AbstractAPI.py
    ├── FTPAPI.py
    ├── FileCatalystAPI.py
    ├── RobocopyAPI.py
    ├── rsyncAPI.py
    ├── SFTPAPI.py
    ├── rsync/             # Place your rsync private key here
    ├── fc_cli/            # Place FileCatalyst jar here (not included)
    └── sftp_libs/         # Bundled PyCrypto for Linux SFTP
```

---

## Requirements

- **Python 2.7** (as specified in `plugin_info.txt`)
- **Thinkbox Deadline** (for plugin mode)
- Protocol-specific dependencies:
  - **SFTP:** PyCrypto (bundled for Linux under `sftp_libs/`)
  - **FileCatalyst:** Licensed FC CLI `.jar` file
  - **rsync:** Private key file placed in `api_objects/rsync/`
  - **Robocopy:** Windows only

---

## Configuration

Create an INI config file with the following sections:

```ini
[Server]
ip         = 192.168.1.100        # Server IP or hostname
port       = 22                   # Port (21 for FTP/FC/rsync, 22 for SFTP; omit for Robocopy)
server_dir = /data/assets         # Remote directory (full Windows path for Robocopy, e.g. C$\Data\assets)

[User]
name = myuser                     # Transfer protocol username
pass = mypassword                 # Transfer protocol password

[Client]
log        = transfer.log         # Log filename (written to ./log/)
service    = sftp                 # Protocol: ftp | filecatalyst | robocopy | rsync | sftp
target_dir = ./downloads          # Where downloaded files are saved (full path for rsync)
source_dir = /local/assets        # Where uploaded files are read from
flags      =                      # Optional: --upload and/or --overwrite (space-separated)
```

---

## Command-Line Usage

```bash
python at_client.py --file=[path] --config=[path] [options] [flags]
```

**Required:**

| Parameter | Description |
|---|---|
| `--file=[path]` | Path to a text file listing the files to transfer (one per line) |
| `--config=[path]` | Path to the INI configuration file |

**Optional (override config file values):**

| Parameter | Description |
|---|---|
| `--ip=[ip]` | Server IP or hostname |
| `--port=[port]` | Port number |
| `--service=[name]` | Transfer protocol: `ftp`, `filecatalyst`, `robocopy`, `rsync`, `sftp` |

**Flags:**

| Flag | Description |
|---|---|
| `--upload` | Upload files (default is download) |
| `--overwrite` | Transfer files even if identical local copies already exist |

**Example:**

```bash
python at_client.py --file=files_to_sync.txt --config=studio_config.ini --service=sftp --upload
```

---

## Running as a Deadline Plugin

1. Place the `AssetTransfer/` directory in your Deadline repository plugins folder.
2. Submit a job with:
   - **Plugin:** `AssetTransfer`
   - **Python Version:** `2.7`
   - **Auxiliary files:** `[0]` requirements file, `[1]` config file (order matters)
3. Deadline will invoke `at_client.py` automatically and track progress via stdout.

Progress output format parsed by Deadline:
```
Transfering: 3/10
```

---

## Protocol-Specific Notes

### FileCatalyst
Place the licensed FileCatalyst CLI jar at:
```
api_objects/fc_cli/
```

### rsync
Place your private key at:
```
api_objects/rsync/
```

### Robocopy
- Windows only.
- `server_dir` in the config must be a full Windows-style UNC path (e.g. `C$\Data\robodata\new`).
- The script automatically sets up and tears down an IPC$ connection using `net use`.

### SFTP
PyCrypto binaries for Linux are bundled under `api_objects/sftp_libs/linux/`. On other platforms, install PyCrypto separately.
