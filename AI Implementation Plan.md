# AI Implementation Plan: Network Connection Security Analyzer

## Project Summary

Build a PowerShell security script that identifies potentially suspicious network activity by analyzing established TCP connections, verifying process digital signatures, and enriching unsigned binary connections with IP geolocation data.

---

## Prompt Engineering Strategy

Each phase below includes a **structured AI prompt** designed to generate production-quality PowerShell code following the project's rules and constraints.

---

## Phase 1: Get Network Connections

**Objective:** Capture and parse all established TCP connections from `netstat -ano`.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write a PowerShell function `Get-EstablishedTcpConnections` that:
>
> 1. Runs `netstat -ano` and captures the output
> 2. Filters for ESTABLISHED TCP connections only
> 3. Parses each line using regex to extract: Local Address, Local Port, Foreign Address, Foreign Port, State, and PID
> 4. Returns a collection of `[PSCustomObject]` with properties: `LocalAddress`, `LocalPort`, `ForeignAddress`, `ForeignPort`, `State`, `ProcessId`
> 5. Excludes loopback addresses (127.0.0.1, ::1)
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Prefer .NET assemblies over cmdlets where possible (e.g., `[System.Diagnostics.Process]`)
> - Do not use `$PID` as a variable name (it is a built-in read-only variable)
> - Do not place variable names adjacent to a colon in output; use `"$($Variable):"` syntax
> - Do not use special characters or terminal icons in console output
> - Include progress indicators for long-running operations

**Expected Output:** A function returning `[PSCustomObject[]]` with parsed connection data.

**Validation Criteria:**
- [ ] Correctly parses both IPv4 and IPv6 netstat output
- [ ] Filters to ESTABLISHED state only
- [ ] Handles empty netstat output gracefully
- [ ] Runs without error on PS 5.1 and PS 7+

---

## Phase 2: Process Analysis Loop

**Objective:** Resolve each unique PID to its process details and executable path.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write a PowerShell function `Get-ProcessDetails` that accepts an array of Process IDs (PIDs) and:
>
> 1. Deduplicates the PID list
> 2. For each unique PID, retrieves the process object using .NET `[System.Diagnostics.Process]::GetProcessById()`
> 3. Extracts the full executable path via the process `MainModule.FileName` property
> 4. Handles processes that have already exited (skip with warning)
> 5. Handles system processes (PID 0, PID 4) where executable paths are not accessible (skip with alert)
> 6. Handles access-denied errors for protected processes (alert and skip)
> 7. Returns `[PSCustomObject[]]` with: `ProcessId`, `ProcessName`, `ExecutablePath`, `IsAccessible`
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Prefer .NET assemblies over cmdlets
> - Do not use `$PID` as a variable name
> - If a value is missing, alert that you don't have it and skip
> - Add progress indicator showing "Processing PID X of Y"

**Expected Output:** A function returning process metadata for each accessible PID.

**Validation Criteria:**
- [ ] Gracefully handles exited processes
- [ ] Skips system/protected processes with informative warnings
- [ ] Returns accurate executable paths
- [ ] No unhandled exceptions for any PID value

---

## Phase 3: Signature Verification

**Objective:** Check each executable's Authenticode digital signature and flag unsigned/invalid binaries.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write a PowerShell function `Test-BinarySignature` that accepts an array of `[PSCustomObject]` (with `ProcessId`, `ProcessName`, `ExecutablePath`) and:
>
> 1. For each object where `ExecutablePath` is not null/empty, runs `Get-AuthenticodeSignature`
> 2. Evaluates the signature status: Valid, NotSigned, HashMismatch, NotTrusted, UnknownError
> 3. Extracts signer information (Subject, Issuer) when available
> 4. Flags binaries where status is NOT "Valid" as suspicious
> 5. Returns `[PSCustomObject[]]` with: `ProcessId`, `ProcessName`, `ExecutablePath`, `SignatureStatus`, `Signer`, `Issuer`, `IsSigned` (boolean)
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Prefer .NET assemblies over cmdlets where possible
> - Do not use `$PID` as a variable name
> - Do not place variable names adjacent to a colon in output; use `"$($Variable):"` syntax
> - Cache results so the same executable path is not checked twice

**Expected Output:** A function returning signature verification results per binary.

**Validation Criteria:**
- [ ] Correctly identifies signed vs unsigned binaries
- [ ] Handles missing or inaccessible paths
- [ ] Caches duplicate path checks
- [ ] Returns signer details for signed binaries

---

## Phase 4: IP Geolocation Enrichment (Unsigned Binaries Only)

**Objective:** For processes with unsigned/invalid signatures, query geolocation data for their destination IPs.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write a PowerShell function `Get-IpGeolocation` that:
>
> 1. Accepts an array of Foreign IP addresses
> 2. Deduplicates the IP list
> 3. Skips private/reserved IP ranges (10.x, 172.16-31.x, 192.168.x, link-local)
> 4. For each public IP, queries `ipinfo.io/{ip}/json` using `[System.Net.Http.HttpClient]` (.NET)
> 5. Parses the JSON response to extract: IP, Country, Region, City, ISP/Org, Hostname
> 6. Implements a local cache (`[System.Collections.Generic.Dictionary]`) to avoid duplicate lookups
> 7. Handles rate limiting (HTTP 429) with exponential backoff retry (max 3 retries)
> 8. Handles network errors and timeouts gracefully
> 9. Returns `[PSCustomObject[]]` with: `IpAddress`, `Country`, `Region`, `City`, `Organization`, `Hostname`
>
> **Important:** Do NOT upload any local data to external sources without confirmation. The only external call is the IP lookup against a public API using the foreign (remote) IP addresses already visible in netstat output. Prompt the user for approval before making any external API calls.
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Prefer .NET assemblies (`[System.Net.Http.HttpClient]`) over `Invoke-RestMethod`
> - Do not use `$PID` as a variable name
> - Do not use special characters or terminal icons in console output
> - Add progress indicator showing "Querying IP X of Y"

**Expected Output:** A function returning geolocation data per unique public IP.

**Validation Criteria:**
- [ ] Skips private/reserved IPs
- [ ] Caches duplicate IP lookups
- [ ] Handles 429 rate limits with retry
- [ ] Prompts user before external API calls
- [ ] Handles network failures gracefully

---

## Phase 5: Comprehensive Process Details

**Objective:** Gather deep forensic details for each unsigned binary process.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write a PowerShell function `Get-ForensicProcessInfo` that accepts a `ProcessId` and `ExecutablePath` and returns:
>
> 1. **Parent Process:** Use `Get-CimInstance Win32_Process -Filter "ProcessId = $procId"` to get `ParentProcessId`, then resolve the parent process name
> 2. **File Timestamps:** Use `[System.IO.FileInfo]` to get CreationTime, LastWriteTime, LastAccessTime
> 3. **Process Runtime Details:** Start time, running user/owner, command line arguments (from CIM/WMI)
> 4. **File Hash:** Compute SHA256 hash using `[System.Security.Cryptography.SHA256]::Create()` and `[System.IO.File]::OpenRead()`
> 5. **Connection Count:** Accept a connection count parameter to include in output
>
> Return a `[PSCustomObject]` with all the above fields.
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Prefer .NET assemblies over cmdlets where possible
> - Do not use `$PID` as a variable name
> - If certain values are missing, alert that you don't have them and skip
> - Do not place variable names adjacent to a colon in output; use `"$($Variable):"` syntax

**Expected Output:** A function returning a rich forensic object per process.

**Validation Criteria:**
- [ ] Correctly resolves parent process chain
- [ ] Computes valid SHA256 hash
- [ ] Handles missing/exited processes
- [ ] Returns accurate file timestamps
- [ ] Works with both CIM (PS 7+) and WMI fallback (PS 5.1)

---

## Phase 6: Output Formatting and Export

**Objective:** Present results in a clear, color-coded format with export capabilities.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write a PowerShell function `Show-SecurityReport` that:
>
> 1. Accepts the combined results from all previous phases
> 2. Displays a **Summary Table** first showing all processes with columns: PID, Process Name, Signature Status, Connection Count, Foreign IPs
> 3. Color-codes output: Red (`[ConsoleColor]::Red`) for unsigned/invalid, Green (`[ConsoleColor]::Green`) for signed/valid
> 4. Below the summary, displays **Detailed Blocks** for each unsigned binary including:
>    - All forensic details (parent process, timestamps, hash, command line)
>    - Associated IP geolocation data in a sub-table
> 5. Groups results by process
> 6. Includes a function `Export-SecurityReport` that exports the full dataset to:
>    - CSV file (flat structure for spreadsheet analysis)
>    - JSON file (nested structure preserving relationships)
>    - Accepts an `-OutputPath` parameter for the export directory
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Use `[System.IO.File]::WriteAllText()` for file export instead of `Out-File`
> - Do not use `$PID` as a variable name
> - Do not use special characters or terminal icons in console output
> - Do not place variable names adjacent to a colon in output; use `"$($Variable):"` syntax

**Expected Output:** Functions for console display and file export.

**Validation Criteria:**
- [ ] Summary table is readable and properly aligned
- [ ] Color coding works on both PS 5.1 and PS 7+
- [ ] CSV export is flat and importable in Excel
- [ ] JSON export preserves data hierarchy
- [ ] Export handles special characters in paths

---

## Phase 7: Main Orchestrator Script

**Objective:** Wire all phases together into a single entry-point script.

**AI Prompt:**

> You are a PowerShell IT Security expert in the style of Don Jones or Doug Finke, Microsoft PowerShell MVP.
>
> Write the main orchestrator function `Invoke-NetworkSecurityAudit` that:
>
> 1. Checks for Administrator privileges; warns if not elevated (some data may be limited)
> 2. Displays a startup banner with script name, version, and date (no special characters/icons)
> 3. Calls each phase function in sequence:
>    - `Get-EstablishedTcpConnections` -> capture connections
>    - `Get-ProcessDetails` -> resolve PIDs to processes
>    - `Test-BinarySignature` -> verify signatures
>    - `Get-IpGeolocation` -> enrich unsigned binary IPs (with user approval prompt)
>    - `Get-ForensicProcessInfo` -> gather deep details for unsigned binaries
>    - `Show-SecurityReport` -> display results
> 4. Accepts parameters:
>    - `-ExportPath [string]` - optional directory to export CSV/JSON
>    - `-SkipGeoLocation [switch]` - skip IP lookups
>    - `-IncludeSignedDetails [switch]` - show full details for signed binaries too
> 5. Shows overall progress "Phase X of 6: [Phase Name]"
> 6. Reports total execution time at the end
>
> Wrap the entire script as a single `.ps1` file with all functions defined within it, using a module-like structure with a `#region` / `#endregion` block per function.
>
> **Constraints:**
> - Use `Try {} Catch {} Finally {}` for error handling throughout
> - Must work on both PowerShell Core and Windows PowerShell v5.1
> - Prefer .NET assemblies over cmdlets where possible
> - Do not use `$PID` as a variable name
> - Do not expose local data or upload data externally without user approval
> - Do not use special characters or terminal icons in console output
> - Do not place variable names adjacent to a colon in output; use `"$($Variable):"` syntax
> - Incorporate best performance practices

**Expected Output:** A complete, self-contained `.ps1` script.

**Validation Criteria:**
- [ ] Runs end-to-end without error on PS 5.1
- [ ] Runs end-to-end without error on PS 7+
- [ ] Produces correct summary and detail output
- [ ] Export files are valid CSV/JSON
- [ ] Gracefully handles non-admin execution
- [ ] Prompts before any external API call

---

## Implementation Sequence

| Order | Phase | Dependency | Estimated Complexity |
|-------|-------|-----------|---------------------|
| 1 | Phase 1: Get Network Connections | None | Low |
| 2 | Phase 2: Process Analysis Loop | Phase 1 output | Low |
| 3 | Phase 3: Signature Verification | Phase 2 output | Medium |
| 4 | Phase 4: IP Geolocation | Phase 1 + Phase 3 output | Medium |
| 5 | Phase 5: Forensic Process Details | Phase 2 + Phase 3 output | Medium |
| 6 | Phase 6: Output and Export | All phases | Medium |
| 7 | Phase 7: Main Orchestrator | All phases | Low |

## Testing Strategy

| Test | Description |
|------|-------------|
| **Unit** | Run each function independently with mock data |
| **Integration** | Run full pipeline on a live system |
| **Cross-Platform** | Validate on Windows PowerShell 5.1 AND PowerShell 7+ |
| **Edge Cases** | Test with no established connections, all signed binaries, network offline |
| **Permissions** | Test as non-admin to verify graceful degradation |
| **Export** | Verify CSV opens in Excel, JSON parses with `ConvertFrom-Json` |

## Security Considerations

- **No data exfiltration:** The script only queries public IP geolocation. No local data is uploaded.
- **User consent:** External API calls require explicit user approval at runtime.
- **Read-only:** The script does not modify any system state, files, or configurations.
- **No credential handling:** No passwords, tokens, or secrets are stored or transmitted.

## File Structure

```
NetworkSecurityAudit/
    Invoke-NetworkSecurityAudit.ps1    # Main script (all-in-one)
    README.md                           # Usage documentation
    examples/
        sample-output.json              # Example JSON export
        sample-output.csv               # Example CSV export
```
