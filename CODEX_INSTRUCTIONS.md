# Codex Build Prompt: Windows Workstation Memory and Performance Collector

Build a complete, production-quality PowerShell script named:

```text
workstation_telemetry_collector.ps1
```

The script will collect lightweight workstation memory and performance telemetry over several weeks so the resulting CSV files can be analysed to justify a RAM upgrade.

The target system is:

* Windows 11 Pro
* PowerShell 7 preferred
* Windows PowerShell 5.1 compatibility desirable where practical
* Dell Pro 14 PC14250
* Intel Core Ultra 5 225U
* 16 GB installed RAM
* Standard non-administrator user during normal operation
* Administrator access available during installation and removal

The finished output must be a single, self-contained `.ps1` script.

Do not require external PowerShell modules, third-party programs, NuGet packages, services, databases, or Internet access.

Use only Windows and PowerShell functionality available locally.

## Primary objectives

The script must:

1. Install itself into a stable system-wide location.
2. Create and manage its own Scheduled Task.
3. Run unattended as `SYSTEM`.
4. Collect a lightweight telemetry sample every five minutes.
5. Append samples into one CSV file per calendar day.
6. Record system memory pressure and useful process-level data.
7. Support clean self-uninstallation.
8. Preserve collected logs during a normal uninstall.
9. Provide clear status, validation, and diagnostic commands.
10. Fail early and clearly when prerequisites are not met.
11. Never silently lose telemetry because of a transient process or performance-counter error.

## Required command-line interface

Use a PowerShell `param()` block with explicit parameter sets or mutually exclusive switches.

Support the following commands:

```powershell
.\workstation_telemetry_collector.ps1 -Install
.\workstation_telemetry_collector.ps1 -Uninstall
.\workstation_telemetry_collector.ps1 -Uninstall -Purge
.\workstation_telemetry_collector.ps1 -RunOnce
.\workstation_telemetry_collector.ps1 -Status
.\workstation_telemetry_collector.ps1 -Validate
.\workstation_telemetry_collector.ps1 -ShowConfig
.\workstation_telemetry_collector.ps1 -Export
.\workstation_telemetry_collector.ps1 -Version
.\workstation_telemetry_collector.ps1 -Help
```

Optional installation parameters:

```powershell
-InstallPath <string>
-SampleIntervalMinutes <int>
-TopProcessCount <int>
-RetentionDays <int>
-TaskName <string>
```

Defaults:

```text
InstallPath: C:\ProgramData\WorkstationTelemetry
SampleIntervalMinutes: 5
TopProcessCount: 20
RetentionDays: 90
TaskName: Workstation Telemetry Collector
```

Validate all parameter values.

Suggested limits:

```text
SampleIntervalMinutes: 1 to 60
TopProcessCount: 5 to 100
RetentionDays: 7 to 3650
```

Reject invalid combinations such as:

```powershell
-Install -Uninstall
-Uninstall -RunOnce
-Purge without -Uninstall
```

Return meaningful process exit codes.

## Installed folder structure

The installer must create:

```text
C:\ProgramData\WorkstationTelemetry\
├── workstation_telemetry_collector.ps1
├── config.json
├── install.json
├── logs\
├── archive\
├── diagnostics\
└── exports\
```

Use `Join-Path` throughout rather than manually concatenating paths.

Do not assume that the script is initially running from a writable directory.

## Self-install behaviour

When launched with `-Install`, the script must:

1. Confirm that Windows is the operating system.
2. Confirm that the current process is elevated.
3. Confirm that Task Scheduler services and cmdlets are available.
4. Confirm that the target installation path is valid.
5. Validate free disk space.
6. Create the installation directory structure.
7. Copy the current script into the installation directory.
8. Avoid copying the script over itself unnecessarily.
9. Create or update `config.json`.
10. Create `install.json`.
11. Register or update the Scheduled Task.
12. Validate the registered task.
13. Trigger an immediate test run.
14. Confirm that a valid CSV row was created.
15. Display the final installation status and paths.

Installation must be idempotent.

Running `-Install` again must safely update the installed script, configuration, and Scheduled Task without deleting existing CSV logs.

Do not create duplicate tasks.

Before replacing an installed script, copy the old installed script into the diagnostics directory with a timestamped filename.

Example:

```text
diagnostics\workstation_telemetry_collector.20260729-154200.ps1
```

## Administrator elevation

For administrative commands such as `-Install`, `-Uninstall`, and `-Purge`:

* Detect whether the current process is elevated.
* Do not silently continue without elevation.
* Either:

  * automatically relaunch the same script with `RunAs` while preserving all arguments safely, or
  * exit with a clear instruction to rerun in an elevated PowerShell session.

Automatic elevation is preferred if it can be implemented robustly.

Do not construct an unsafe unquoted argument string.

Handle script paths containing spaces.

## Installed script execution

The Scheduled Task must always execute the installed copy:

```text
C:\ProgramData\WorkstationTelemetry\workstation_telemetry_collector.ps1
```

It must not refer to the original source path used during installation.

Use the best available PowerShell executable in this order:

1. `pwsh.exe`
2. `%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe`

Store the chosen executable path in `install.json`.

The Scheduled Task action should resemble:

```text
pwsh.exe -NoLogo -NoProfile -NonInteractive -ExecutionPolicy Bypass -File "C:\ProgramData\WorkstationTelemetry\workstation_telemetry_collector.ps1" -RunOnce
```

Use proper argument quoting.

## Scheduled Task requirements

Use the `ScheduledTasks` PowerShell module where available.

The task must:

* Run as `SYSTEM`
* Use the highest available privileges
* Run whether or not a user is logged on
* Run hidden
* Start every five minutes by default
* Use the configured sample interval
* Start at system startup
* Run on AC power
* Continue when switching to battery
* Be allowed to start while on battery
* Not stop merely because the computer switches to battery
* Start when available after a missed scheduled start
* Ignore or prevent overlapping executions
* Restart after failure
* Use a finite execution time limit, such as three minutes
* Avoid waking the computer unless clearly justified
* Avoid duplicate triggers
* Use a clear task description

Use task settings equivalent to:

```text
MultipleInstances: IgnoreNew
StartWhenAvailable: true
AllowStartIfOnBatteries: true
DontStopIfGoingOnBatteries: true
ExecutionTimeLimit: 3 minutes
RestartCount: 3
RestartInterval: 1 minute
```

A practical implementation may use:

* one startup trigger; and
* one repeating time trigger.

Ensure the repetition duration is sufficiently long or indefinite.

Validate the generated task XML or Scheduled Task object after registration.

## Self-uninstall behaviour

When launched with:

```powershell
-Uninstall
```

the script must:

1. Require elevation.
2. Stop the Scheduled Task if it is running.
3. Disable the task.
4. Unregister the task.
5. Verify that the task no longer exists.
6. Preserve:

   * CSV logs
   * archives
   * exports
   * configuration
   * diagnostics
7. Clearly report the retained installation path.

Do not immediately delete the script currently executing.

If removal of the installed script is desired during normal uninstall, launch a separate cleanup process that waits until the current PowerShell process exits before deleting only the installed script and operational files.

It is acceptable for normal `-Uninstall` to leave the complete data directory in place.

When launched with:

```powershell
-Uninstall -Purge
```

the script must:

1. Show exactly which installation directory will be deleted.
2. Refuse dangerous paths such as:

   * `C:\`
   * `C:\ProgramData`
   * empty paths
   * environment-variable roots
   * paths that do not match the expected installation metadata
3. Stop and unregister the Scheduled Task.
4. Remove the entire installation directory safely.
5. Use a delayed cleanup process if needed because the running script is inside the directory.
6. Confirm or clearly report whether deletion was successfully scheduled.

Never recursively delete a directory unless it has passed strict safety validation.

## Configuration file

Create `config.json` with a schema similar to:

```json
{
  "schema_version": 1,
  "sample_interval_minutes": 5,
  "top_process_count": 20,
  "retention_days": 90,
  "log_directory": "C:\\ProgramData\\WorkstationTelemetry\\logs",
  "archive_directory": "C:\\ProgramData\\WorkstationTelemetry\\archive",
  "export_directory": "C:\\ProgramData\\WorkstationTelemetry\\exports",
  "collect_process_metrics": true,
  "collect_cpu_metrics": true,
  "collect_disk_metrics": true,
  "collect_pagefile_metrics": true,
  "collect_battery_metrics": true,
  "collect_development_workload_counts": true,
  "working_set_sort": "PrivateBytesMB"
}
```

The script must validate the configuration before collection.

If configuration is malformed:

* do not overwrite it silently;
* copy it into diagnostics with a timestamp;
* record a clear error;
* use safe defaults for that run if possible.

Write JSON using sufficient depth and UTF-8 encoding.

Use atomic replacement when updating configuration files.

## Installation metadata

Create `install.json` containing at least:

```json
{
  "schema_version": 1,
  "installed_at": "ISO-8601 timestamp",
  "installed_by": "DOMAIN\\User",
  "computer_name": "COMPUTER",
  "script_version": "1.0.0",
  "installed_script_path": "C:\\ProgramData\\WorkstationTelemetry\\workstation_telemetry_collector.ps1",
  "task_name": "Workstation Telemetry Collector",
  "powershell_executable": "C:\\Program Files\\PowerShell\\7\\pwsh.exe",
  "source_script_path": "original source path"
}
```

Do not store passwords, tokens, credentials, or sensitive user content.

## Script version

Define one authoritative version variable near the top:

```powershell
$script:ScriptVersion = '1.0.0'
```

The `-Version` command must display it.

Include a CSV schema version in every output row.

## Collection model

Each scheduled invocation should collect one telemetry sample and exit.

Do not leave a permanent PowerShell process running.

The Scheduled Task supplies the repetition.

Each invocation should normally finish within several seconds.

## Concurrency protection

Prevent two collection instances from writing simultaneously.

Use a named system-wide mutex or another robust inter-process lock.

Example concept:

```text
Global\WorkstationTelemetryCollector
```

If another collector is already running:

* log a diagnostic message;
* exit successfully or with a specific nonfatal exit code;
* do not wait indefinitely;
* do not corrupt the CSV.

Release the lock reliably in a `finally` block.

## Daily CSV files

Write one CSV file per local calendar day:

```text
logs\2026-07-29.csv
logs\2026-07-30.csv
```

Use a filename format that sorts chronologically:

```text
yyyy-MM-dd.csv
```

Use local time for daily file selection.

Also record both:

* local timestamp with offset; and
* UTC timestamp.

Example:

```text
TimestampLocal: 2026-07-29T15:42:00.1234567+09:30
TimestampUtc: 2026-07-29T06:12:00.1234567Z
```

Use invariant culture for numeric CSV output.

The CSV must:

* contain a header exactly once;
* remain valid when opened in Excel;
* use consistent columns across all rows;
* quote fields correctly;
* use UTF-8 encoding;
* survive process names or usernames containing punctuation;
* never append a second header after an interrupted run.

Before appending, validate that an existing file starts with the expected header.

If an existing daily CSV has the wrong schema or header:

* do not append incompatible rows;
* rename it into diagnostics;
* create a fresh daily CSV;
* record the reason.

## CSV row design

Use one row per sample.

The row should contain system metrics and aggregated process metrics.

Do not create one row per process in the primary daily CSV.

Process detail should be stored in fixed JSON fields within the CSV row or in a separate process CSV.

The preferred design is two daily CSV files:

```text
logs\2026-07-29-system.csv
logs\2026-07-29-processes.csv
```

### System CSV

One row per sample.

### Processes CSV

One row per selected process per sample.

This design is preferred because it is easy to analyse later and avoids embedding JSON inside CSV.

Use the same `SampleId` and timestamps in both files so they can be joined.

Generate `SampleId` as a GUID without braces.

Example:

```text
4f78b9303d624fe7be43581c7e13af07
```

## Required system CSV fields

Include at least:

```text
SchemaVersion
CollectorVersion
SampleId
TimestampLocal
TimestampUtc
ComputerName
UserName
SessionUserCount
BootTimeLocal
UptimeSeconds
InstalledMemoryMB
AvailableMemoryMB
UsedMemoryMB
UsedMemoryPercent
CommittedMemoryMB
CommitLimitMB
CommitPercent
PhysicalMemoryLoadPercent
PageFileAllocatedMB
PageFileCurrentUsageMB
PageFilePeakUsageMB
MemoryPagesPerSec
MemoryPageReadsPerSec
MemoryPageWritesPerSec
MemoryPageFaultsPerSec
MemoryCacheMB
MemoryPoolPagedMB
MemoryPoolNonPagedMB
MemoryCompressedMB
SystemProcessCount
SystemThreadCount
SystemHandleCount
TotalCpuPercent
ProcessorQueueLength
LogicalProcessorCount
DiskQueueLength
DiskReadBytesPerSec
DiskWriteBytesPerSec
SystemDriveFreeGB
SystemDriveFreePercent
PowerSource
BatteryChargePercent
BatteryStatus
ChromeProcessCount
EdgeProcessCount
FirefoxProcessCount
CursorProcessCount
VSCodeProcessCount
PowerShellProcessCount
PwshProcessCount
WindowsTerminalProcessCount
WSLProcessCount
DockerProcessCount
HyperVWorkerProcessCount
VMwareProcessCount
VirtualBoxProcessCount
CollectionDurationMilliseconds
CollectionWarningCount
CollectionErrorCount
```

Use blank values where a metric is unavailable.

Do not use misleading zero values for unavailable information.

## Memory calculation requirements

Calculate memory values consistently.

Installed physical RAM should preferably come from:

```powershell
Get-CimInstance Win32_ComputerSystem
```

Available physical RAM may come from:

```powershell
Get-CimInstance Win32_OperatingSystem
```

or an appropriate performance counter.

Use bytes internally and convert only when preparing output.

Use:

```text
1 MiB = 1,048,576 bytes
1 GiB = 1,073,741,824 bytes
```

Field names may retain `MB` and `GB` for readability, but calculations must consistently use binary units.

Calculate:

```text
UsedMemoryMB = InstalledMemoryMB - AvailableMemoryMB
UsedMemoryPercent = UsedMemoryMB / InstalledMemoryMB × 100
CommitPercent = CommittedMemoryMB / CommitLimitMB × 100
```

Handle divide-by-zero safely.

Round percentages to two decimal places.

Round byte-derived MB and GB fields to two decimal places.

## Performance counter handling

Performance counters may be localised, missing, or temporarily unavailable.

The collector must not fail completely because one counter cannot be read.

Use a defensive design:

1. Try native PowerShell or CIM classes first where practical.
2. Use `Get-Counter` only for counters that meaningfully improve the report.
3. Collect counters in a small number of calls rather than repeatedly.
4. Catch counter-specific failures.
5. Set unavailable fields to blank.
6. Increment warning or error counters.
7. Write diagnostic details separately.

Where a rate counter requires two samples, use a short sample interval such as one second and no more than two samples.

Avoid making every five-minute run unnecessarily slow.

Do not depend entirely on English performance-counter names where a more robust CIM source exists.

## Process telemetry

Create a process CSV with at least the following fields:

```text
SchemaVersion
CollectorVersion
SampleId
TimestampLocal
TimestampUtc
ComputerName
ProcessName
ProcessId
ParentProcessId
ExecutablePath
CommandLineCategory
SessionId
UserName
WorkingSetMB
PrivateMemoryMB
PagedMemoryMB
VirtualMemoryMB
PeakWorkingSetMB
HandleCount
ThreadCount
TotalProcessorTimeSeconds
StartTimeLocal
AgeSeconds
Responding
Is64Bit
SelectionReason
```

Collect only the configured number of top processes, default 20.

Select processes primarily by private memory usage.

Also include any process that appears in the top list by working set, even when this causes a small number of additional rows.

Set `SelectionReason` to values such as:

```text
TopPrivateMemory
TopWorkingSet
Both
```

Do not fail because a process exits while being queried.

Process enumeration is inherently racy.

Wrap per-process access in individual `try/catch` blocks.

Access-denied errors must be treated as expected.

Do not collect full command lines into the CSV.

Instead, set `CommandLineCategory` to a safe classification such as:

```text
Browser
Development
PowerShell
WSL
Docker
VirtualMachine
MicrosoftOffice
Security
System
Other
```

Avoid collecting sensitive document names, URLs, shell commands, arguments, or credentials.

`ExecutablePath` may be recorded where accessible, but provide a configuration switch to disable it.

Default to recording executable paths because the collector runs locally for an authorised diagnostic purpose.

Do not record window titles.

## User attribution

Where possible, identify the user account that owns a selected process.

Use an efficient implementation.

Do not execute one slow WMI query for every process if a bulk CIM query can provide process owners or process metadata more efficiently.

Where ownership cannot be resolved, leave the field blank.

Do not let process-owner resolution significantly increase collection time.

## Parent process identifiers

Use `Win32_Process` or another appropriate source to map process IDs to parent process IDs.

Avoid executing a separate CIM query per process.

Build lookup tables from one bulk query.

## CPU metrics

For total system CPU usage, use a reliable source such as:

```text
Win32_PerfFormattedData_PerfOS_Processor
```

or an appropriate performance counter.

Do not attempt to calculate instantaneous per-process CPU percentages from cumulative `CPU` values unless taking two samples.

The process CSV should record cumulative:

```text
TotalProcessorTimeSeconds
```

This can be analysed over time later using consecutive samples.

Do not label cumulative CPU time as CPU percentage.

## Development workload counts

Collect only simple, reliable process counts.

Count known process names associated with:

```text
chrome
msedge
firefox
Cursor
Code
powershell
pwsh
WindowsTerminal
wsl
wslhost
vmmem
vmmemWSL
Docker Desktop
com.docker.backend
vmwp
vmware
VirtualBox
```

Normalise process names without `.exe`.

Keep the implementation data-driven using arrays or hashtables rather than repetitive code.

Do not attempt fragile browser tab detection.

Do not inspect browser databases, window titles, or browser automation APIs.

Do not call `docker.exe`, `wsl.exe`, Hyper-V cmdlets, VMware tools, or VirtualBox tools during routine collection unless explicitly enabled.

The basic collector should remain lightweight and reliable.

## Battery and power

Where available, collect:

```text
PowerSource
BatteryChargePercent
BatteryStatus
```

Use CIM or built-in Windows APIs.

Correctly handle desktop computers or laptops where battery information is unavailable.

Suggested `PowerSource` values:

```text
AC
Battery
Unknown
NotApplicable
```

## Memory compression

Collect Windows memory compression where it can be done reliably and cheaply.

Potential sources include:

```powershell
Get-MMAgent
Get-Process -Name Memory Compression
```

However, do not assume `Get-MMAgent` exposes current compressed-memory size.

If a reliable current value cannot be collected, leave `MemoryCompressedMB` blank.

Do not invent or infer a value from unrelated counters.

## System totals

Collect process, thread, and handle totals efficiently.

Possible sources include:

```text
Win32_PerfFormattedData_PerfOS_System
Get-Process
```

Use a single process enumeration where possible and reuse it for counts and top-process selection.

## Disk metrics

Collect lightweight system-drive and aggregate disk information:

```text
SystemDriveFreeGB
SystemDriveFreePercent
DiskQueueLength
DiskReadBytesPerSec
DiskWriteBytesPerSec
```

Do not enumerate every file or directory.

Do not perform storage-health scans.

Do not use SMART queries during the regular five-minute sample.

## Logging and diagnostics

Implement a diagnostic log separate from the telemetry CSV.

Example:

```text
diagnostics\collector-2026-07.log
```

Use monthly diagnostic logs.

Each diagnostic entry should include:

```text
timestamp
severity
event
message
exception type
script version
```

Supported severities:

```text
DEBUG
INFO
WARNING
ERROR
FATAL
```

Avoid excessive diagnostic output during normal successful runs.

One successful collection should not require dozens of log entries.

Log important lifecycle events such as:

```text
InstallStarted
InstallCompleted
TaskRegistered
TaskValidationFailed
CollectionStarted
CollectionCompleted
CollectionPartial
CollectionFailed
UninstallStarted
UninstallCompleted
ConfigInvalid
CsvHeaderMismatch
ArchiveCreated
RetentionCleanup
```

Do not write credentials, environment-variable values, command-line contents, browser URLs, or document names into diagnostic logs.

## Error handling

Set:

```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'
```

Where appropriate, override individual commands with controlled error handling.

Use top-level:

```powershell
try {
}
catch {
}
finally {
}
```

Use structured internal functions.

Do not place the entire application in one giant block.

Use meaningful exception messages.

Expected transient errors should not generate an unreadable full stack trace unless diagnostic detail is useful.

Fatal errors must:

* be written to diagnostics where possible;
* be displayed interactively when applicable;
* return a nonzero exit code.

## Exit codes

Define constants or an enum-like hashtable.

Use consistent exit codes, for example:

```text
0   Success
1   General failure
2   Invalid arguments
3   Administrator privileges required
4   Installation validation failure
5   Scheduled Task operation failure
6   Configuration failure
7   Collection failure
8   CSV write failure
9   Lock already held
10  Purge safety validation failure
```

Document the exit codes in `-Help`.

A lock already held may optionally return `0` if treated as an expected skipped run, but the behaviour must be documented and consistent.

## Atomic writes

Use atomic-write techniques for:

* `config.json`
* `install.json`
* generated manifests
* export ZIP files

Write to a temporary file in the same directory, flush it, then move or replace it.

CSV append operations cannot be fully atomic, so:

* hold the collector mutex;
* build the entire row in memory;
* serialise the row before opening the file;
* append in one operation;
* close the file promptly.

## CSV writing implementation

Avoid repeatedly using `Export-Csv -Append` without validating headers.

Create a reusable CSV append function that:

1. Accepts a strongly ordered object.
2. Determines expected column order.
3. Creates the file with the header if absent.
4. Validates the header if present.
5. Serialises exactly one or more rows.
6. Appends using UTF-8.
7. correctly escapes commas, quotes, and new lines.

Use `[ordered]` dictionaries to preserve column order.

Ensure PowerShell 7 and Windows PowerShell 5.1 encoding differences are handled.

Prefer UTF-8 without BOM where practical, but consistency and Excel compatibility are more important.

## Retention and archive handling

Keep active daily CSV files for the configured retention period.

On collection runs, perform retention maintenance no more than once per day.

Use a maintenance state file or inspect an existing marker.

Do not scan and compress logs every five minutes.

Recommended behaviour:

* logs newer than `RetentionDays`: retain;
* logs older than `RetentionDays`: move into a ZIP archive by month;
* diagnostics may use a separate retention period or follow the same default.

Archive names:

```text
archive\telemetry-2026-07.zip
```

Avoid adding the same file to an archive repeatedly.

After validating that a file exists in the ZIP archive, delete the original old CSV.

If ZIP update or validation fails, keep the original CSV.

Use built-in .NET compression APIs or `Compress-Archive`.

Do not require 7-Zip.

## Export command

The `-Export` command must create a ZIP file suitable for submitting for analysis.

Example output:

```text
exports\WorkstationTelemetry-PC14250-20260805-143500.zip
```

Include:

```text
logs\
config.json
install.json
diagnostics\
export_manifest.json
```

Do not include the installed script unless useful.

The export manifest must include:

```text
exported_at
computer_name
collector_version
date_range
system_csv_count
process_csv_count
total_sample_count
total_process_row_count
files
sha256 hashes
```

Calculate SHA-256 hashes using `Get-FileHash`.

Do not include files outside the installation directory.

Validate archive creation before reporting success.

Do not delete source logs after export.

## Status command

The `-Status` command must report:

```text
Installed: Yes/No
Install path
Installed script version
Current source script version
Scheduled Task present
Scheduled Task state
Scheduled Task enabled
Last run time
Last task result
Next run time
Configured interval
Log directory
Latest system CSV
Latest process CSV
Latest successful sample time
Age of latest sample
Total log size
System CSV count
Process CSV count
Oldest sample date
Newest sample date
Current configuration validity
```

Use readable console formatting.

Return nonzero only when an actual error prevents status determination.

A missing installation should be reported clearly, not as an unhandled exception.

## Validate command

The `-Validate` command must test:

```text
Operating system
PowerShell version
Administrative state
Installation directory
Installed script
Configuration JSON
Installation metadata
Scheduled Task presence
Scheduled Task action path
Scheduled Task arguments
Scheduled Task principal
Scheduled Task triggers
Scheduled Task interval
Write access
CSV header schema
Latest sample age
Available disk space
Mutex availability
```

Display:

```text
PASS
WARNING
FAIL
```

for each check.

Provide an overall result.

Do not change system state during validation except for harmless temporary-file write testing within the installation directory.

Clean up temporary validation files.

## ShowConfig command

The `-ShowConfig` command should:

* locate the active configuration;
* validate it;
* print it in readable JSON;
* show defaults for missing optional properties;
* display the effective configuration after defaults are applied.

Do not expose unrelated environment information.

## Help command

Provide built-in help that includes:

* purpose;
* syntax;
* examples;
* administrative requirements;
* installed paths;
* task behaviour;
* data collected;
* privacy notes;
* uninstall behaviour;
* purge warning;
* exit codes.

Also include standard comment-based help at the top of the script:

```powershell
<#
.SYNOPSIS
.DESCRIPTION
.PARAMETER Install
.EXAMPLE
.NOTES
#>
```

## Privacy requirements

The collector is intended to measure resource demand, not employee activity.

Do not collect:

* keystrokes;
* clipboard contents;
* browser history;
* browser URLs;
* browser tab titles;
* window titles;
* email subjects;
* document names;
* shell command lines;
* PowerShell history;
* file-access history;
* network destinations;
* DNS queries;
* screenshots;
* microphone or camera data;
* precise location;
* passwords;
* tokens;
* cookies;
* authentication material.

Process names, numeric resource consumption, process identifiers, executable paths, process ownership, and broad workload categories are acceptable.

Include a concise privacy statement in `-Help`.

## Security requirements

* Never invoke downloaded code.
* Never weaken system security settings globally.
* Do not modify PowerShell execution policy permanently.
* Use `-ExecutionPolicy Bypass` only in the Scheduled Task action.
* Quote all paths.
* Handle spaces safely.
* Avoid `Invoke-Expression`.
* Avoid constructing executable command strings unnecessarily.
* Avoid `cmd.exe` unless absolutely required for delayed cleanup.
* Use `Start-Process` with explicit argument lists where possible.
* Do not create writable executable paths for ordinary users.
* Restrict permissions on the installation directory where practical.
* Ensure ordinary users can read exported logs only where appropriate.
* Do not grant broad `Everyone:FullControl` permissions.
* Treat configuration and installation metadata as untrusted input and validate them.

## Functions

Organise the implementation into clear functions.

Suggested function names:

```powershell
Get-IsAdministrator
Restart-ScriptElevated
Get-DefaultConfiguration
Get-EffectiveConfiguration
Read-Configuration
Write-AtomicJson
Write-DiagnosticLog
Get-InstallMetadata
Resolve-PowerShellExecutable
Install-Collector
Uninstall-Collector
Remove-CollectorSafely
Register-CollectorScheduledTask
Remove-CollectorScheduledTask
Get-CollectorTask
Test-CollectorTask
Invoke-Collection
Get-SystemTelemetry
Get-ProcessTelemetry
Get-PowerTelemetry
Get-PerformanceTelemetry
Get-DevelopmentProcessCounts
Append-CsvRows
Test-CsvHeader
Invoke-RetentionMaintenance
Export-CollectorData
Get-CollectorStatus
Test-CollectorInstallation
Show-CollectorHelp
Acquire-CollectorMutex
Release-CollectorMutex
ConvertTo-InvariantValue
Get-SafeFileName
Test-SafePurgePath
```

These names are suggestions, not mandatory, but the final script must remain modular and readable.

## Compatibility

Target PowerShell 7 first.

Retain compatibility with Windows PowerShell 5.1 where reasonably possible.

Avoid syntax that unnecessarily prevents PowerShell 5.1 execution.

Where a feature differs between versions:

* detect `$PSVersionTable.PSVersion`;
* use a compatible fallback;
* document any unavoidable limitation.

Do not use PowerShell classes unless they significantly improve reliability and remain compatible.

## Performance requirements

A normal collection run should:

* consume little CPU;
* use little memory;
* produce minimal disk I/O;
* normally complete within ten seconds;
* avoid launching many subprocesses;
* avoid querying expensive providers repeatedly;
* perform one bulk process query rather than many per-process queries;
* reuse collected data across calculations.

Do not use `Measure-Command` as the main collection mechanism.

Use a `Stopwatch` for collection duration.

## Testability

Include an internal `-RunOnce` mode that writes real telemetry.

Also provide an optional:

```powershell
-DryRun
```

parameter where practical.

For installation dry run, display:

* destination paths;
* task name;
* executable;
* action arguments;
* triggers;
* settings;
* configuration;

without making changes.

For uninstall dry run, display what would be stopped, unregistered, retained, or removed.

Do not fabricate successful validation results during dry run.

## Required tests

Before considering the implementation complete, test or reason through all of the following:

1. Script path contains spaces.
2. Install path contains spaces.
3. PowerShell 7 is installed.
4. PowerShell 7 is absent.
5. Running install without elevation.
6. Fresh installation.
7. Reinstallation over an existing installation.
8. Existing logs are preserved during reinstall.
9. Existing task has incorrect action arguments.
10. Existing task uses an obsolete script path.
11. Config file is missing.
12. Config file contains invalid JSON.
13. Config file contains out-of-range values.
14. A process terminates during collection.
15. Access to a process is denied.
16. Performance counters are unavailable.
17. Battery CIM class is unavailable.
18. Daily CSV already exists.
19. Daily CSV is empty.
20. Daily CSV has a wrong header.
21. Two collectors start simultaneously.
22. Log directory is temporarily unavailable.
23. Disk is nearly full.
24. Normal uninstall.
25. Purge uninstall.
26. Purge path safety validation fails.
27. Export with no telemetry files.
28. Export with several weeks of telemetry.
29. Scheduled Task missed a run while asleep.
30. Task is triggered while another instance is running.
31. Computer name or username contains punctuation.
32. Collection crosses midnight.
33. Daylight-saving transition.
34. Computer resumes from sleep.
35. System clock changes.
36. CSV files are opened in Excel while collection runs.
37. Script is executed from a network share.
38. Installed copy is already the current script.
39. Scheduled Task registration partially fails.
40. A stale temporary file exists after a previous crash.

## Code quality

The final script must be:

* complete;
* runnable;
* fully implemented;
* unabridged;
* syntactically valid;
* consistently formatted;
* commented where logic is non-obvious;
* free of placeholder functions;
* free of TODO markers;
* free of pseudocode;
* free of omitted sections;
* free of fake sample output presented as real collection data.

Do not respond with an outline.

Do not provide fragments.

Do not say that a section is left for the user to implement.

Provide the entire script in one code block.

## Response requirements

Your response must contain:

1. A short architecture summary.
2. The complete `workstation_telemetry_collector.ps1` script.
3. Exact installation command.
4. Exact status command.
5. Exact validation command.
6. Exact manual collection command.
7. Exact export command.
8. Exact normal uninstall command.
9. Exact purge uninstall command.
10. A short explanation of the generated CSV files.
11. Known limitations, only where genuine.

Use complete one-line PowerShell commands.

Do not split shell commands across lines with backslashes or PowerShell continuation characters.

Do not abbreviate the script.

Do not omit any code because of length.

Prioritise reliability, safe failure, low overhead, predictable CSV schemas, and clean unattended execution.

This prompt directs Codex to produce the complete script rather than an architectural outline or partial implementation.
