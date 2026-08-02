# ESSL Biometric Attendance Dashboard

> **Live biometric punch data → real-time attendance dashboard + automated reports. Direct SQL Server connection. Zero manual exports.**

A production Node.js dashboard that connects directly to the ESSL iClock SQL Server (`eBioserver` database), exposes a REST API over the raw punch data, and delivers real-time attendance tracking, status classification, and one-click Excel/CSV exports across all devices and locations.

Built for a multi-outlet entertainment chain's HO and store operations — covering biometric readers across branches, with department-level drill-down for head office.

---

## The problem it replaced

ESSL's built-in reporting requires logging into the iClock web interface, selecting date ranges, and downloading one report at a time — one device at a time, one format at a time. For a chain with multiple locations:

- No cross-device unified view for a given date
- No automated P/H/S/A (Present / Half-day / Short / Absent) classification against business rules
- No daily digest for HR without someone manually pulling and formatting data
- No latecomer identification with a configured grace-period cutoff
- No department-level breakdown for HO roster management

This dashboard replaces all of that with a browser tab.

---

## What it does

### Real-time attendance view (`/api/summary`)

For any date, shows every employee's:

| Field | Source |
|---|---|
| Check-in time | Earliest punch of the day |
| Check-out time | Latest punch of the day (if >1 punch) |
| Duration | Hours between first and last punch |
| **Status** | Classified against business rules |
| Late flag | Check-in after 09:30 |
| Device / Location | Resolved from `Devices → Locations` join |

**Status classification:**
- `P` (Present) — ≥ 7 hours worked
- `H` (Half-day) — ≥ 4 hours, < 7 hours
- `S` (Short) — punched in, < 4 hours
- `A` (Absent) — no punch recorded

Filterable by device ID or location name. KPI summary counts P / H / S / A / Late at the top.

### Live punch feed (`/api/live`)

Rolling view of the last 80 punches across all readers — employee name, location, time, and IN/OUT direction resolved from `AttDirectionCode`. Updates on demand.

### Multi-date range export (`/api/export`)

CSV export covering any date range with all the enriched fields — date, code, name, department, location, check-in, check-out, duration, hours, status, and punch count. BOM-prefixed for direct Excel open without encoding issues.

### Excel report with formula-linked sheets

Monthly Excel exports structured as:
- **Summary tab** — all employees, all days, full attendance grid
- **HO department tab** — Kolkata head office only, with hardcoded dept roster so absent employees still appear (not just those who punched)
- **Per-department sheets** — auto-generated from department field, formula-linked for month-end HR processing

### Device inventory (`/api/devices`)

Lists all attendance-capable biometric readers with their resolved location names from the `Devices → Locations` schema join, with graceful fallback through three query patterns for different iClock schema versions.

---

## Engineering decisions

### Direct SQL Server connection — why not the iClock REST API

The ESSL iClock web API is documented but rate-limited, session-based, and returns pre-aggregated data. Direct SQL gives:
- Sub-second query response on `DeviceLogs` with the right indexes
- Raw punch access — every individual tap, not just aggregated summaries
- Full `Employees` and `Devices` schema access for enrichment
- No session management, no rate limits, no API versioning drift

### The LogDate timezone trap

`eBioServer` stores `LogDate` as a plain IST wall-clock datetime with no timezone metadata. MSSQL reads it back tagged as UTC. This means:

- **Never apply +05:30 offset** — that would shift every timestamp by 5.5 hours and misalign day boundaries, bleeding punches from one calendar day into the next
- Day boundaries are taken literally at `00:00–24:00` with no offset
- `new Date(dt).toISOString().slice(11,16)` gives the correct IST time directly

This was a non-obvious data integrity issue: applying standard UTC→IST conversion looked correct in tests but produced midnight-to-5:30am punches appearing on the wrong date in production.

### ESSL server login failure — root cause diagnosis

The iClock web server went down completely with no login available — affecting attendance sync for the full company. Diagnosed root cause: the SQL Server `essl` service account had a Windows password auto-expiry policy applied. When the password expired, the IIS application pool lost database connectivity and the web UI returned a blank login failure.

Fix:
1. Connected to SQL Server via SSMS with a separate admin account
2. Reset the `essl` service account password
3. Ran `iisreset` to restart the IIS application pool
4. Disabled password-expiry policy on the `essl` service account to prevent recurrence

Attendance sync restored in under 10 minutes. No data was lost — the biometric devices buffer locally and upload when connectivity resumes.

### GratyHR Astra integration

Biometric punch data is synced from eBioserver into **GratyHR Astra** HR software via API:
- Employee provisioning: new employees registered in ESSL are auto-provisioned into GratyHR
- Monthly attendance reports: attendance summary pushed to GratyHR for payroll processing
- Eliminates the double-entry of attendance data between the biometric system and the HR platform

### Device-to-location resolution

ESSL's `Devices` table has a `LocationId` foreign key → `Locations.Id` → `Locations.Description`. This join is not present in all iClock versions. The server tries three fallback query patterns in order:
1. Full `Devices → Locations` join (preferred — returns named locations)
2. `DeviceName` from `Devices` table only
3. Raw `DeviceId` from `DeviceLogs` as last resort

This makes the dashboard resilient across different ESSL firmware versions and schema variants.

---

## API reference

| Endpoint | Params | Returns |
|---|---|---|
| `GET /api/summary` | `date`, `dev`, `loc` | Daily attendance with KPI counts |
| `GET /api/live` | `dev`, `loc` | Last 80 punches live |
| `GET /api/devices` | — | All attendance devices + locations |
| `GET /api/export` | `from`, `to`, `dev`, `loc` | CSV for date range |
| `GET /api/excel` | `from`, `to`, `dev`, `loc` | Excel with department sheets |

All `date` / `from` / `to` params are `YYYY-MM-DD`. `dev` filters by device ID, `loc` by location name (matched via `Locations.Description`).

---

## Business rules

| Rule | Value |
|---|---|
| Full day threshold | ≥ 7 hours between first and last punch |
| Half-day threshold | ≥ 4 hours |
| Late cutoff | Check-in after 09:30 |
| WO pattern (CCTV/SOP roles) | Rotational — not fixed Sunday |

---

## Stack

`Node.js` · `Express` · `mssql (SQL Server)` · `ESSL eBioserver DB` · `GratyHR Astra API` · `Excel COM`

---

## Context

Part of the broader HR + operations automation stack built for a 25+-outlet entertainment chain. The attendance data feeds into:
- Monthly salary processing (via GratyHR Astra)
- Store SOP compliance scoring (rotational WO tracking)
- CCTV staffing verification (punch-in vs CCTV headcount cross-check)
