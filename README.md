# The Summoner and Wraith

**A hands-on cybersecurity training platform for blue-team analysts.**

Summoner deploys a configurable simulated implant (Wraith) on your machine and challenges analysts to find it using real tools — the same way they would work an actual incident. Everything is self-contained, leaves no real malware behind, and cleans up completely when you're done.

---

> **Run this in an analyst VM only.**
> Summoner drops and executes a binary, makes real outbound network connections, and can install persistence mechanisms — all intentionally. Run it on an isolated lab machine or snapshot-backed VM. Never on a production host or a machine connected to a corporate network without explicit authorisation.
>
> **Disable AV/EDR before launching a scenario** unless you're specifically testing detection. Wraith is designed to behave like malware — security tooling may quarantine the binary or block network activity before the analyst gets a chance to find it themselves.
>
> **Recommended analyst VMs:**
> - [REMnux](https://github.com/REMnux/remnux-distro) — the gold standard Linux malware analysis distro
> - [FLARE VM](https://github.com/mandiant/flare-vm) — Mandiant's Windows-based analysis environment
> - [BattleStation 26](https://github.com/DFIRmadness/battlestation26) — community Ubuntu 24.04 forensics workstation

---

<img width="1038" height="685" alt="Screenshot_2026-06-12_16-35-00" src="https://github.com/user-attachments/assets/0de0406b-314c-4d0c-8944-e4ee7859ae3c" />



## Training Benefits

### Process Hunting

Analysts practice finding a malicious process in a live process list — the exact skill needed on day one of an incident response engagement. Wraith can be deployed under a random gibberish name (obviously fake, for beginners), a lookalike name (`svch0st.exe`, `lsas.exe`, `conpanels.exe` — requires a careful eye), or a custom name.

Hard-mode scenarios use PPID spoofing to hide the process's parent, forcing analysts to go beyond simple name-matching and correlate process tree relationships the way experienced responders do.

### Network Traffic Analysis

Wraith beacons to a C2 host on configurable protocols at difficulty-scaled intervals. Easy mode beacons continuously every 1–3 seconds so traffic is always visible in a quick `netstat` or short Wireshark capture. Hard mode uses long jittered intervals over encrypted HTTPS on port 443, blending in with normal web traffic and requiring patient, deliberate observation.

Analysts practice:
- Reading `netstat -ano` / `ss -tnp` output and correlating connections to PIDs
- Filtering Wireshark for suspicious DNS queries, outbound connections, and HTTP headers
- Identifying the C2 domain, IP, and port from live traffic
- Finding the per-scenario implant ID hidden in the `X-Session-Token` HTTP header

### Binary Reverse Engineering

Wraith carries a per-scenario campaign tag string embedded at deploy time. The encoding is scaled to difficulty — easy mode is straightforward, hard mode requires more than basic tooling to recover it.

The RE Analysis panel in the UI asks analysts to submit their findings: file hash, beacon mode, beacon interval, network protocols, persistence mechanism, campaign tag, and implant ID. Each field is validated independently, so analysts get per-field feedback.

### Digital Forensics Fundamentals

Analysts learn to compute and verify SHA256 hashes, identify suspicious file drop locations by difficulty level, correlate process metadata (PID, name, path, parent) with network telemetry, and enumerate persistence mechanisms. This mirrors the workflow used in real incident response.

### Progressive Hints

No answer is simply handed over. A per-field hint system reveals one more character or path component per click:

- **PID / Port** — one digit at a time
- **Process name** — first character → first three characters → full name
- **C2 IP** — one octet at a time
- **C2 domain** — one label at a time
- **File path** — one directory component at a time

A sequential hint ladder (5 levels) provides broader methodology guidance without revealing the specific answer, and automatically skips hints that cover fields the analyst has already confirmed.

---

## Tool Testing

Summoner is also an effective test harness for defensive tooling:

| Test | What to check |
|---|---|
| **EDR / AV detection** | Does your agent flag Wraith? Does it catch a lookalike name? What about with PPID spoofing? |
| **SIEM alerting** | Does your SIEM fire on suspicious DNS queries, the unusual outbound port, or the `X-Session-Token` header? |
| **Network monitoring** | Do your IDS/IPS rules catch the C2 traffic? Does Zeek log the domain? |
| **Process auditing** | Does your process monitor catch orphaned processes or suspicious parent-child relationships? |
| **Hash-based detection** | Does your tooling alert on the dropped binary's SHA256? |
| **Persistence detection** | Do your integrity monitors catch the persistence mechanism Summoner installed? |

Because every aspect of Wraith's behaviour is configurable, you can tune exactly which signals it produces and verify whether each one triggers a detection.

---

## Architecture

```
Summoner (instructor binary)
├── Web UI  — localhost browser interface for scenario management
├── Embedded Wraith binaries — both Windows and Linux baked in at compile time
└── Scenario engine
    ├── Generates randomised parameters per difficulty
    ├── Patches per-scenario campaign tag into Wraith binary (encoding scaled to difficulty)
    ├── Assigns a unique implant ID (wraithXXXXXXX)
    ├── Launches Wraith as a detached/orphan/PPID-spoofed process
    ├── Writes a signed answer-key manifest (~/.summoner/)
    └── Cleans up all artifacts on demand or automatically on Ctrl+C

Wraith (target implant — extracted and run by Summoner)
├── Reads and self-deletes config file on startup
├── Beacons to C2 on configured protocols every cycle:
│   ├── DNS  — resolves C2 domain
│   ├── HTTP — GET with campaign tag or random UA; X-Session-Token: wraithXXXXXXX
│   ├── HTTPS — TLS GET with same headers
│   └── TCP  — raw connection to C2 host:port
└── Exits cleanly on SIGINT / SIGTERM
```

Summoner and Wraith are distributed as **two single-file binaries** — one for Windows, one for Linux. Both Wraith payloads are embedded inside Summoner at compile time. Instructors distribute one file.

---


## Difficulty Levels

| | Easy | Medium | Hard |
|---|---|---|---|
| **Process name** | Random gibberish | Lookalike (`svch0st.exe`) | Lookalike |
| **C2 port** | Random suspicious (4444, 1337, 8080…) | 80 or 443 | 443 only |
| **Beacon mode** | Continuous (~1–3s) | Jitter (~45s ± 50%) | Jitter (~2min ± 50%) |
| **Network protocols** | DNS + HTTP + TCP | DNS + HTTP | DNS + HTTPS + TCP |
| **User-Agent** | Campaign tag string | Campaign tag string | Random browser UA |
| **Campaign tag encoding** | Easy | Moderate | Challenging |
| **Parent mode** | Orphan | Orphan | PPID spoof |
| **Drop path** | Home / Desktop | Temp directory | System-adjacent directory |

Every parameter can be overridden individually.

---

## Running Summoner

### Web UI (recommended)

```bash
./summoner_linux_amd64        # Linux
summoner_windows_amd64.exe    # Windows
```

Summoner starts on port 8080 (auto-increments if busy) and opens your browser. Run as administrator/root to unlock memory deployment, system-level persistence, and PPID spoofing.

```bash
./summoner_linux_amd64 --port 9090   # custom port
```

Press `Ctrl+C` to shut down. All active scenarios are cleaned up automatically before exit.

### Web UI Tabs

**Configure & Launch**
Select difficulty, expand Advanced Options to override any parameter, toggle persistence, and launch. The Previous Launches list tracks active scenarios and lets you clean them up without switching tabs.

**Active Scenarios**
Live view of all running and stopped scenarios. Jump to Validate or Cleanup from each card.

**Validate Your Findings**
Enter your answers field by field. Each field locks green when correct. Use the per-field hint buttons for progressive disclosure or Request Hint for broader methodology guidance. Reveal All shows the full answer key.

**Cleanup**
Remove individual scenarios or wipe everything. Reveal All on each card shows answers to assist with cleanup.

**RE Analysis**
Answer reverse-engineering questions about the dropped binary: file hash, beacon behaviour, embedded campaign tag, and the implant ID from network traffic.

**Training**
Collapsible reference sections covering tools and techniques for each type of analysis.

### CLI

```bash
# Launch a scenario
./summoner_linux_amd64 run --difficulty medium

# List active scenarios
./summoner_linux_amd64 list

# Check whether Wraith is still alive
./summoner_linux_amd64 status --scenario <id>

# Submit findings
./summoner_linux_amd64 validate --scenario <id>

# Request next hint
./summoner_linux_amd64 hint --scenario <id>

# Clean up
./summoner_linux_amd64 cleanup --scenario <id>
```

`summoner run` flags:

| Flag | Default | Description |
|---|---|---|
| `--difficulty` | `medium` | `easy`, `medium`, or `hard` |
| `--name` | auto | Process name (`.exe` appended on Windows if missing) |
| `--binary-type` | `disguised` | `disguised`, `lookalike`, or `random` |
| `--domain` | `dfirmadness.com` | C2 domain — mutually exclusive with `--ip` |
| `--ip` | — | C2 IP — skips DNS entirely; mutually exclusive with `--domain` |
| `--port` | auto | C2 port |
| `--path` | auto | Binary drop path |
| `--beacon` | auto | `static`, `jitter`, or `continuous` |
| `--deploy` | `disk` | `disk` or `memory` (Linux only) |
| `--parent-mode` | auto | `orphan`, `spoof`, or `random` |
| `--parent-target` | — | Process to spoof as parent (e.g. `explorer.exe`) |
| `--persistence` | off | Enable persistence |
| `--mechanism` | auto | Persistence mechanism (leave blank for difficulty-appropriate default) |

---

## Network Behavior

Wraith makes **real outbound network requests** to the configured destination. Failed connections still generate observable telemetry in Wireshark, EDR logs, and `netstat` — that's the point.

> Use only domains and IPs you own or that are designated for lab use.
> **Do not use real malware C2 domains.** HTTP requests will reach the server.

When a domain is configured, Summoner resolves it at launch and stores the resulting IP in the answer key so it matches what analysts see in packet captures. When an IP is configured directly, the domain is cleared and no DNS resolution occurs.

---

## Scenario Artifacts

| Artifact | How to find it |
|---|---|
| Process name | `ps`, Task Manager, Process Hacker |
| PID | Same as above |
| Binary path | `/proc/<pid>/exe`, `Get-Process`, Process Hacker |
| SHA256 hash | `sha256sum`, `Get-FileHash`, pestudio |
| C2 domain | Wireshark DNS filter, `netstat`, OS DNS cache |
| C2 IP | Wireshark, `netstat -ano`, `ss -tnp` |
| C2 port | Same as above |
| Campaign tag | Static analysis of the dropped binary; Wireshark User-Agent header |
| Implant ID | Wireshark → follow HTTP stream → `X-Session-Token` header |

---

## Memory Deploy Mode (Linux only)

```bash
./summoner_linux_amd64 run --deploy memory
```

Wraith is loaded into a RAM-backed file descriptor via `memfd_create` — the binary never touches disk. The process appears as `/memfd:wraith (deleted)` in process listings. There is no file for `find` or file integrity monitoring to catch; detection requires `/proc/<pid>/maps`, `/proc/<pid>/exe`, or live EDR telemetry. The file path field is skipped during validation in this mode.

---

---

## Cleanup

Cleanup removes: the Wraith process, the dropped binary, all persistence entries, and the manifest. Summoner always prompts for confirmation before destructive actions.

**Automatic cleanup on exit:** pressing `Ctrl+C` while Summoner is running automatically cleans up every active scenario before exiting. Manifests are stored in `~/.summoner/` and can be used to clean up manually if Summoner exits unexpectedly.

---

## Changelog

### v1.01
- **Fix: PPID spoof to SYSTEM-owned processes (e.g. `svchost.exe`)** — Summoner now enables `SeDebugPrivilege` before opening the target process handle, which is required for processes running as SYSTEM even when Summoner is elevated. Previously these targets always fell back to orphan mode with an "Access is denied" warning.
- **Fix: PPL-protected process fallback** — When spoofing a parent by name (e.g. `svchost.exe`), Summoner now iterates all running instances and skips any that are Protected Process Light (PPL). Previously it would fail if the first matching instance was PPL-protected.

### v1.0
- Initial release.
