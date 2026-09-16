# Camellia CLI

A lightweight **volunteer / distributed computing client**, protocol-compatible with the [BOINC](https://boinc.berkeley.edu/) scheduler ecosystem. Camellia attaches to BOINC-style projects, downloads work units, runs the science applications on your CPU and GPU, and reports the results back to the project server.

Coded by **Alperen Yavuz**.

---

## Highlights

- **BOINC-compatible scheduler protocol** — talks to existing project servers (`cgi-bin/scheduler`, file download/upload handlers).
- **Separate CPU / GPU cache management** — independent slot and project directories, so GPUs and CPUs never compete for the same disk space.
- **Real work loop** — downloads inputs, runs the science executable in an isolated slot, tracks progress, checkpoints, and reports results.
- **GPU/CPU hardware detection** — automatic CPU, GPU, RAM and disk discovery on Windows, Linux and macOS.
- **GUI RPC server** — a BOINC-style management interface so BOINC Manager (or your own tooling) can connect.
- **Cross-platform** — pure Go, no runtime dependencies.

---

## Table of Contents

1. [How it works](#how-it-works)
2. [Requirements](#requirements)
3. [Building](#building)
4. [Usage](#usage)
5. [Configuration](#configuration)
6. [GUI RPC interface](#gui-rpc-interface)
7. [Data directory](#data-directory)
8. [Project layout](#project-layout)
9. [Work flow](#work-flow)
10. [Environment variables](#environment-variables)

---

## How it works

Camellia follows the classic volunteer-computing cycle:

1. **Attach** to a project (account + authenticator).
2. The **scheduler engine** periodically queries the project server, reports finished results, and requests new work units.
3. The **worker engine** assigns work units to disposable *slots*, downloads the input files, launches the science application, tracks progress, and checkpoints periodically.
4. Finished results are **uploaded and reported**; credits are tracked on a daily basis.
5. Everything (projects, results, transfers, statistics) is persisted to `client_state.xml`.

---

## Requirements

- **Go 1.26** or later (`go.mod` declares `go 1.26.7`).
- A BOINC-compatible project URL with an **authenticator** (for attaching).

No third-party Go modules are used — the codebase builds entirely on the standard library.

---

## Building

```sh
# Build into ./bin
go build -o bin/camellia ./cmd/camellia

# Or install into your GOPATH/bin
go install ./cmd/camellia

# Verify everything compiles and vets clean
go build ./...
go vet ./...
```

---

## Usage

```
Usage:
  camellia              Run daemon (foreground)
  camellia --status     Show client status
  camellia --stop       Stop running client
  camellia --version    Show version
  camellia --help       Show this help
```

### Start the daemon

```sh
camellia
```

On startup Camellia prints a summary of your detected hardware and begins
contacting attached projects for work.

### Check status

```sh
camellia --status
```

Shows the daemon state: attached projects and downloaded/completed tasks.

### Stop the daemon

```sh
camellia --stop
```

Politely asks the running daemon to shut down. Running science apps are
terminated and checkpoints are written so work can resume on the next start.

> **Note:** Camellia listens on port **31418**. Do not confuse it with real
> BOINC clients, which use port **31416**.

---

## Configuration

On first run Camellia creates `cc_config.xml` in the [data directory](#data-directory).
Constants and defaults live in `internal/config/config.go`.

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<cc_config>
  <options>
    <user_agent>Camellia/1.0.0</user_agent>
    <allow_remote_gui_rpc/>
    <report_results_early/>
    <use_all_gpus/>
    <max_app_clients>64</max_app_clients>
    <http_transfer_timeout>30</http_transfer_timeout>
    <http_servers_busy_timeout>30</http_servers_busy_timeout>
    <dont_contact_ref_site/>
  </options>
  <log_flags/>
  <gpu_cache>
    <enabled>1</enabled>
    <cache_size_mb>2048</cache_size_mb>
    <cpu_cache_size_mb>4096</cpu_cache_size_mb>
    <separate_slots>1</separate_slots>
    <separate_projects>1</separate_projects>
  </gpu_cache>
</cc_config>
```

### Options

| Setting                      | Description                                                        |
| ---------------------------- | ------------------------------------------------------------------ |
| `user_agent`                 | User-agent reported to project servers                              |
| `allow_remote_gui_rpc`       | Listen on `0.0.0.0` instead of `127.0.0.1`                          |
| `max_app_clients`            | Maximum concurrent science applications                             |
| `report_results_early`       | Report results as soon as they finish                               |
| `http_transfer_timeout`      | HTTP timeout (seconds) for file transfers                           |
| `http_servers_busy_timeout`  | Back-off timeout (seconds) when a server reports busy               |
| `dont_contact_ref_site`      | Never contact the reference (home) site                             |
| `use_all_gpus`               | Use every detected GPU                                              |

### GPU cache

| Setting                | Description                                              |
| ---------------------- | -------------------------------------------------------- |
| `enabled`              | Enable separate GPU cache management                      |
| `cache_size_mb`        | GPU cache size (MB)                                       |
| `cpu_cache_size_mb`    | CPU cache size (MB)                                       |
| `separate_slots`       | Use `slots_gpu/` instead of a shared `slots/` for GPUs    |
| `separate_projects`    | Use `projects_gpu/` instead of a shared `projects/`       |

---

## GUI RPC interface

Camellia exposes a BOINC-style line-based TCP protocol on **`127.0.0.1:31418`**
(`0.0.0.0:31418` with `allow_remote_gui_rpc`). Each request is a single XML
document terminated by an ETX (`0x03`) byte.

Supported requests:

| Request                      | Description                                  |
| ---------------------------- | --------------------------------------------- |
| `auth1` / `auth2`            | Nonce-based authentication (`gui_rpc_auth.cfg`) |
| `exchange_versions`          | Version handshake                              |
| `get_state`                  | Full client state XML                          |
| `get_cc_status`              | Run / network mode, disk usage                 |
| `get_messages`               | Client messages                                |
| `get_file_transfers`         | Active file transfers                          |
| `get_statistics`             | Daily CPU/GPU statistics                       |
| `get_disk_usage`             | Per-project disk usage                         |
| `get_daily_xfer_history`     | Daily transfer history                         |
| `get_global_prefs_override`  | Preference overrides                           |
| `set_global_prefs_override`  | Set preference overrides                       |
| `set_run_mode`               | `always` / `auto` / `never`                    |
| `set_network_mode`           | `always` / `auto` / `never`                    |
| `result_op`                  | `abort` / `suspend` / `resume` a task          |
| `project_op`                 | `detach` / `suspend` / `resume` / `update`     |
| `file_transfer_op`           | File transfer operations                       |
| `project_attach`             | Attach to a project (URL + authenticator)      |
| `run_benchmarks`             | Trigger benchmarks                             |
| `get_host_info`              | Host hardware information                      |
| `quit`                       | Shut the daemon down                           |

The same wire format used by `camellia --stop` (`<quit/>`) also drives the
daemon shutdown path.

---

## Data directory

Location depends on the platform:

| OS | Path                                                     |
| --- | -------------------------------------------------------- |
| Windows | `%ProgramData%\Camellia\`                       |
| macOS   | `~/Library/Application Support/Camellia/`      |
| Linux   | `~/.local/share/camellia/`                      |

Contents:

| Path                | Purpose                                        |
| ------------------- | ---------------------------------------------- |
| `client_state.xml`  | Persisted state (projects, results, stats)     |
| `cc_config.xml`     | Client configuration                           |
| `gui_rpc_auth.cfg`  | GUI RPC password                                |
| `projects/`         | CPU project directories (one per project)       |
| `projects_gpu/`     | GPU project directories                        |
| `slots/`            | CPU work-unit slots                            |
| `slots_gpu/`        | GPU work-unit slots                            |
| `cache/` & `cache_gpu/` | Cache staging areas                       |
| `templates/`        | Job templates                                   |
| `notices/`          | Project notices                                 |

---

## Project layout

```
cmd/camellia/            Entry point; daemon wiring, CLI commands, adapters
internal/cache/          CPU/GPU slot allocation and disk usage tracking
internal/config/         cc_config.xml loading, defaults, data-dir resolution
internal/detect/         Hardware detection (CPU, GPU, RAM, disk, host CPID)
internal/guirpc/         GUI RPC server + request dispatch
internal/project/        Project directory setup, account.xml, cc_config generation
internal/scheduler/      BOINC scheduler client + scheduling engine (RPC, work fetch, reporting)
internal/state/          client_state.xml model and persistence
internal/worker/         Task execution loop (download, run, checkpoint, upload)
internal/xml/            XML reply helpers
```

---

## Work flow

1. **Scheduler engine** sends a scheduler request including host info and any
   finished results, then processes the reply:
   - new work is stored as `state=NEW` results, with `GPU` detection from the
     plan class / command line,
   - confirmed reported results are removed after acknowledgement.
2. **Worker engine** walks results and, for each new task:
   - allocates a slot (`slots/` or `slots_gpu/`),
   - downloads input files (direct URL or project file handler),
   - locates the science executable (`app.exe`/`main.exe` on Windows,
     `app`/`main` elsewhere),
   - runs it in the slot with `stdout.txt` / `stderr.txt` capture,
   - checkpoints progress into the slot,
   - on success/error marks the result `READY` / `ERROR` and uploads outputs.
3. Checkpoints (`fraction_done.txt`) let interrupted tasks **resume** after a
   restart.

---

## Environment variables

| Variable                   | Description                                |
| -------------------------- | ------------------------------------------ |
| `CAMELLIA_GUI_RPC_PORT`    | Override the GUI RPC port (default `31418`) |

---

## License

This project is open source. Contributions, bug reports and feature requests
are welcome via [GitHub Issues](../../issues).