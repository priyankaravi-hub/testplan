# Debug Infrastructure (Show Tech-Support) — Functional Specification

**Document:** FS-TS-001
**Version:** 0.1 (Draft)
**Date:** April 2026
**Platform:** UpscaleAI SuperSONiC / Vega8
**Author:** Venki Pallipadi ([vpallipadi@upscaleai.com](mailto:vpallipadi@upscaleai.com))
**Jira PRD:** [ASV2-2252](https://bugatti-asic.atlassian.net/browse/ASV2-2252)

---

## Revision History


| Version | Date       | Author          | Changes                                                                   |
| ------- | ---------- | --------------- | ------------------------------------------------------------------------- |
| 0.1     | 2026-04-03 | Venki Pallipadi | Initial draft from Jira PRD requirements for logging infra / techsupport  |


---

## Table of Contents

1. [Why Debug Infrastructure Matters](#1-why-debug-infrastructure-matters)
2. [Scope & Applicability](#2-scope--applicability)
3. [PRD Requirements Traceability](#3-prd-requirements-traceability)
4. [Glossary](#4-glossary)
5. [Architecture](#5-architecture)
   - [5.1 Ondemand Tech-support generation](#51-ondemand-tech-support-generation)
   - [5.2 Scheduled Tech-support generation](#52-scheduled-tech-support-generation)
   - [5.3 Log Anonymization](#53-log-anonymization)
6. [Tech-Support Bundle Content](#6-tech-support-bundle-content)
   - [6.1 TS-REQ-01: Bundle Generation](#61-ts-req-01-bundle-generation)
   - [6.2 Bundle Content Categories](#62-bundle-content-categories)
7. [Since-Timestamp Mode](#7-since-timestamp-mode)
   - [7.1 TS-REQ-02: Since-Timestamp Filtering](#71-ts-req-02-since-timestamp-filtering)
   - [7.2 Timestamp Format](#72-timestamp-format)
   - [7.3 Filtering Implementation](#73-filtering-implementation)
   - [7.4 Bundle content and since applicability](#74-bundle-content-and-since-applicability)
8. [Rate Limiting](#8-rate-limiting)
   - [8.1 TS-REQ-03: Tech-Support Rate Limit](#81-ts-req-03-tech-support-rate-limit)
   - [8.2 Rate Limit Parameters](#82-rate-limit-parameters)
9. [Scheduled Tech-Support](#9-scheduled-tech-support)
   - [9.1 TS-REQ-04: Scheduled Tech-Support](#91-ts-req-04-scheduled-tech-support)
   - [9.2 Scheduled Tech-support Parameters](#92-scheduled-tech-support-parameters)
10. [Tech-Support Bundle Anonymization](#10-tech-support-bundle-anonymization)
    - [10.1 TS-REQ-05: Tech-Support Bundle Anonymization](#101-ts-req-05-tech-support-bundle-anonymization)
    - [10.2 Tech-support generation Parameter](#102-tech-support-generation-parameter)
    - [10.3 Tech-support post-process Command](#103-tech-support-post-process-command)
11. [Tech-support show CLI](#11-tech-support-show-cli)
    - [11.1 show techsupport](#111-show-techsupport)
    - [11.2 show with since option](#112-show-with-since-option)
    - [11.3 show and rate limit](#113-show-and-rate-limit)
    - [11.4 show schedule](#114-show-schedule)
    - [11.5 show with scramble](#115-show-with-scramble)
    - [11.6 List techsupport bundles](#116-list-techsupport-bundles)
    - [11.7 Delete a techsupport bundle](#117-delete-a-techsupport-bundle)
    - [11.8 Show command syntax](#118-show-command-syntax)
12. [Tech-support Config CLI](#12-tech-support-config-cli)
    - [12.1 config ratelimit](#121-config-ratelimit)
    - [12.2 config schedule](#122-config-schedule)
13. [SONiC Integration](#13-sonic-integration)
    - [13.1 Component Changes](#131-component-changes)
    - [13.2 CONFIG_DB Schema](#132-config_db-schema)
    - [13.3 STATE_DB Schema](#133-state_db-schema)
    - [13.4 SAI API](#134-sai-api)
    - [13.5 Warm Reboot Considerations](#135-warm-reboot-considerations)
14. [Operational Commands & Monitoring](#14-operational-commands--monitoring)
    - [14.1 DB dump](#141-db-dump)
    - [14.2 Syslog Messages](#142-syslog-messages)
15. [Error Handling & Troubleshooting](#15-error-handling--troubleshooting)
    - [15.1 Error Conditions](#151-error-conditions)
    - [15.2 Troubleshooting Guide](#152-troubleshooting-guide)
16. [Interactions & Dependencies](#16-interactions--dependencies)
    - [16.1 Feature Dependencies](#161-feature-dependencies)
17. [Scale & Performance](#17-scale--performance)
18. [Engineering Tasks](#18-engineering-tasks)
19. [Testing Considerations](#19-testing-considerations)
    - [19.1 Unit Tests](#191-unit-tests)
    - [19.2 Integration Tests](#192-integration-tests)
    - [19.3 Scale Tests](#193-scale-tests)
    - [19.4 Negative Tests](#194-negative-tests)
20. [Open Issues & TODOs](#20-open-issues--todos)
21. [References](#21-references)

---

## 1. Why Debug Infrastructure Matters

AI data center fabrics fail in ways that are hard to reproduce. A GPU job stalls at 40% throughput. PFC storms cascade across a rack. A SONiC daemon crashes and restores in under a second, leaving no obvious trace. In each of these cases, the first thing a network engineer reaches for is `show tech-support`.

The problem: standard `show tech-support` on SONiC collects **everything** — all rotated syslogs (up to the 5,000-file retention limit), all SWSS recordings, all SAI traces. On a production Vega8 switch where the `/var/log` partition is ~3.9 GiB, this generates 100-300 MB tarballs and takes **5-15 minutes** to complete. During that window, the switch is consuming significant CPU and disk I/O on a process that may be running because the switch is already under stress.

Many operational pain points emerge at scale:

1. **Time window mismatch.** An incident happens at 14:32. By the time the operator runs `show tech-support` at 14:45, the bundle includes 72 hours of logs, burying the relevant 10-minute window in gigabytes of noise. Operators waste an hour filtering with `grep` before they find the relevant syslog lines.
2. **Runaway bundle generation.** On a flapping link or during an automated health-check loop, `show tech-support` may be triggered dozens of times in minutes — each invocation consuming CPU for log compression, disk I/O for the tarball write, and disk space for the output. A 200 MB bundle × 20 invocations = 4 GB, filling `/var/dump` and `/var/log` simultaneously.
3. **Relevant logs rotated out.** Some of the information collected in techsupport comes from buffers that may rollover and delayed tech-support collection will mean the relevant logs are not available for debug.

This FS specifies three targeted improvements:

- A `since <timestamp>` parameter that scopes log collection to the relevant time window.
- A configurable rate limiter that caps bundle generation to a safe frequency.
- Mini-Dump capability to collect a selective subset of tech-support bundle
- Option to schedule tech-support bundles, enabling programmed collection from central controller or AI processing pipelines.
- Option to scramble/anonymize the logs to ensure information like IP addresses are not logged as is.
- The baseline bundle generation behavior formalized as a specified feature (not just inherited from upstream SONiC).

These changes make tech-support a precision tool instead of a blunt instrument — critical for debugging at the scale of a 1,000-switch AI fabric.

---

## 2. Scope & Applicability

### 2.1 In-Scope


| Area                                | Description                                                                      |
| ----------------------------------- | -------------------------------------------------------------------------------- |
| Tech-support Mini bundle generation | Invocation of `generate_dump.py`, content collected, output path and naming      |
| Since-timestamp filtering           | `--since <timestamp>` parameter scoping log collection to a time window          |
| Rate limiting                       | Configurable maximum bundle generation frequency (min interval)                  |
| Mini-dump capability                | `--type mini` parameter to collect a selective subset                            |
| Automated / scheduled tech-support  | Ability to schedule tech-support collection                                      |
| Tech-support bundle anonymizing     | `--scramble` parameter to anonymize the bundle at source                         |
| List / Delete Bundles               | Options to list and delete techsupport bundles                                   |
| CLI commands                        | All `show techsupport` and `config techsupport` commands                         |
| Configuration persistence           | Rate limit, schedule configs persists across reboots/upgrades via config_db.json |
| Tech-support bundle persistence     | Collected bundles and scrambling tables persists across reboots/upgrades         |


### 2.2 Out-of-Scope


| Area                                      | Reason                                           | When |
| ----------------------------------------- | ------------------------------------------------ | ---- |
| Remote upload (SCP/SFTP push) of bundles  | Separate feature; requires credential management | V2   |
| Log rotation policy configuration via CLI | Separate feature (logrotate config)              | V2   |
| Retention / auto-deletion of old bundles  | Separate feature                                 | V2   |
| Per-container log filtering               | Out of scope for V1 since-timestamp              | V2   |


### 2.3 Target Hardware

- UpscaleAI Vega8 (SkyForce 8000 / 9000), Spectrum-4 ASIC
- `/var/log` partition: ~3.9 GiB
- `/var/dump` output directory: shared with kernel core dumps

---

## 3. PRD Requirements Traceability


| Jira Key                                                         | Requirement                         | FS Section                                                                                                                                                                 |
| ---------------------------------------------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [ASV2-2253](https://bugatti-asic.atlassian.net/browse/ASV2-2253) | Tech-Support Mini Bundle Generation | [Section 6 (Bundle Content)](#6-tech-support-bundle-content), [Section 11.1 (show techsupport)](#111-show-techsupport)                                                     |
| [ASV2-2254](https://bugatti-asic.atlassian.net/browse/ASV2-2254) | Tech-support since Timestamp        | [Section 7 (Since-Timestamp Mode)](#7-since-timestamp-mode), [Section 11.2 (show with since option)](#112-show-with-since-option)                                         |
| [ASV2-2255](https://bugatti-asic.atlassian.net/browse/ASV2-2255) | Tech-Support Rate Limit             | [Section 8 (Rate Limiting)](#8-rate-limiting), [Section 11.3 (show and rate limit)](#113-show-and-rate-limit), [Section 12.1 (config ratelimit)](#121-config-ratelimit)        |
| [ASV2-ABCD](https://bugatti-asic.atlassian.net/browse/ASV2-2255) | Scheduled Tech-Support              | [Section 9 (Scheduled Tech-Support)](#9-scheduled-tech-support), [Section 11.4 (show schedule)](#114-show-schedule), [Section 12.2 (config schedule)](#122-config-schedule) |
| [ASV2-ABCD](https://bugatti-asic.atlassian.net/browse/ASV2-2255) | Tech-Support Bundle Anonymization   | [Section 10 (Tech-Support Anonymization)](#10-tech-support-bundle-anonymization), [Section 11.5 (show with scramble)](#115-show-with-scramble)                                |


---

## 4. Glossary


| Term                    | Definition                                                                                                                                                                             |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Tech-support bundle** | Compressed tarball (`sonic_dump_<hostname>_<timestamp>.tar.gz`) containing logs, configs, and ASIC state collected by `generate_dump.py`.                                              |
| **generate_dump.py**    | The SONiC script (in `sonic-utilities`) that orchestrates tech-support data collection. Invoked by `show techsupport`.                                                                 |
| **Since timestamp**     | An ISO 8601 timestamp or relative expression (`"1 hour ago"`, `"2026-04-03 14:32:00"`) passed to log collection commands to limit output to entries after that point.                  |
| **Rate limit**          | A policy that caps how often tech-support bundles may be generated. Expressed as a minimum interval between bundles (seconds).                                                         |
| **WJH**                 | What Just Happened — Spectrum-4's out-of-band drop capture mechanism. WJH events are included in the tech-support bundle for hardware-level drop analysis.                             |
| **CONFIG_DB**           | SONiC Redis database holding persistent switch configuration. Rate limit parameters are stored here.                                                                                   |
| **STATE_DB**            | SONiC Redis database holding runtime state. Last bundle generation time and count are stored here.                                                                                     |
| **/var/dump**           | Default output directory for tech-support bundles and kernel core dumps on SONiC.                                                                                                      |
| **SWSS rec**            | `swss.rec` and `sairedis.rec` — recording files capturing orchagent control plane operations and SAI API calls, respectively. Critical for debugging config-to-ASIC programming paths. |


---

## 5. Architecture

## 5.1 Ondemand Tech-support generation

Tech-support bundle generation is a CLI-driven, on-demand operation. The control path flows from the UCLI through `generate_dump.py`, which drives collection from all subsystems. Rate limiting is enforced at the CLI entry point, before `generate_dump.py` is invoked, by consulting and updating STATE_DB.

```
+-----------------------------------------------------------------------------+
|                    CONTROL PLANE (switch CPU)                               |
|                                                                             |
|  Operator                                                                   |
|      |                                                                      |
|      |  show techsupport [since <timestamp>] [--type <mini|default|debug>]  |
|      v                                                                      |
|  +-----------------------------------------------------------------------+  |
|  |                  sonic-utilities CLI                                  |  |
|  |                                                                       |  |
|  |  1. Check rate limit (per type):  STATE_DB -> TECHSUPPORT_STATE       |  |
|  |     last_run_time, run_count_in_window                                |  |
|  |     If within rate limit -> reject with error message                 |  |
|  |                                                                       |  |
|  |  2. Invoke generate_dump.py [--since <timestamp>]                     |  |
|  |     [--type <mini|default|debug>]                                     |  |
|  |                                                                       |  |
|  |  3. On completion: update STATE_DB last_run_time                      |  |
|  +--------------------+--------------------------------------------------+  |
|                       |                                                     |
|                       v                                                     |
|  +-----------------------------------------------------------------------+  |
|  |                  generate_dump.py                                     |  |
|  |                                                                       |  |
|  |  Collectors (each respects --since if provided):                      |  |
|  |    Selected based on --type option                                    |  |
|  |    syslog / journald   -->  time-filtered with --since                |  |
|  |    /etc/ (config)      -->  always full (no filter)                   |  |
|  |    /proc/ (runtime)    -->  always full (point-in-time)               |  |
|  |    SWSS rec files      -->  filtered by --since mtime                 |  |
|  |    SAI redis rec       -->  filtered by --since mtime                 |  |
|  |    hw-mgmt             -->  always full                               |  |
|  |    WJH events          -->  time-filtered with --since                |  |
|  |    kdump / warmboot    -->  filtered by mtime                         |  |
|  |    Platform ASIC dump  -->  always full (point-in-time)               |  |
|  +--------------------+--------------------------------------------------+  |
|                       |                                                     |
|                       v                                                     |
|  +-----------------------------------------------------------------------+  |
|  |   /var/dump/sonic_dump_<hostname>_<timestamp>.tar.gz                  |  |
|  +-----------------------------------------------------------------------+  |
|                                                                             |
|  +-------------- SONiC Redis Database Layer ----------------------------+   |
|  |  TECHSUPPORT table (rate limit config)                               |   |
|  |  TECHSUPPORT_STATE (last run, run count)                             |   |
|  |  Techsupport Bundle Info                                             |   |
|  +----------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------+
```

**Key design decisions:**

- Rate limit enforcement happens **before** `generate_dump.py` is invoked. This ensures no CPU/IO is consumed on a rejected request.
- `generate_dump.py` receives `--since` as a command-line argument. Collectors that support time-based filtering (syslog, journald, WJH) apply it. Point-in-time snapshots (ASIC state, `/proc/`) are always full regardless of `--since`.
- STATE is updated **after** a successful bundle write, not at invocation start. A failed bundle generation does not consume a rate limit slot.

---

## 5.2 Scheduled Tech-support generation

Tech-support bundle generation can be configured to trigger on a periodic schedule, with type/options specified for the collection. The schedule only controls the trigger and the actual collection happens through the tech-support pipeline described in the previous section. The config for the schedule is saved in CONFIG_DB to preserve it across reboots/upgrades, with CRUD APIs for it.

```
+-----------------------------------------------------------------------------------------------+
|                                   CONTROL PLANE (switch CPU)                                  |
|                                                                                               |
|  Operator                                                                                     |
|      |                                                                                        |
|      |  config scheduled-techsupport <add|remove|update> type <mini|default|debug>            |
|      |      [period <time interval>]                                                          |
|      v                                                                                        |
|  +-----------------------------------------------------------------------------------------+  |
|  |                      cronjob triggers the scheduler                                     |  |
|  |                                                                                         |  |
|  |  1. For each type, check configured time interval and last collected time interval.     |  |
|  |  2. Based on the check, invoke techsupport with appropriate options                     |  |
|  |  3. As techsupport is limited to one invocation at a time system-wide,                  |  |
|  |     invoke the quickest one first, if multiple types are ready at the same time.        |  |
|  +-------------------------------------------------------+---------------------------------+  |
|                                                        |                                      |
|                                                        v                                      |
|  +-----------------------------------------------------------------------------------------+  |
|  |                                tech-support trigger                                     |  |
|  +-------------------------------------------------------+---------------------------------+  |
|                                                        |                                      |
|                                                        v                                      |
|  +-----------------------------------------------------------------------------------------+  |
|  |   /var/dump/sonic_dump_<hostname>_<timestamp>.tar.gz                                    |  |
|  +-----------------------------------------------------------------------------------------+  |
|                                                                                               |
|  +------------------------ SONiC Redis Database Layer ------------------------------------+   |
|  |  TECHSUPPORT_SCHEDULE_CONFIG table (config)                                            |   |
|  |  TECHSUPPORT_SCHEDULE_STATE (last run state)                                           |   |
|  |  Techsupport Bundle Info                                                               |   |
|  +----------------------------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------------------------+
```

**Key design decisions:**

- Provide individual schedule per bundle type.
- Auto set --since from the previous invocation time from the same schedule, with an option to override.
- Persist the config and state information ensuring the schedule works across reboots and upgrades.
- STATE is updated **after** a successful bundle write, not at invocation start. A failed bundle generation will retrigger of the scheduled collection.

---

## 5.3 Log Anonymization

Tech-support bundles by default include customer sensitive information gathered from various sources. Some sensitive information like passwords and API-keys should always be redacted. Other information like ip address, node name, serial number, etc though sensitve, are useful for debugging and consistent anonymization will help alleviate customer concerns. Additional nice to have feature would be to provide the customer the ability to reverse the anonymization on-demand. 

```
+-------------------------------------------------------------------------------------------------------+
|                                   CONTROL PLANE (switch CPU)                                          |
|                                                                                                       |
|  Operator / Schedule                                                                                  |
|      |                                                                                                |
|      |  show techsupport --scramble ....                                                              |
|      |                                                                                                |
|      |                                                                                                |
|      v                                                                                                |
|  +-----------------------------------------------------------------------------------------------+    |
|  |                                                                                               |    |
|  |  1. Collect time capsule                                                                      |    |
|  |  2. Run all text files through scramble pipeline                                              |    |
|  |  3. tar.gz after scramble and update the techsupport bundle state                             |    |
|  +-----------------------------------------------------------------------------------------------+    |
|      |                                                                                                |
|      |                                                                                                |
|      v                                                                                                |
|  +-----------------------------------------------------------------------------------------------+    |
|  |   /var/dump/sonic_dump_<hostname>_<timestamp>.tar.gz                                          |    |
|  |   /var/dump/sonic_dump_<hostname>_<timestamp>.enc.key                                         |    |
|  +-----------------------------------------------------------------------------------------------+    |
|                                                                                                       |
|  +------------------------ SONiC Redis Database Layer -------------------------------------------+    |
|  |  Techsupport Bundle Info                                                                      |    |
|  +-----------------------------------------------------------------------------------------------+    |
+-------------------------------------------------------------------------------------------------------+
```

**Key design decisions:**

- Consistent across the techsupport bundle mapping, preserving the structure, subnet, etc
- Provide customer the ability to fully or selectively de-anonymize the fields
- A combination of regex and Context Specific search to identify the sensitive information

---

## 6. Tech-Support Bundle Content

### 6.1 TS-REQ-01: Bundle Generation

**[TS-REQ-01]: Tech-Support Bundle Generation**

- **Jira:** [ASV2-2253](https://bugatti-asic.atlassian.net/browse/ASV2-2253)
- **Description:** The system SHALL generate a tech-support bundle on operator demand via `show techsupport`. The bundle is a compressed tarball containing all diagnostic artifacts needed to diagnose SONiC control plane, data plane, and platform issues without requiring live access to the switch.
The system SHALL generate a tech-support bundle on operator demand via `show techsupport type mini`. The Mini bundle is a compressed tarball containing a subset of artifacts compared to the full bundle, completing the collection in a shorter time window.
- **Acceptance Criteria:**
  - `show techsupport` completes and writes a `.tar.gz` to `/var/dump/`
  - `show techsupport type mini` completes and writes a `.tar.gz` to `/var/dump/`
  - Bundle naming: `sonic_dump_<hostname>_<YYYYMMDD_HHMMSS>.tar.gz`
  - Bundle is readable (`tar -tzf <file>` exits 0)
- **Priority:** P0

### 6.2 Bundle Content Categories

The following content included in every tech-support bundle:


| Category                     | Sources                                                           | Mini Yes/No |
| ---------------------------- | ----------------------------------------------------------------- | ----------- |
| System logs                  | `/var/log/syslog`*, `/var/log/syslog.1`, compressed rotations     | Yes         |
| Messagelogs                  | `/var/log/messages`*, `/var/log/messages.1`, compressed rotations | Yes         |
| Journalctl                   | journalctl                                                        | Yes         |
| Authentication logs          | `/var/log/auth.log`*                                              | No          |
| BGP / FRR logs               | `/var/log/frr/bgpd.log`, `/var/log/frr/zebra.log`                 | No          |
| SWSS control plane recording | `/var/log/swss/swss.rec`                                          |             |
| SAI API recording            | `/var/log/swss/sairedis.rec`                                      |             |
| Running configuration        | `show running-configuration` output                               |             |
| Interface state              | `show interfaces status`, `show interfaces counters`              |             |
| BGP state                    | `show bgp summary`, `show ip route`                               |             |
| ASIC platform dump           | Platform `sx_dump` / SDK state                                    |             |
| WJH (What Just Happened)     | WJH drop events from Spectrum-4                                   |             |
| Hardware management          | `/usr/share/hw-mgmt/`                                             |             |
| Kernel dump                  | `/var/core/` (if present)                                         | No          |
| Warm boot state              | `/var/warmboot/`                                                  | Yes         |
| System info                  | `uname -a`, `df -h`, `free -m`, `uptime`                          | Yes         |


---

## 7. Since-Timestamp Mode

### 7.1 TS-REQ-02: Since-Timestamp Filtering

**[TS-REQ-02]: Tech-Support Since Timestamp**

- **Jira:** [ASV2-2254](https://bugatti-asic.atlassian.net/browse/ASV2-2254)
- **Description:** The system SHALL accept an optional `since <timestamp>` parameter that scopes log collection to entries at or after the specified time. For log files, only entries with timestamps ≥ since-time are included. For file-based artifacts, only files with mtime ≥ since-time are included. `since <timestamp>` should work both with the default and mini types.
- **Acceptance Criteria:**
  - `show techsupport since "2026-04-03 14:32:00"` generates a smaller bundle than a full `show techsupport`
  - `show techsupport type mini since "2026-04-03 14:32:00"` generates a smaller bundle than a full `show techsupport type mini`
  - Syslog entries before the since-time are excluded
  - Point-in-time snapshots (ASIC state, `/proc/`, running config) are always included regardless of since-time
  - If since-time is in the future, the CLI rejects with an error
  - If since-time predates all available logs, behavior is equivalent to a full collection (no error)
- **Priority:** P0

### 7.2 Timestamp Format

The `since` parameter SHALL accept the following formats:


| Format                 | Example                  | Notes                                |
| ---------------------- | ------------------------ | ------------------------------------ |
| ISO 8601 datetime      | `"2026-04-03 14:32:00"`  | Interpreted as local switch timezone |
| ISO 8601 with timezone | `"2026-04-03T14:32:00Z"` | UTC explicitly specified             |


### 7.3 Filtering Implementation

Log collection SHALL use the following per-source filtering approach:


| Source                                  | Filter Mechanism                                                                                                                             |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| systemd journal (`journalctl`)          | `--since <timestamp>` argument                                                                                                               |
| `/var/log/syslog`* (rsyslog flat files) | Parse RFC 3164 timestamp on each line; skip lines before since-time. Compressed rotations (`.gz`) are streamed with `zcat` before filtering. |
| SWSS rec files                          | Include file if `mtime >= since-time`. Files are append-only records; mtime reflects last write.                                             |
| WJH events                              | Query WJH subscriber with time-bounded event range.                                                                                          |
| Rotated log files (`.1`, `.2.gz`)       | Include file if `mtime >= since-time`. If mtime < since-time, the file predates the window entirely — skip.                                  |
| Kernel core files                       | Include if `mtime >= since-time`.                                                                                                            |


**Note on compressed rotations:** When since-time falls in the middle of a `.gz` rotation, the entire compressed file is included (no line-level filtering within compressed archives). This is a V1 simplification. V2 may add decompression + filtering for compressed rotations.

### 7.4 Bundle content and since applicability


| Category                     | Sources                                                       | --since applies Yes/No |
| ---------------------------- | ------------------------------------------------------------- | ---------------------- |
| System logs                  | `/var/log/syslog`*, `/var/log/syslog.1`, compressed rotations | Yes                    |
| Authentication logs          | `/var/log/auth.log`*                                          | Yes                    |
| BGP / FRR logs               | `/var/log/frr/bgpd.log`, `/var/log/frr/zebra.log`             | Yes                    |
| SWSS control plane recording | `/var/log/swss/swss.rec`                                      | Yes (mtime-based)      |
| SAI API recording            | `/var/log/swss/sairedis.rec`                                  | Yes (mtime-based)      |
| Running configuration        | `show running-configuration` output                           | No                     |
| Interface state              | `show interfaces status`, `show interfaces counters`          | No (point-in-time)     |
| BGP state                    | `show bgp summary`, `show ip route`                           | No (point-in-time)     |
| ASIC platform dump           | Platform `sx_dump` / SDK state                                | No (point-in-time)     |
| WJH (What Just Happened)     | WJH drop events from Spectrum-4                               | Yes                    |
| Hardware management          | `/usr/share/hw-mgmt/`                                         | No                     |
| Kernel dump                  | `/var/core/` (if present)                                     | Yes (mtime-based)      |
| Warm boot state              | `/var/warmboot/`                                              | No                     |
| System info                  | `uname -a`, `df -h`, `free -m`, `uptime`                      | No                     |


---

## 8. Rate Limiting

### 8.1 TS-REQ-03: Tech-Support Rate Limit

**[TS-REQ-03]: Tech-Support Rate Limit**

- **Jira:** [ASV2-2255](https://bugatti-asic.atlassian.net/browse/ASV2-2255)
- **Description:** The system SHALL enforce a configurable rate limit on tech-support bundle generation to prevent excessive CPU and disk I/O consumption. The rate limit is expressed as a minimum interval between consecutive bundles. Rate limit enforcement SHALL reject the request before invoking `generate_dump.py`.
- **Acceptance Criteria:**
  - A second `show techsupport` within the minimum interval is rejected with a clear error message including the time remaining until the next bundle is allowed
  - Rate limit state persists across CLI sessions
  - Separate rate limit for default and mini tech-support types.
  - Superuser (`sudo`) can override the rate limit with `--force`
  - A disabled rate limit (`config techsupport rate-limit disable`) allows unlimited generation
- **Priority:** P1

### 8.2 Rate Limit Parameters

Rate limit is configured per tech-support collection type - mini and default.


| Collection type | Parameter        | Config Key | Parameter Type | Range          | Default     | Description                                                        |
| --------------- | ---------------- | ---------- | -------------- | -------------- | ----------- | ------------------------------------------------------------------ |
| mini            | Minimum interval | `interval` | uint32         | 0-3600 seconds | 180 seconds | Minimum time between consecutive bundle generations. 0 = disabled. |
| default         | Minimum interval | `interval` | uint32         | 0-3600 seconds | 300 seconds | Minimum time between consecutive bundle generations. 0 = disabled. |


**Default:** 180 seconds (3 minutes) for mini and 300 seconds (5 minutes) for default tech-support bundles. This allows a human operator reacting to an incident to generate a second bundle for comparison while preventing automated loops from generating more than 20 bundles per hour. This interval shoule be independently configurable for each tech-support type.

---

## 9. Scheduled Tech-Support

### 9.1 TS-REQ-04: Scheduled Tech-Support

**[TS-REQ-04]: Scheduled Tech-Support**

- **Jira:** [ASV2-ABCD](https://bugatti-asic.atlassian.net/browse/ASV2-2256)
- **Description:** The system SHALL provide a mechanism to schedule tech-support bundle generation at configurable times or intervals, supporting automated collection (e.g., for periodic health checks or proactive incident capture). Scheduling persists across reboots. CLI commands control schedule config, list, and removal. Invocations triggered by schedule are subject to rate limiting and all safeguards.
- **Acceptance Criteria:**
  - Operator can configure, list, and remove scheduled bundles via CLI, with the key being the type.
  - Ability to configure independent schedules for different colleciton types (mini, default)
  - Scheduled invocations bundle will persist associated metadata used in techsupport listing.
  - CLI and logs clearly indicate scheduled vs. on-demand invocations.
  - Schedule survives reboot and warmboot.
  - Optionally can be extended to have named schedules, instead of the type being the key. (P2)
- **Priority:** P1

### 9.2 Scheduled Tech-support Parameters


| Collection type | Parameter | Config Key | ISO 8601 Period | Range            | Default    | Description                                           |
| --------------- | --------- | ---------- | --------------- | ---------------- | ---------- | ----------------------------------------------------- |
| mini            | Period    | `interval` | timestamp       | Minutes - 7 days | No default | Period between consecutive bundle generation trigger. |
| default         | Period    | `interval` | timestamp       | Minutes - 7 days | No default | Period between consecutive bundle generation trigger. |


---

## 10. Tech-Support Bundle Anonymization

### 10.1 TS-REQ-05: Tech-Support Bundle Anonymization

**[TS-REQ-05]: Tech-Support Bundle Anonymization**

- **Jira:** [ASV2-ABCD](https://bugatti-asic.atlassian.net/browse/ASV2-2255)
- **Description:** The system will provide an option to anonymize tech-support bundles, both during the generation or post-process. 
- **Acceptance Criteri a:**
  - Additional parameter to trigger anonymized logs either through manual or scheduled triggers.
  - Option to anonymize a specific tech-support bundle post successful generation.
  - Abiility to de-anonymize a specific bundle on-demand.
  - Bundle name should indicate that the bundle is scrambled.
- **Priority:** P1

### 10.2 Tech-support generation Parameter


| Parameter | Default     | Description                                            |
| --------- | ----------- | ------------------------------------------------------ |
| scramble  | No scramble | Option to scramble sensitive information in the bundle |


### 10.3 Tech-support post-process Command

Command to scramble / unscramble an already generated tech-support.

| Parameter |  Description |
|-----------|------------|------|
| scramble |  Option to scramble sensitive information in a specific tech-support bundle |
| unscramble |  Option to unscramble sensitive information in a specific tech-support bundle |

---

## 11. Tech-support show CLI

### 11.1 show techsupport

```
! Generate a full tech-support bundle
switch# show techsupport

! Generate a mini tech-support bundle
switch# show techsupport type mini

```

### 11.2 show with since option

```
! Generate a bundle scoped to events since a timestamp
switch# show techsupport since "2026-04-03 14:32:00"

! Combine with mini techsupport
switch# show techsupport since "2026-04-03T06:32:00Z" type mini
```

### 11.3 show and rate limit

```
! Override rate limit (requires sudo)
switch# sudo show techsupport --force

! Combine since and force
switch# sudo show techsupport since "2026-04-03T06:32:00Z" --force

! Combine with mini techsupport
switch# sudo show techsupport type mini --force

```

```

! Show the current rate limit setting.
switch# show techsupport ratelimit

```

### 11.4 show schedule

```
! Show the current scheduled techsupport rules.
show techsupport schedule
```

### 11.5 show with scramble

```
! Trigger techsupport with anonymization
show techsupport scramble

! Combine with mini
show techsupport scramble type mini

! Combine with since
show techsupport since "2026-04-03T06:32:00Z" scramble

```

```
! Stand alone command to scramble/unscramble already generated tech-support bundle
show techsupport scramble <bundle name>

show techsupport unscramble <bundle name>
```

### 11.6 List techsupport bundles

```
show techsupport history
```

### 11.7 Delete a techsupport bundle

```
show techsupport delete <bundle-name>
```

### 11.8 Show command syntax

The `since` keyword is a positional argument to `show techsupport`. No configuration is required -- it is a per-invocation parameter.

```
show techsupport [since <timestamp>] [scramble] [type <mini/default>] [--force]

Arguments:
  since <timestamp>   Scope log collection to entries at or after <timestamp>.
                      Accepts ISO 8601 datetime, relative expressions ("1 hour ago"),
                      or Unix epoch integers.
  type <mini/default> Option to trigger a mini tech-support bundle or the default one.
  scramble            Option to scramble the sensitive information in the tech-support bundle
  --force             Override rate limit. Requires sudo.
```

```
show techsupport schedule
```

```
show techsupport ratelimit
```

```
show techsupport scramble <bundle name>

show techsupport unscramble <bundle name>
Arguments:
  <bundle name>   Name of the existing bundle to be scrambled/unscrambled.
```

```
show techsupport history
```

```
show techsupport delete <bundle name>
```

---

## 12. Tech-support Config CLI

### 12.1 config ratelimit

**Tighten rate limit for high-security environment**

```
(config)# techsupport rate-limit interval 600
```

**Disable rate limit during initial bring-up / lab work:**

```
(config)# techsupport rate-limit interval 0
```

**Relax rate limit for mini Tech-support bundles**

```
(config)# techsupport rate-limit type mini interval 100
```

### 12.2 config schedule

**Schedule a default tech-support every day**

```
(config)# techsupport schedule period P1D
```

**Schedule a mini tech-support every hour**

```
(config)# techsupport schedule type mini period PT1H
```

**Delete Tech-support schedule**

```
(config)# techsupport schedule delete
```

```
(config)# techsupport schedule delete type mini
```

## 13. SONiC Integration

### 13.1 Component Changes


| Component                  | Repository          | Change                                                                                                        |
| -------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------- |
| `show techsupport` CLI     | `sonic-utilities`   | Add `since <timestamp>` argument; add `--force` flag; add rate limit check before invoking `generate_dump.py` |
| `show techsupport` CLI     | `sonic-utilities`   | Add `type mini` argument; To trigger the mini tech-support bundle                                             |
| `generate_dump.py`         | `sonic-utilities`   | Add `--since <timestamp>` argument; implement per-collector time filtering                                    |
| `config techsupport` CLI   | `sonic-utilities`   | New `techsupport rate-limit interval <n>` and `no techsupport rate-limit` commands                            |
| `config techsupport` CLI   | `sonic-utilities`   | New `techsupport schedule <period>` and `no techsupport schedule` commands                                    |
| `show techsupport history` | `sonic-utilities`   | New command to read tech-support bundle information                                                           |
| YANG model                 | `sonic-buildimage`  | `sonic-techsupport.yang` for `TECHSUPPORT` CONFIG_DB table                                                    |
| CONFIG_DB schema           | `sonic-swss-common` | New `TECHSUPPORT` table                                                                                       |
| STATE_DB schema            | `sonic-swss-common` | New `TECHSUPPORT_STATE` table                                                                                 |


---

### 13.2 CONFIG_DB Schema

**Table: TECHSUPPORT_RATE_LIMIT**


| Key Format              | Field    | Type     | Default | Description |
| ----------------------- | -------- | -------- | ------- | ----------- |
| `TECHSUPPORT_RATE_LIMIT | default` | interval | uint32  | `300`       |
| `TECHSUPPORT_RATE_LIMIT | mini`    | interval | uint32  | `180`       |


**Table: TECHSUPPORT_SCHEDULE**


| Key Format            | Field    | Type   | Default  | Description |
| --------------------- | -------- | ------ | -------- | ----------- |
| `TECHSUPPORT_SCHEDULE | default` | period | ISO 8601 | "PT0S"      |
| `TECHSUPPORT_SCHEDULE | mini`    | period | ISO 8601 | "PTOS"      |


---

### 13.3 STATE_DB Schema

**Table: TECHSUPPORT_RUN_INFO**


| Key Format            | Field    | Type               | Description |
| --------------------- | -------- | ------------------ | ----------- |
| `TECHSUPPORT_RUN_INFO | default` | last_run_timestamp | uint64      |
| `TECHSUPPORT_RUN_INFO | mini`    | last_run_timestamp | uint64      |


**Note:** STATE_DB is ephemeral -- it is not persisted to disk and is cleared on reboot. Rate limit state therefore resets on every reboot, which is the correct behavior (a rebooting switch should be able to immediately generate a bundle for the crash analysis).

The below information maybe mirrored on a json file in /var/dump to preserve across reboots/upgrades.
**Table: TECHSUPPORT_SCHEDULE_RUN_INFO**


| Key Format                     | Field    | Type               | Description |
| ------------------------------ | -------- | ------------------ | ----------- |
| `TECHSUPPORT_SCHEDULE_RUN_INFO | default` | last_run_timestamp | uint64      |
| `TECHSUPPORT_SCHEDULE_RUN_INFO | mini`    | last_run_timestamp | uint64      |


**Table: TECHSUPPORT_BUNDLE_INFO**


| Key Format               | Field            | Type                | Description                                                       |
| ------------------------ | ---------------- | ------------------- | ----------------------------------------------------------------- |
| `TECHSUPPORT_BUNDLE_INFO | `                | last_run_timestamp` | timestamp                                                         |
|                          | trigger          | string              | `manual` or `scheduled` baased on the trigger.                    |
|                          | type             | string              | `mini` or `default` based on the invocation.                      |
|                          | since            | timestamp           | timestamp if since was used in generation. `0` otherwise.         |
|                          | force            | boolean             | `true` if force option was used in generation. `false` otherwise. |
|                          | scramble         | boolean             | `true` if the bundle is scrambled. `false` otherwise.             |
|                          | enc_key_filename | string              | Associated key file for scrambled bundle.                         |
|                          | status           | string              | "complete                                                         |


---

### 13.4 SAI API

No SAI API changes. Tech-support bundle generation is a control-plane-only feature. Platform-specific ASIC dump commands (vendor SDK `sx_dump`) are already invoked by `generate_dump.py` and are unchanged.

---

### 13.5 Warm Reboot Considerations

Tech-support bundle generation is not active during warm reboot. STATE_DB is cleared on reboot, so the rate limit timer resets. This is intentional -- an operator diagnosing a warm reboot failure should be able to generate a bundle immediately.

---

## 14. Operational Commands & Monitoring

### 14.1 DB dump

```
sonic-db-cli CONFIG_DB HGETALL "TECHSUPPORT_RATE_LIMIT|default"
sonic-db-cli CONFIG_DB HGETALL "TECHSUPPORT_RATE_LIMIT|mini"
sonic-db-cli CONFIG_DB HGETALL "TECHSUPPORT_SCHEDULE|default"
sonic-db-cli CONFIG_DB HGETALL "TECHSUPPORT_SCHEDULE|mini"
```

```
sonic-db-cli STATE_DB HGETALL "TECHSUPPORT_RUN_INFO|default"
sonic-db-cli STATE_DB HGETALL "TECHSUPPORT_RUN_INFO|mini"
sonic-db-cli STATE_DB HGETALL "TECHSUPPORT_SCHEDULE_RUN_INFO|default"
sonic-db-cli STATE_DB HGETALL "TECHSUPPORT_SCHEDULE_RUN_INFO|mini"
sonic-db-cli STATE_DB KEYS "TECHSUPPORT_BUNDLE_INFO|*"
sonic-db-cli STATE_DB HGETALL "TECHSUPPORT_BUNDLE_INFO|..."
```

### 14.2 Syslog Messages

The following syslog messages SHALL be emitted by the tech-support system:


| Event                       | Severity | Message Format                                                                         |
| --------------------------- | -------- | -------------------------------------------------------------------------------------- |
| Bundle generation started   | INFO     | `TECHSUPPORT: Bundle generation started. since=<timestamp or "full">`                  |
| Bundle generation completed | INFO     | `TECHSUPPORT: Bundle written to <path> (<size_mb> MB, elapsed <seconds>s)`             |
| Bundle generation failed    | ERROR    | `TECHSUPPORT: Bundle generation failed: <error>`                                       |
| Rate limit rejected         | WARNING  | `TECHSUPPORT: Bundle generation rejected (rate limit). Remaining interval: <seconds>s` |
| Rate limit forced override  | WARNING  | `TECHSUPPORT: Rate limit overridden by <username> (--force)`                           |


---

## 15. Error Handling & Troubleshooting

### 15.1 Error Conditions


| Error Condition                                          | System Behavior                                                                                                                                | Recovery                                                          |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Rate limit active                                        | CLI rejects with remaining seconds displayed. Syslog WARNING. STATE_DB not modified.                                                           | Wait for interval to expire, or use `--force` with sudo.          |
| `/var/dump` full                                         | `generate_dump.py` fails to write tarball. CLI error message. Syslog ERROR. STATE_DB not modified.                                             | Delete old bundles: `sudo rm /var/dump/sonic_dump_*.tar.gz`.      |
| Invalid since-timestamp                                  | CLI rejects before invoking `generate_dump.py`. Error message with expected format.                                                            | Provide a valid timestamp in a supported format.                  |
| Since-timestamp in the future                            | CLI rejects with error: "since timestamp is in the future".                                                                                    | Provide a past timestamp.                                         |
| `generate_dump.py` partial failure (one collector fails) | Remaining collectors proceed. Failed collector's section is omitted from bundle. Syslog WARNING per failed collector. Bundle is still written. | Inspect the syslog within the bundle itself for collector errors. |
| Since-timestamp predates all logs                        | No error. Bundle is equivalent to a full collection. Syslog INFO notes that since-time predates oldest available log.                          | Expected behavior — no action needed.                             |


### 15.2 Troubleshooting Guide

**Problem: `show techsupport` rejected with rate limit error**

Override if needed: `sudo show techsupport --force`
Adjust rate limit: `(config)# techsupport rate-limit interval 60`

**Problem: Bundle is too large / takes too long**
Use the `since` option to limit the time window
Use the `type mini` option to generate a smaller bundle

**Problem: Failed due to space errors**
Check what is consuming space: `ls -lh /var/dump/*.tar.gz`
Delete some bundles and retrigger.

**Problem: Bundle is missing logs for the incident time window**
Verify the log files haven't been rotated away:

---

## 16. Interactions & Dependencies

### 16.1 Feature Dependencies


| Dependency            | Why                                                                                                                                                   |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| STATE_DB (Redis)      | Rate limit state tracking requires Redis. If Redis is unavailable, the rate limit check is skipped with a syslog WARNING and the bundle is generated. |
| CONFIG_DB (Redis)     | Rate limit config requires Redis. If unavailable, default interval (180s) is used.                                                                    |
| `/var/dump` directory | Must exist and have write permission. Standard SONiC setup guarantees this.                                                                           |
| `generate_dump.py`    | Must be present in sonic-utilities. This FS does not introduce a new script -- it extends the existing one.                                           |


---

## 17. Scale & Performance


| Parameter                                | Target                             | Notes                                                                                    |
| ---------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------- |
| Full bundle generation time              | < 10 minutes                       | Includes ASIC dump (~8s), log compression, tarball write. Baseline varies by log volume. |
| Since-scoped bundle generation time      | < 2 minutes (for 1-hour window)    | Log collection is the bottleneck; filtering eliminates most I/O.                         |
| Full bundle size                         | 50-300 MB                          | Varies by log verbosity, uptime, core dumps present.                                     |
| Since-scoped bundle size (1-hour window) | 5-30 MB                            | Typically 10-20x smaller than full bundle.                                               |
| mini bundle generation time              | < 2 minutes                        | Includes critical debug and point in time info.                                          |
| mini bundle size                         | 10 MB                              | Includes critical debug and point in time info.                                          |
| Rate limit enforcement overhead          | < 1 ms                             | Two STATE_DB reads + one CONFIG_DB read.                                                 |
| `/var/dump` space consumed per bundle    | 50-300 MB (full), 5-30 MB (scoped) | Operators should monitor `/var/dump` usage.                                              |
| Min rate limit interval                  | 0 (disabled)                       | Upper bound is 3600s (1 hour).                                                           |


---

## 18. Engineering Tasks


| ID         | Description                                                                                                                               | Dependency | Jira Reqs            |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ---------- | -------------------- |
| TS-TASK-01 | CONFIG_DB schema: `TECHSUPPORT_RATE_LIMIT` table with `interval` field; YANG model `sonic-techsupport.yang`                               | —          | TS-REQ-03            |
| TS-TASK-02 | CONFIG_DB schema: `TECHSUPPORT_SCHEDULE` table with `period` field; YANG model `sonic-techsupport.yang`                                   | —          | TS-REQ-04            |
| TS-TASK-03 | STATE_DB schema: `TECHSUPPORT_RUN_INFO` table with `last_run_timestamp`                                                                   | —          | TS-REQ-03            |
| TS-TASK-04 | STATE_DB schema: `TECHSUPPORT_SCHEDULE_RUN_INFO` table with `last_run_timestamp`                                                          | —          | TS-REQ-04            |
| TS-TASK-05       | STATE_DB schema: `TECHSUPPORT_BUNDLE_INFO` table with all the fields                                                                      | —          | TS-REQ-05            |
| TS-TASK-06       | CLI `config techsupport rate-limit interval <interval>`: write to CONFIG_DB                                                                      | TS-TASK-01       | TS-REQ-03            |
| TS-TASK-07       | CLI `show techsupport rate-limit`: read CONFIG_DB, display config                                                                      | TS-TASK-01       | TS-REQ-03            |
| TS-TASK-08       | CLI `config techsupport schedule period <period>`: write to CONFIG_DB                                                                      | TS-TASK-02       | TS-REQ-04            |
| TS-TASK-09       | CLI `show techsupport schedule`: read CONFIG_DB, display config                                                                      | TS-TASK-01       | TS-REQ-04            |
| TS-TASK-10       | Rate limit enforcement in `show techsupport` entry point: read STATE_DB + CONFIG_DB, compute elapsed time, reject or proceed              | TS-TASK-01, TS-TASK-03 | TS-REQ-03            |
| TS-TASK-11       | `--force` flag for `show techsupport`: bypass rate limit check, log override to syslog                                                    | -       | TS-REQ-03            |
| TS-TASK-12       | STATE_DB update on successful bundle write: record `last_run_timestamp`                                            | TS-TASK-03 | TS-REQ-03            |
| TS-TASK-13       | STATE_DB update on successful bundle write: record `last_run_timestamp`                                            | TS-TASK-04 | TS-REQ-04            |
| TS-TASK-14       | STATE_DB update on successful bundle write: record Bundle Info                                            | TS-TASK-05 | TS-REQ-05            |
| TS-TASK-15       | add `--since <timestamp>` argument; implement timestamp parsing for all supported formats                             | —          | TS-REQ-02            |
| TS-TASK-16      | add `type mini` support to generate a smaller bundle. | -       | TS-REQ-01            |
| TS-TASK-17      | add support for `scramble` and `unscramble` | - | TS-REQ-06 |
| TS-TASK-18      | add support for techsupport bundle `list` and `delete` APIs | - | TS-REQ-06 |
| TS-TASK-19      | Syslog messages: emit INFO/WARNING/ERROR syslog events for all bundle generation lifecycle events                                         | - | - |


---

## 19. Testing Considerations

The comprehensive test plan will be  maintained separately. Key test categories are summarized here.

### 19.1 Unit Tests
- Timestamp parsing: all supported formats (ISO 8601 with/without timezone, relative, Unix epoch)
- Invalid timestamp rejection (future timestamp, malformed string)
- Rate limit enforcement: elapsed < interval → reject; elapsed ≥ interval → proceed
- `--force` bypasses rate limit regardless of elapsed time
- CONFIG_DB/STATE_DB read/write correctness for all rate limit parameters
- STATE_DB absence (Redis unavailable): rate limit skipped, bundle proceeds

### 19.2 Integration Tests
- `show techsupport` end-to-end: bundle file written, readable, non-empty
- `show techsupport type mini` end-to-end: bundle file written, readable, non-empty and smaller than the default bundle
- `show techsupport since <interval>"`: bundle smaller than full bundle; syslog entries before since-time absent from bundle
- `show techsupport type mini since <interval>` end-to-end: bundle file written, readable, non-empty and smaller than the mini bundle
- Rate limit enforcement: second invocation within interval rejected with correct remaining-time message; STATE_DB updated after first success
- `--force` override: second invocation within interval succeeds; syslog WARNING emitted with username
- `config techsupport rate-limit interval 60`: subsequent rate limit enforces 60s interval
- `config techsupport rate-limit interval 0`: rate limit disabled, unlimited generation
- `config techsupport schedule period PT1H`: triggers a scheduled techsupport every hour
- `config techsupport schedule type mini period PT1H`: triggers a scheduled mini techsupport every hour
- `show techsupport scramble` end-to-end: scrambled bundle file written, readable, non-empty
- `show techsupport scramble <techsupport bundle name>` end-to-end: scrambled bundle file written, readable, non-empty
- `show techsupport unscramble <techsupport bundle name>` end-to-end: scrambled bundle file written, readable, non-empty

### 19.3 Scale Tests

- Since-scoped bundle (1-hour window) on switch with 3 GB of rotated logs: verify performance within target (< 2 minutes)
- Full bundle on switch with full `/var/log` partition (3.9 GB): verify completion within 10 minutes
- Concurrent invocations: second simultaneous `show techsupport` is rejected by rate limit while first is in progress

### 19.4 Negative Tests

- `/var/dump` full: verify clean error message, no STATE_DB update
- `generate_dump.py` partial failure (one collector throws exception): verify bundle still written, failed collector absent, syslog WARNING emitted
- Since-timestamp with no matching logs: verify bundle contains point-in-time snapshots, syslog INFO noting since-time predates oldest log

---

## 20. Open Issues & TODOs


| #   | Issue                                                                                                                                                                      | Owner | Notes                                                                                        |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- | -------------------------------------------------------------------------------------------- |
| 1   | `/var/dump` space monitoring: should CLI warn when `/var/dump` is > 80% full? Or should this be a separate alerting feature?                                               | TBD   | Out of scope for V1.                                                                         |
| 2   | Rate limit persistence across reboots: currently STATE_DB is ephemeral (resets on boot). Confirm this is the correct behavior -- or should we persist to CONFIG_DB?        | Venki | Current decision: ephemeral is correct for crash analysis.                                   |
| 3   | `--force` permission model: should `--force` require `sudo` only, or should there be a dedicated capability/role?                                                          | TBD   | V1: sudo is sufficient.                                                                      |

---

## 21. References


| Document                             | Link                                                                                           |
| ------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Jira PRD: Show tech-support          | [ASV2-2252](https://bugatti-asic.atlassian.net/browse/ASV2-2252)                               |
| Jira: Tech-Support Bundle Generation | [ASV2-2253](https://bugatti-asic.atlassian.net/browse/ASV2-2253)                               |
| Jira: Since Timestamp Support        | [ASV2-2254](https://bugatti-asic.atlassian.net/browse/ASV2-2254)                               |
| Jira: Rate Limit Support             | [ASV2-2255](https://bugatti-asic.atlassian.net/browse/ASV2-2255)                               |
| SONiC Logging Revamp                 | [log-rotate-techsupport-archive-task.md](../../logging/log-rotate-techsupport-archive-task.md) |
| AI Fabric Performance FS/DS          | [AI_FABRIC_PERFORMANCE_FS_DS.md](../../ai-fabric-performance/AI_FABRIC_PERFORMANCE_FS_DS.md)   |
| Adaptive Routing FS (reference)      | [ADAPTIVE_ROUTING_FUNCTIONAL_SPEC.md](../adaptive-routing/ADAPTIVE_ROUTING_FUNCTIONAL_SPEC.md) |
| sonic-utilities (generate_dump.py)   | [sonic-net/sonic-utilities](https://github.com/sonic-net/sonic-utilities)                      |
| Tech-support feature spec            | [tech_support_feature_spec](https://github.com/upscale-ai-network/engineering-notes/pull/68)   |
| Tech-support scrambling spec         |                                                                                                |
