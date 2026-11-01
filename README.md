# ServiceNowScheduledExport

<img width="1648" height="138" alt="image" src="https://github.com/user-attachments/assets/fbc8da03-d7b7-42cf-a818-0580b475fbed" />

<img width="267" height="35" alt="image" src="https://github.com/user-attachments/assets/9c2344c8-8e40-4cfe-ab1b-0d1a450646f0" />
Execute Now

Working notes + reference for **ServiceNow Export Sets** (System Export Sets → Scheduled Data Exports):
pushing table rows to a file on a MID Server on a schedule, with delta exports, conditional runs and
pre/post scripts — and what to do with those files downstream (Power BI / Qlik).

**Contents**
0. [Build order — the pages to visit, in order](#a-build-order--the-pages-to-visit-in-order)
1. [The filename and its timestamp — read this first](#1-the-filename-and-its-timestamp--read-this-first)
2. [How the four records fit together](#2-how-the-four-records-fit-together)
3. [The live config in this instance (2026-11-01)](#3-the-live-config-in-this-instance-2026-11-01)
4. [Every configuration option, and where to set it](#4-every-configuration-option-and-where-to-set-it)
5. [Delta exports](#5-delta-exports)
6. [Scripts: conditional, pre-export, post-export](#6-scripts-conditional-pre-export-post-export)
7. [Recipes people actually use](#7-recipes-people-actually-use)
8. [Feeding Power BI / Qlik Sense — and saving space with deltas](#8-feeding-power-bi--qlik-sense--and-saving-space-with-deltas)
9. [Troubleshooting](#9-troubleshooting)
10. [Verifying any of this from the Table API](#10-verifying-any-of-this-from-the-table-api)

---

## A. Build order — the pages to visit, in order

Order matters: the **Export Set** points at an **Export Definition** and an **Export Target**, and the
**Scheduled Data Export** points at the set. Build bottom-up, and press **Execute Now** before trusting a schedule.
Each step below is one page to stand on, with what to have ready first, the fields to fill, and the one suggestion
that saves rework.

### Step 1 — Export Definition
> **Breadcrumb:** `All → System Export Sets → Export Definitions → New`

- **Prerequisite:** the source table exists and has rows (`u_torontoeventsfinal`, filled by
  [ServiceNowJDBC](https://github.com/eltonaguiar/ServiceNowJDBC)); role `export_set_admin`.
- **Fields to fill:** `Name` = `TorontoEventsFinal_ExportDef` · `Table` = `u_torontoeventsfinal` · `Fields` = the 12
  columns you want, **in output order** · leave the row filter empty for a whole-table extract.
- **Suggestion:** one definition per *file shape*, not per schedule. If a consumer needs different columns, clone the
  record — editing it in place leaves you a perfectly valid file with the wrong columns.
- **Checkpoint:** the field list reads back in the same order as the CSV header.

### Step 2 — Export Target
> **Breadcrumb:** `All → System Export Sets → Export Targets → New`

- **Prerequisite:** a MID Server that is **Validated: Yes** and is on the network that can see the destination
  ([ServiceNowMidServer](https://github.com/eltonaguiar/ServiceNowMidServer)); the folder must already exist under the
  MID's `agent\export\` and be writable.
- **Fields to fill:** `Name` = `ExportToWindows11Mid` · `Export target type` = *File on MID Server* ·
  `MID Server` = `win11-mid` · `Path` = `toronto/events` (relative to `agent\export\`, no leading slash).
- **Suggestion:** one target per consumer and per retention regime (`toronto/events/bi/`, `toronto/events/archive/`) —
  it is the cheapest way to give BI a clean folder it can watch.
- **Checkpoint:** `agent\export\toronto\events\` exists on the MID and the MID user can write to it.

### Step 3 — Export Set
> **Breadcrumb:** `All → System Export Sets → Create Export Set`

- **Prerequisite:** steps 1 and 2 finished.
- **Fields to fill:** `Name` = `TorontoEvents` · `Export definition` = step 1 · `Export target` = step 2 ·
  `File name` = `torontoevents.csv` · `Format` = CSV · `Append timestamp` = on · `Max rows` = `100000`.
- **Suggestion:** keep *Append timestamp* on. The cost is a file per run; the benefit is that a failed run cannot
  masquerade as a fresh one.
- **Checkpoint:** **Execute Now** produces a file.

### Step 4 — the format decisions on the same form
> **Breadcrumb:** same Export Set form, lower half

- **Prerequisite:** step 3 saved.
- **Fields to fill:** `Delimiter` = `,` · `Use display value` = on/off **deliberately** · `Max rows` (see
  [§4](#4-every-configuration-option-and-where-to-set-it) for every option's effect).
- **Suggestion:** machine consumers want raw values (`sys_id`, choice *values*); humans want display values. Decide
  which this file is for — mixing the two is what breaks joins downstream.
- **Checkpoint:** header = field names; cells = the format you chose.

### Step 5 — Scheduled Data Export
> **Breadcrumb:** `All → System Export Sets → Administration → Scheduled Exports → New`

- **Prerequisite:** a manual *Execute Now* has succeeded — never schedule something you have not seen work once.
- **Fields to fill:** `Name` = `ExportTorontoEventsFinal` · `Export set` = step 3 · `Run` = daily `08:00:00` ·
  `Run as` = a service account (not a person who will leave) · `Enable delta export` = on · `Delta field` =
  `sys_updated_on` · leave the conditional / pre / post scripts **empty** for now.
- **Suggestion:** add the scripts only after the plain schedule has run cleanly at least once, so you know which part
  failed; see [§6](#6-scripts-conditional-pre-export-post-export) for what each hook can and cannot do.
- **Checkpoint:** *Last export scheduled run* advances.

### Step 6 — Execute Now (smoke test)
> **Breadcrumb:** Export Set form → **Execute Now**

- **Prerequisite:** step 5 saved (the schedule carries the delta settings, the set carries the shape).
- **Fields to fill:** none.
- **Suggestion:** run it **twice**. The first run is a full extract; the second shows you what a delta looks like
  (usually a header-only file) before a schedule surprises you at 08:00.
- **Checkpoint:** file bigger than the header; history row `completed`.

### Step 7 — Export Log
> **Breadcrumb:** `All → System Export Sets → Export Log`

- **Prerequisite:** at least one run.
- **Fields to fill:** none — this is read-only.
- **Suggestion:** check here **first** whenever a file is missing: it names the MID and the transfer, so it separates
  "never generated" from "generated but never delivered".
- **Checkpoint:** `Transferred torontoevents.csv to the MID Server 'win11-mid'`.

Roles: `export_set_admin` to build all of the above, `export_set_scheduler` to own the schedule.

**The parameters people actually change, and what they cost**

| Change | Effect | Cost |
|---|---|---|
| *Max rows* up | one bigger file instead of silent truncation | instance render time + MID memory; above ~100 K rows use *Parallel export* |
| *Append timestamp* off | one file, overwritten each run | no per-run history; a failed run leaves yesterday's file looking current |
| *Use display value* on | human-readable cells | breaks joins in BI and shifts with language/ACL |
| Delta on | tiny files, only changed rows | you inherit a watermark and the "empty file" behaviour |
| Conditional on | no files when nothing changed | one more script to debug |
| Time zone (instance/user) | the filename stamp matches your clock | DST moves the offset twice a year |

**Validation, end to end** — these numbers are real on this instance (172 B empty delta, 2,128,595 B full):

```bash
# 1 is there a run, and how big was the payload?
curl -s -u "$SN_USER:$SN_PASS" -H 'Accept: application/json' \
  "https://<instance>.service-now.com/api/now/table/sys_export_set_run?sysparm_limit=3&sysparm_fields=number,state,sys_created_on,completed,attachment_size"

# 2 did the file land?
ls -l "/c/ServiceNow/MID/agent/export/toronto/events/"        # Windows: dir "C:\ServiceNow\MID\agent\export\toronto\events"

# 3 does the header match the export definition's field list?
head -1 torontoevents_*.csv

# 4 is the delta watermark moving?
curl -s -u "$SN_USER:$SN_PASS" -H 'Accept: application/json' \
  "https://<instance>.service-now.com/api/now/table/scheduled_data_export?sysparm_fields=name,last_export_run,enable_delta_export"
```

If (1) says `completed` but (2) shows nothing, the fault is the target/MID, not the export — see [§9](#9-troubleshooting).

---

## 1. The filename and its timestamp — read this first

```
torontoevents_20261101050335.csv
└─────┬─────┘ └──────┬───────┘
   File name        └─ yyyyMMddHHmmss  =  2026-11-01 05:03:35
   (export set)       appended because "Append timestamp" is checked
```

| Piece | Where it comes from |
|---|---|
| `torontoevents` | Export Set → **File name** = `torontoevents.csv` (the `.csv` extension is kept) |
| `_` | Separator ServiceNow inserts before the timestamp |
| `20261101` | Date `yyyyMMdd` — `2026-11-01` |
| `050335` | Time `HHmmss` in the **instance's time zone**, 24-hour, zero-padded |
| `.csv` | From the File name you typed (matching the `Format` field) |

So `torontoevents_20261101050335.csv` = *"torontoevents.csv, exported at 05:03:35 on 1 Nov 2026"*.
It is ISO-8601 **basic** format: no separators, no timezone, no AM/PM. Guessing `torontoevents_2026_Nov_01_752AM.CSV`
is a natural assumption but wrong — it is always `yyyyMMddHHmmss`, and the format is **not configurable** in the UI
(only *append* yes/no).

### Why the numbers look wrong (the real trap)

The suffix is rendered in the instance/user time zone, which is **not** necessarily UTC and not necessarily yours.
Proof from this instance, matching run history to the file on disk by time **and** byte size:

| `sys_export_set_run.sys_created_on` (instance = UTC) | File on the MID | Size |
|---|---|---|
| `2026-11-01 13:03:35` | `torontoevents_20261101050335.csv` | 172 B (= 172 B) |
| `2026-11-01 13:04:06` | `torontoevents_20261101050406.csv` | 5,479 B (= 5,479 B) |

Exactly **8 hours earlier** → the stamp is being rendered in **US/Pacific (PST, UTC−8)**, because
`glide.sys.default.tz` is blank on this instance (platform default is US/Pacific) and the run-as user has no
Time zone set. One export run therefore shows three different times:

```
13:04:06 UTC  (sys_export_set_run, sys_export_set_log, "Last export scheduled run")
08:04:06 EST  (what your Toronto wall clock said)
05:04:06      (what the file NAME says - US/Pacific)
```

**Fix it in one place** — set the instance default time zone (or the run-as user's Time zone) to `America/Toronto`:

```
System Properties → glide.sys.default.tz = America/Toronto
   ...or: User Administration → Users → <run-as user> → Time zone = America/Toronto
```

New files then read `torontoevents_20261101080406.csv` and match your wall clock. Existing files keep their old stamps,
and note daylight saving shifts the offset by an hour twice a year (PST−8 / PDT−7), so never build a parser that assumes
a fixed offset — always pair the filename with `sys_export_set_run`'s timestamps if exactness matters.

### Getting a different date/time format in the name

The stamp format is fixed. The levers you actually have:

| Want | Do this |
|---|---|
| A different **time zone** in the stamp | Set `glide.sys.default.tz` / the run-as user's Time zone (above) |
| A **prefix or suffix** of your own | Export Set → File name, e.g. `torontoevents_prod_` → `torontoevents_prod_20261101080406.csv` |
| One **stable filename** that is overwritten each run | Uncheck **Append timestamp** (you lose per-run history; add a downstream copy step) |
| A **date-based folder** | Not supported by the Export Target (single static path) → move files into `…/2026-11-01/` on the file host (recipe 3) |
| A **fully custom name** (`events_2026-11-01T08-04-06Z.csv`) | Rename on the file host after the export (recipe 1), or bypass export sets and use a Scheduled Job + REST/Attachment API |

---

## 2. How the four records fit together

```
Export Definition   (what rows / which fields)      sys_export_definition
        │
Export Target       (which MID Server + which dir)  sys_export_target
        │
Export Set          (file name, format, max rows)   sys_export_set
        │
Scheduled Data Export (when + delta + scripts)      scheduled_data_export
        │
Export History      (one row per run, with the attachment)  sys_export_set_run
        └─ Export Log (per-run detail messages)             sys_export_set_log
```

An **export set** is only the "file" layer. *What* is exported and *where* it lands live in the other two records,
and *when* / *whether* / *delta* live in the scheduled data export. When something looks wrong, check them in that
order — a stale **definition** (wrong view or field list) produces a perfectly good file with the wrong columns.

---

## 3. The live config in this instance (2026-11-01)

Pulled with the Table API (`/api/now/table/sys_export_set`, `sys_export_definition`, `sys_export_target`,
`scheduled_data_export`, `sys_export_set_run`), not from memory.

**Export Set — `TorontoEvents`** (`sys_id b3b0d82dc3d09750310abf0d05013149`)

| Field | Value |
|---|---|
| File name | `torontoevents.csv` |
| Format | `CSV` |
| Append timestamp | `true` |
| Max rows | `100000` (events table is ~8 K; headroom for growth) |
| Use display value | `true` ← see the BI warning in §8 |
| Use label | `false` (so headers are field names: `u_location`, `u_event_id`, …) |

**Export Definition — `TorontoEventsFinal_ExportDef`** — table `u_torontoeventsfinal`, view `TorontoEventsFinal`,
field list `u_location,u_event_id,u_last_updated,u_event_date,u_title,sys_created_on,sys_created_by,sys_tags,sys_updated_on,sys_updated_by,sys_mod_count,sys_id`.

**Export Target — `ExportToWindows11Mid`** — method `File on MID Server`, MID `win11-mid` (`c1a9c14c…`),
path `toronto/events` → files land in `C:\ServiceNow\MID\agent\export\toronto\events\`.

**Scheduled Data Export — `ExportTorontoEventsFinal`**

| Field | Value |
|---|---|
| Run | `daily` at `08:00:00` (instance tz → 11:00 Toronto if instance is US/Pacific) |
| Delta exports | **enabled**, `Delta on or after = sys_updated_on` |
| Last export scheduled run | `2026-11-01 13:03:35` (the delta watermark) |
| Conditional | `false` |
| Execute pre-export / post-export script | `false` / `false` |
| Parallel export | `false` |

**Export history** (`sys_export_set_run`) — note the delta runs are the tiny ones:

| # | Run at (UTC) | Delta? | Size | Meaning |
|---|---|---|---|---|
| ESET0010001 | 12:55:27 | no | 2,128,595 B | full export (~2.1 MB) |
| ESET0010002 | 12:55:58 | yes | 172 B | delta, no changes since previous run |
| ESET0010004 | 13:02:05 | yes | 172 B | delta, no changes |
| ESET0010005 | 13:02:50 | no | 5,479 B | full export (20-row test view) |
| ESET0010006 | 13:03:35 | yes | 172 B | delta, no changes → `torontoevents_20261101050335.csv` |
| ESET0010007 | 13:04:06 | no | 5,479 B | full → `torontoevents_20261101050406.csv` |

**A 172-byte file is not a failure** — that is the CSV header with zero data rows, and it is exactly what a delta run
produces when nothing changed (`delta_export_status` says so: *"Export includes only records updated on or after …"*).
Suppressing those files is what §6's conditional script is for.

The header (`use_label=false`, `use_display_value=true`) is:

```csv
"u_location","u_event_id","u_last_updated","u_event_date","u_title","sys_created_on","sys_created_by","sys_tags","sys_updated_on","sys_updated_by","sys_mod_count","sys_id"
```

---

## 4. Every configuration option, and where to set it

### Export Definition (`sys_export_definition`)

| Option | What it does |
|---|---|
| Table | The table to export (regular tables only — see delta caveat) |
| View | Which list view's columns to use; the view's **filter is not applied**, only its columns |
| Field list | Exact column set and order in the CSV |
| Condition / filter | Which rows — this is where "only open incidents", "only this assignment group" goes |
| Related lists | Optional child rows flattened into extra columns |

> Column order follows the field list. Renaming a field in the definition is the supported way to change CSV headers —
> there is no header-alias option.

### Export Target (`sys_export_target`)

| Option | What it does |
|---|---|
| Method | `File on MID Server` is the only practical one for export sets |
| MID Server | Which agent writes the file. MID must be **Geneva or later** and **export sets are not supported on clustered MIDs** |
| File path | Directory relative to `<MID install>\agent\export\`. Static — no `%date%`/`${now}` templating |

### Export Set (`sys_export_set`)

| Option | What it does |
|---|---|
| File name | Base name **including extension** (`torontoevents.csv`) |
| Format | `CSV`, `XLS`, `XLSX` (CSV is the safe choice for BI; XLSX row limits and formatting bite at scale) |
| Append timestamp | Adds `_yyyyMMddHHmmss` before the extension (§1) |
| Max rows | Hard cap per file; rows past the cap are simply not in the file. Size it above your growth, or use parallel export |
| Use label | Header = field **labels** instead of field names |
| Use display value | Cell values = **display values** instead of raw values (see §8 — usually wrong for BI) |

### Scheduled Data Export (`scheduled_data_export`)

| Option | What it does |
|---|---|
| Run as | The user whose ACLs/time zone apply. A too-privileged account leaks data; a too-limited one exports blanks |
| Run / Day / Repeat interval / Time | `Daily` / `Weekly` / `Monthly` / `Periodically (Nd Nh)`, plus time-of-day (stored as `1970-01-01 HH:mm:ss`) |
| Time zone of the run | The run-as user's time zone — set it, don't inherit |
| Enable delta exports + Delta on or after | §5 |
| Last export scheduled run | Leave blank for "everything"; set automatically afterwards. Clear it to force a full re-export |
| Conditional + Condition | §6 — run only when the script returns true |
| Execute pre-export script / Pre script | §6 |
| Execute post-export script / Post script | §6 |
| Parallel export | Splits a large export across jobs; useful above ~100 K rows |
| Active | Off = the schedule stays defined but never fires |

Roles: `export_set_admin` to build, `export_set_scheduler` to schedule.

---

## 5. Delta exports

Tick **Enable delta exports** and pick what "changed" means:

- **Updated** (`Delta on or after = sys_updated_on`) — records changed since the last run **plus** records created since
  then. This is what you want for BI sync.
- **Created** (`sys_created_on`) — new records only; edits to existing rows are invisible.

The watermark lives in **Last export scheduled run** and advances after each run (`sys_export_set_run.delta_export_status`
records the value used: *"Export includes only records updated on or after 2026-11-01 13:02:05"*).

**Documented limits**

- Delta works on **regular tables only** — not database views (no `sys_*` system fields to compare).
- You **cannot** also filter the export definition on `sys_updated_on` / `sys_created_on`: enabling delta then makes the
  scheduled data export record refuse to save (error banner on the form). Put "open only" style filters on other fields.
- Clearing **Last export scheduled run** forces the next run to be a full export — the standard recovery move when a
  delta chain breaks or you rebuild the warehouse.
- Delta runs still produce a **file every time**, header-only when nothing changed (§3).

**Cost/benefit** — measured on this instance: full = 2,128,595 B, empty delta = 172 B. For a 500 K-row incident table a
full CSV is typically 150–300 MB/day; a daily delta of a few thousand touched rows is single-digit MB. That is the whole
argument for delta: the instance spends its time filtering instead of serialising, the MID spends seconds writing
instead of minutes, and BI ingest drops by 1–2 orders of magnitude.

**Pairing delta with a periodic full** is the standard pattern: daily delta for freshness, weekly (or first-of-month) full
export to a *different file name* for reconciliation and for rebuilding a broken warehouse. Two export sets, same
definition and target, names `torontoevents_delta.csv` / `torontoevents_full.csv`.

---

## 6. Scripts: conditional, pre-export, post-export

All three run **server-side on the instance** as Glide scripts (JavaScript). They can query tables, log, call
`RESTMessageV2`, create records — and they **cannot touch the file on the MID Server**: the instance has already
handed the payload to the agent. Anything filesystem-shaped (rename, move, delete, compress) belongs on the file host
(§7).

### Conditional — skip a run entirely

```js
// Condition script: return true to run this export, false to skip it.
// Typical use: don't fire a delta export when nothing changed, so no
// header-only file lands in the drop folder.
(function () {
  var ga = new GlideAggregate('u_torontoeventsfinal');
  ga.addQuery('sys_updated_on', '>=', gs.daysAgoStart(1));   // since yesterday 00:00
  ga.addAggregate('COUNT');
  ga.query();
  var n = (ga.next() ? parseInt(ga.getAggregate('COUNT'), 10) : 0);
  if (n > 0) {
    return true;
  }
  gs.info('[ExportTorontoEventsFinal] skipped: 0 records changed since ' + gs.daysAgoStart(1));
  return false;
})();
```

This is the supported answer to "make it skip a run when there is nothing to export" — the schedule doesn't fire, so
no 172-byte file appears. Keep the query cheap: an indexed field (`sys_updated_on` is indexed) and a count aggregate.

### Pre-export script — validate before exporting

```js
// Post a marker / fail loudly if the source looks wrong, so a broken export
// is obvious in the log instead of being discovered in the BI report.
(function () {
  var ga = new GlideAggregate('u_torontoeventsfinal');
  ga.addAggregate('COUNT');
  ga.query();
  var n = ga.next() ? parseInt(ga.getAggregate('COUNT'), 10) : 0;
  gs.info('[ExportTorontoEventsFinal] pre-export: ' + n + ' rows in source table');
  if (n === 0) {
    gs.warn('[ExportTorontoEventsFinal] pre-export: source table is EMPTY - export will be header-only');
  }
})();
```

### Post-export script — audit, notify, chain the next step

```js
// Record what was produced and tell a human when it's abnormal.
(function () {
  var set = new GlideRecord('sys_export_set');
  if (!set.get('b3b0d82dc3d09750310abf0d05013149')) {   // this export set
    return;
  }
  // Most recent run for this set
  var run = new GlideRecord('sys_export_set_run');
  run.addQuery('set', set.getUniqueValue());
  run.orderByDescending('sys_created_on');
  run.setLimit(1);
  run.query();
  if (!run.next()) {
    return;
  }
  var size = parseInt(run.getValue('attachment_size') || '0', 10);
  gs.info('[ExportTorontoEventsFinal] post-export: ' + run.getValue('number') +
          ' size=' + size + 'B state=' + run.getValue('state'));

  if (size < 300) {                                    // header-only / suspiciously small
    var rm = new RESTMessageV2('Teams - Alerts', 'postMessage');
    rm.setStringParameterNoEscape('text',
      'Events export produced a header-only file (' + run.getValue('number') + ').');
    rm.execute();
  }
})();
```

Because both scripts run **inside** the schedule, they are also the right place to write an audit row
(`u_export_audit`: set, run number, size, timestamp) so dashboards can graph export health instead of guessing.

---

## 7. Recipes people actually use

### When to use export sets — and when not

**Use them when**

- the consumer wants a **file**: a shared folder, SFTP drop, Power BI folder connector, Qlik folder data connection, or an Excel user who will never use an API;
- you want **zero API load on the instance** from BI refreshes — the export runs once, the BI tool re-reads a file;
- you need **scheduled snapshots** for audit or reconciliation ("what did this look like last Tuesday?");
- the rows must be **display values** (labels, choice text) for humans;
- the destination sits **behind a firewall** where a MID already exists but the BI server has no OAuth/REST path.

**Don't use them when**

- you need **near-real-time** (minutes, not hours) → REST / IntegrationHub / Remote Tables;
- the payload includes **attachments or binaries** — export sets do not export attachments (documented);
- the source is a **database view** and you want delta — not supported (no `sys_updated_on` to compare);
- you need **per-row authorisation** — an export runs as one user and writes everything that user can read;
- there is **no MID Server** (export sets require one, non-clustered, Geneva or later);
- the consumer only reads **JSON/XML** — this is a flat-file mechanism.

Rule of thumb: export sets are a **bulk egress** tool. For sync-into-BI, delta + watermark is the cheapest thing that
works; for anything event-driven, use an API.


### 1. Rename to a predictable, sortable name (and keep a stable "latest" copy)

Run on the MID/Windows host (Task Scheduler, every 5 minutes), not in a ServiceNow script:

```powershell
# C:\ServiceNow\MID\agent\export\toronto\events\ -> ..\toronto\events\archive\
$src = 'C:\ServiceNow\MID\agent\export\toronto\events'
Get-ChildItem "$src\torontoevents_*.csv" | ForEach-Object {
    $stamp = $_.LastWriteTime.ToString('yyyy-MM-ddTHH-mm-ss')     # local, honest name
    $dest  = "$src\archive\torontoevents_$stamp.csv"
    Move-Item $_.FullName $dest -Force
    Copy-Item $dest "$src\latest\torontoevents_latest.csv" -Force  # BI points at this
}
```

### 2. Never land an empty file

Two layers, use both: the **conditional script** (§6) for a clean skip, and a size guard on the host for the cases the
script can't see (delta produced 0 rows after the condition passed):

```powershell
Get-ChildItem "$src\torontoevents_*.csv" | Where-Object { $_.Length -lt 300 } |
    ForEach-Object { Write-EventLog -LogName Application -Source "SNExport" -EventId 1001 `
        -EntryType Warning -Message "Empty export discarded: $($_.Name)"; Remove-Item $_.FullName }
```

### 3. Date-partitioned landing zone

`Export Target → File path` is static, so partition after the fact: `archive\2026-11-01\torontoevents_…csv`.
Power BI/Qlik then read one folder per day (or the whole archive with a folder-vs-file pattern) and you get cheap
retention: delete day-folders older than N days.

### 4. Retention

```powershell
Get-ChildItem "$src\archive" -Directory |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-30) } | Remove-Item -Recurse -Force
```

### 5. Full weekly + delta daily

Two Scheduled Data Exports on the same export set: daily delta `torontoevents_delta.csv`,
Sunday 02:00 full `torontoevents_full.csv` (delta off, or clear *Last export scheduled run* before that run).
Downstream treats the full as the authoritative snapshot.

### 6. Alerting on silence

The absence of files is the failure mode that looks like success. Watch the newest file's age on the host, or query
`sys_export_set_run` from a monitoring tool: if `sys_created_on` of the newest run for the set is older than
`now − 2 × interval`, page someone.

### 7. Two consumers, two shapes

Humans (Excel, ad-hoc) → `use_display_value=true`, `use_label=true`, labels in headers.
Systems (Power BI, Qlik, warehouse) → `use_display_value=false`, `use_label=false`, raw values and field names.
Do not try to serve both from one export set; make two, it costs nothing.

---

## 8. Feeding Power BI / Qlik Sense — and saving space with deltas

### The incident example

Export the `incident` table instead of a custom one: definition fields
`number, sys_id, state, priority, category, assignment_group, caller_id, opened_at, resolved_at, sys_created_on, sys_updated_on`,
filter to your reporting scope, **delta on `sys_updated_on`**, daily; plus a weekly full.

Ingest rules that matter for both tools:

| Rule | Why |
|---|---|
| Keep `sys_id` as a **text** column | It is the only stable key across delta files |
| Dedupe on `sys_id`, keep the row with **max `sys_updated_on`** | A record edited 5 times appears in 5 delta files |
| Store `sys_updated_on` as **UTC datetime**, format in the report | Delta watermark and run history are UTC; the *filename* is not (§1) |
| Parse CSV as **Windows-1252** | ServiceNow writes CSV in Windows-1252; skipping this mangles `’`, `é`, `–` |
| Turn off `use_display_value` for system consumption | Display values are localised and ACL-dependent: a join on `state` breaks the day someone changes a label |
| Treat a small file as legitimate | Deltas are supposed to be small; filter *empty* (header-only), not "small" |
| `max_rows` > your table size, or use parallel export | Silent truncation otherwise |

### Power BI

1. **Get Data → Folder** → the export landing folder (`…\toronto\events\archive`). `Combine Files` gives you every
   daily file stacked.
2. Force the encoding in the generated query (Power Query defaults to UTF-8 and will corrupt Windows-1252):

```m
let
    Source   = Folder.Files("C:\ServiceNow\MID\agent\export\toronto\events\archive"),
    Keep     = Table.SelectRows(Source, each Text.StartsWith([Name], "torontoevents_")),
    Loaded   = Table.AddColumn(Keep, "Rows", each
                   Csv.Document([Content], [Delimiter = ",", Encoding = 1252, QuoteStyle = QuoteStyle.Csv])),
    // promote headers from the first file, then expand
    Head     = Table.First(Loaded)[Rows],
    Hdr      = Table.PromoteHeaders(Head, [PromoteAllScalars = true]),
    Cols     = Table.ColumnNames(Hdr),
    Expanded = Table.ExpandTableColumn(Table.Combine(Loaded[Rows]), Cols, Cols),
    Typed    = Table.TransformColumnTypes(Expanded, {
                   {"sys_updated_on", type datetime}, {"sys_created_on", type datetime},
                   {"opened_at", type datetime},      {"resolved_at", type datetime}}),
    // last write wins per record
    Latest   = Table.Group(Typed, {"sys_id"}, {
                   {"row", each Table.First(Table.Sort(_, {{"sys_updated_on", Order.Descending}}))}}),
    Final    = Table.ExpandRecordColumn(Latest, "row", Cols)
in
    Final
```

3. **Incremental refresh** on `sys_updated_on` with `RangeStart`/`RangeEnd` parameters: the mashup is then allowed to
   re-read only recent files, and Power BI keeps the accumulated history in the dataset. Add a filter step
   `each [sys_updated_on] >= RangeStart and [sys_updated_on] < RangeEnd` *before* `Table.Group`.
4. Publish, then schedule refresh ~30 minutes after the export window (and remember you still need a real refresh for
   late-arriving deltas — a record can be re-exported in a later delta, which the group-by handles).
5. Reconcile monthly against a full export: `count(distinct sys_id)` and `countrows` per day should match.

### Qlik Sense

```qlik
// 1. Load every delta/extract file, then keep the newest version of each record
Encounter:
LOAD
    sys_id,
    number,
    state,
    priority,
    assignment_group,
    opened_at,
    resolved_at,
    Timestamp#(sys_updated_on, 'YYYY-MM-DD hh:mm:ss') AS sys_updated_on_ts
FROM [lib://SNExport/torontoevents_*.csv]
    (txt, codepage is 1252, embedded labels, delimiter is ',', msq);

// 2. Dedupe -> one row per record, newest edit wins
Latest:
LOAD * Resident Encounter
Where Not Exists(sys_id);
// load order matters: sort the Resident load by sys_updated_on descending first
DROP Table Encounter;
```

The classic Qlik space/time saver is the **QVD layer**, which is also where delta exports pay off twice:

```qlik
// Incremental QVD pattern: accumulate deltas, never re-read history
Incremental:
LOAD * FROM [lib://QVD/incident_latest.qvd] (qvd)                  // yesterday's accumulated state
WHERE NOT Exists(sys_id, sys_updated_on_ts);

Concatenate(Incremental)
LOAD * FROM [lib://SNExport/torontoevents_delta*.csv]
    (txt, codepage is 1252, embedded labels, delimiter is ',', msq)
WHERE NOT Exists(sys_id, sys_updated_on_ts);

STORE Incremental INTO [lib://QVD/incident_latest.qvd] (qvd);
```

Only the new delta files are parsed each reload (seconds), the QVD is read back once, and the app never touches the
weekly full except during a reconciliation reload. If a delta chain is ever suspect: clear **Last export scheduled run**
(or run the weekly full set), delete the QVD, reload once.

### Where the savings come from (delta vs always-full)

| | Full daily | Delta daily + weekly full |
|---|---|---|
| Instance render | Every row, every day | Only changed rows |
| MID write | 150–300 MB/day at 500 K incidents | a few MB/day |
| Network/host disk | grows ~2–6 GB/month | ~hundreds of MB/month |
| BI ingest | full re-parse daily | parse ~1 % of the volume |
| Failure recovery | trivial (one file = the truth) | needs the watermark + a periodic full |
| Reconciliation | built in | needs the weekly full |

Rule of thumb: **delta for freshness, full for truth**. Weekly full + daily delta + dedupe by `sys_id` gives you
full-accuracy data at roughly 1–2 % of the bytes.

---

### Alternatives for Qlik Sense — loading only the deltas

Export sets are one of five ways to get ServiceNow rows into Qlik while still loading only what changed. All five need
the same two things: a **watermark** (`sys_updated_on`, stored in the app) and **a story for deletes**.

| # | Route | How "updated only" works | Cost / caveat |
|---|---|---|---|
| 1 | **Export set + delta** (this repo) | ServiceNow filters via `last_export_run`; Qlik reads the newest file(s) | needs a MID + file access; no deletes; one run's latency |
| 2 | **REST Table API from Qlik** (REST connector or a script) | you send `sys_updated_on>=<stored watermark>`, page with `sysparm_offset` / `sysparm_limit` | no MID; mind API limits and page size; store the watermark in a QVD |
| 3 | **Remote Table / Data Stream** (IntegrationHub, licensed) | ServiceNow pushes changed rows to a target DB as they change | cleanest and fastest; needs the plugin, an external DB, credit spend |
| 4 | **JDBC/ODBC driver in Qlik** (CData / Progress / DataDirect) | Qlik sends `WHERE sys_updated_on >= '$(vWatermark)'`; the driver translates to REST | licence cost; hits the same API limits underneath |
| 5 | **Export set → MID → loader → DB** (file watcher runs `LOAD DATA`) | the DB does the incremental merge; Qlik reads the DB | most moving parts; good if a warehouse already ingests files |

**The Qlik pattern that works for all of them** — incremental QVD, never a full reload:

```qlik
// keep the watermark in the QVD so it survives reloads
Watermark:
LOAD Timestamp(max(sys_updated_on)) AS wm RESIDENT Incident;
LET vWm = peek('wm', 0, 'Watermark');

DeltaRaw:
LOAD * FROM [lib://SNExport/torontoevents_delta*.csv]
    (txt, codepage is 1252, embedded labels, delimiter is ',', msq);

Delta:
NoConcatenate LOAD *, Timestamp#(sys_updated_on,'YYYY-MM-DD hh:mm:ss') AS upd RESIDENT DeltaRaw
WHERE Timestamp#(sys_updated_on,'YYYY-MM-DD hh:mm:ss') >= '$(vWm)';

Concatenate(Incident)
LOAD * RESIDENT Delta WHERE NOT Exists(sys_id);

DROP TABLES DeltaRaw, Delta, Watermark;
STORE Incident INTO [lib://QVD/incident.qvd] (qvd);
```

Practical rules for a delta-only feed:

- **Overlap the watermark** 10–15 minutes and dedupe on `sys_id`, so a row committed on the boundary is not lost.
- **Deltas never show deletes.** Either re-load a small full slice periodically (weekly), or add an explicit
  `active`/soft-delete column so a deleted row still *arrives* as changed.
- **`sys_updated_on` is UTC in the API** but a display time zone in the UI — normalise once, at load.
- The **first load must be a full export**; incremental-from-empty gives a partial history that looks complete.
- If the delta chain breaks (instance refreshed, watermark cleared) do not patch it: clear *Last export scheduled run*
  on the ServiceNow side, run one full, rebuild the QVD once.

## 9. Troubleshooting

| Symptom | Cause | Do this |
|---|---|---|
| File is only the header row (e.g. 172 B) | Delta run, nothing changed since the previous run | Expected. Confirm with `sys_export_set_run.delta_export_status`; suppress with a conditional script (§6) |
| File never appears | MID down / target path wrong / no write permission on the share | `sys_export_set_log` + `ecc_queue`; verify the MID is **Up** and validated; check `<MID>\agent\export\<path>` exists |
| Filename timestamp looks 8 h (or 1 h in summer) off | Stamp is rendered in the instance/user time zone, not UTC | Set `glide.sys.default.tz` and the run-as user's Time zone (§1) |
| Timestamp format isn't what you want | The `_yyyyMMddHHmmss` format is fixed | Rename on the file host (§7.1), or change time zone for a different value |
| "Cannot save" the Scheduled Data Export | Delta enabled **and** the definition filters on `sys_updated_on`/`sys_created_on` | Remove that filter from the definition, or turn delta off |
| Delta export returns nothing after the first run | Watermark advanced, no changes since | Touch nothing: verify with `sys_export_set_run`; clear *Last export scheduled run* for a forced full |
| Blank cells where you expect values | Run-as user's ACLs can't read those fields/rows | Export as a user with read access; never `admin` for data extracts |
| Rows missing at the top of the file | `Max rows` cap | Raise it (or parallel export) |
| Attachments/related docs missing | Export sets don't export attachments (documented) | Use the Attachment API / REST message for binaries |
| Two MIDs and duplicate/interleaved files | Export sets are **not supported on clustered MIDs** | Point the target at a single, non-clustered MID |
| Field order/headers changed | Someone edited the view, not the definition's field list | Fix the export definition and re-test with Execute Now |
| CSV mojibake in BI (`â€™`, `Ã©`) | Read as UTF-8; the file is Windows-1252 | Set the encoding at the source (§8) |

---

## 10. Verifying any of this from the Table API

Everything in this README was read live; these are the calls (Basic auth or OAuth, `sysparm_*` params):

```bash
# The export set (file name, append timestamp, max rows, format)
curl -s -u "$SN_USER:$SN_PASS" -H 'Accept: application/json' \
  "https://<instance>.service-now.com/api/now/table/sys_export_set?sysparm_limit=5&sysparm_exclude_reference_link=true"

# What is exported (table, view, field list)
.../api/now/table/sys_export_definition

# Where it lands (MID server, path)
.../api/now/table/sys_export_target

# When / delta / scripts
.../api/now/table/scheduled_data_export

# Run history: timestamps, sizes, delta watermark per run  <- pairs with the filenames
.../api/now/table/sys_export_set_run?sysparm_limit=20

# Per-run log messages ("Transferred torontoevents.csv to the MID Server 'win11-mid'")
.../api/now/table/sys_export_set_log?sysparm_limit=20
```

The fastest end-to-end check: press **Execute Now** on the export set, then compare the newest
`sys_export_set_run` row (`sys_created_on`, `attachment_size`, `state`) with the file that appears in the target
folder. Same size + same time (modulo the time-zone offset in §1) = the pipeline is healthy.

---

## Files in this repo

| File | What it is |
|---|---|
| `torontoevents_20261101050335.csv` | 172 B — delta run with no changes: header only (the "blank export") |
| `torontoevents_20261101050406.csv` | 5,479 B — a full export with records (20-row test view) |

```
"u_location","u_event_id","u_last_updated","u_event_date","u_title","sys_created_on","sys_created_by","sys_tags","sys_updated_on","sys_updated_by","sys_mod_count","sys_id"
```

## References

- Export sets: <https://www.servicenow.com/docs/r/integrate-applications/system-export-sets/c_ExportSets.html>
- Create an export set (all export set fields): <https://www.servicenow.com/docs/r/integrate-applications/system-export-sets/t_CreateAnExportSet.html>
- Schedule an export (delta, conditional, pre/post scripts, parallel): <https://www.servicenow.com/docs/r/integrate-applications/system-export-sets/t_ScheduleAnExport.html>
- Exporting data / CSV encoding (Windows-1252): <https://www.servicenow.com/docs/r/platform-administration/table-administration-and-data-management/c_ExportData.html>
- Related repo — querying the exported events data with MySQL/JDBC: <https://github.com/eltonaguiar/ServiceNowJDBC>
- MID Server operations (the box these files land on): see `ServiceNowMidServer` in the same GitHub account
