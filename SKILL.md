---
name: comindware-log-analysis
description: Analyze Comindware platform logs to diagnose incidents. Use when you need to find the root cause of an issue from Comindware logs (e.g. user mentions "логи Comindware", "анализ логов платформы"): performance problems and slow work, errors and exceptions, service crashes/restarts, integration and data exchange issues, authentication and licensing problems, database problems. Runs an interview to determine the problem type and time window, then analyzes only the relevant log files from the user-provided directory within the specified period, and produces a detailed report with log excerpts, a timeline, and recommended solutions.
---

# Comindware Log Analysis

Skill for investigating incidents from Comindware platform logs. Works in three steps: **Interview → Analysis → Report**.

## Step 1. Interview the user

Before reading any logs, find out what the problem is. Ask questions and only proceed to analysis after the user has answered. Ask questions with the `question` tool; more than one category may be selected.

### 1.1 Problem type

Ask the user to pick a category (multiple allowed):

| Category | Key symptoms |
|-----------|-------------------|
| Performance | Slow work, freezes, timeouts, high server load |
| Errors and exceptions | UI/API errors, exceptions, business-logic failures |
| Service crashes | Platform services restart/stop, service fails to start |
| Integrations and data exchange | Issues with external systems, data exchange, webhooks, import/export |
| Authentication and licensing | Login problems, authorization failures, expired/licensing errors |
| Database | DB errors, slow queries, locks, data loss (on Linux stack the DB is Apache Ignite — memory exhaustion, stalled checkpoints, node restarts) |

### 1.2 Clarifying questions

After the category is chosen, ask follow-ups (not all at once — only the ones relevant to the selected category):

- When did the problem start (time, date, time zone)? Is there an exact event time?
- What was the user doing at the moment of the failure (concrete scenario)?
- Is the problem reproducible consistently or a one-off?
- Did anything change before the incident (platform/application update, config change, migration, data growth)?
- Are all users affected or just one? All applications or a specific one?
- Do you already know the Comindware version/edition and the deployment type (Windows+IIS, docker, etc.)?

### 1.3 Path to the logs

Ask the user to provide the **directory that contains all incident logs**. This can be:
- the standard `logs` folder of a Comindware installation;
- an incident dump (a folder or an unpacked archive);
- any directory with platform and system logs.

If the path is not given, do not start the analysis — ask for it.

### 1.4 Analysis period (required)

Logs are large — analyze **only** the records within the incident window, otherwise tokens are wasted. Always ask the user for:

- **Window start** date and time (from).
- **Window end** date and time (to).
- The **time zone** the user's timestamps are in (note: log timestamps may use a different TZ).

Rules:
- If the user does not give a period, do not start reading logs — ask for it.
- You may propose a sensible default window (e.g., ±2 hours around the known incident time) and confirm it with the user.
- If the user gives a date without a time, default to the full day (00:00–23:59) and confirm it is acceptable.
- For a long incident you may split the analysis into several windows, but analyze each window separately.
- There may be more than one period (e.g., "yesterday from 10:00 and today from 08:00") — record each segment.

Before analyzing, write down the selected window: `from <datetime> to <datetime>, TZ=<time zone>`.

## Step 2. Analyze the logs

### 2.1 First inspect the structure

Before deep searching:
1. If `LEARNINGS.md` exists in the skill directory (`/home/alesha/gh-repos/comindware-log-analysis/`), read it and apply the recorded lessons (markers, correlations, methodologies) during this analysis.
2. List the contents of the given directory (recursively).
3. Determine the stack and deployment type from the file inventory (do not ask the user — derive it from the logs):
   - **Windows/IIS deployment**: IIS logs (`W3SVC*.log`, `u_ex*.log`), `*.evtx` Windows Event logs;
   - **Linux deployment**: nginx (`access.log`/`error.log` rotation), `*.service` systemd units, `journal.log` (systemd journal dump), Kafka (`kafka/`, `kafkaClient_*.log`), Apache Ignite (`igniteClient_*.log`, `ignite/Ignite.config`), `journalservice/` (audit/search index service);
   - MongoDB logs (`mongod.log`) → MongoDB database;
   - platform root logs `comindware*.log` or service logs.
4. Identify the **stand name** (e.g., `cmwdata`) from file/directory prefixes: `<standname>.yml`, `<standname>_logs/`, `<standname>-access.log`, `comindware<standname>.service`. The stand name is used across configs and log names.
5. Summarize: which files were found, their sizes, and the period they cover (from first to last record). Note the large files (hundreds of MB in nginx/performance/igniteClient) — read them only via targeted search, never fully.

### 2.1a Typical incident dump layout

A real incident dump often looks like this (adapt to whatever is actually present):

```
<standname>.yml            # main platform config
apigateway.yml             # API gateway config
adapterhost.yml            # adapter host config
comindware<standname>-env  # environment file
comindware<standname>.service  # systemd unit
status.log                 # services status at dump time
journal.log                # systemd journal dump
<standname>_logs/          # main platform logs, one file per component per day:
    <component>_YYYY-MM-DD.log     # e.g. system_, error_, performance_, process_, audit_, heartbeat_, backup_, upgrade_
    adapter_external_*.log         # external integrations/adapters (system, heartbeat, kafkaClient)
    adapter_internal_*.log         # internal adapters (+ _error_ suffix for adapter errors)
    apigateway_*.log               # API gateway
    igniteClient_*.log             # Apache Ignite client (can be very large)
    kafkaClient_*.log              # Kafka client
nginx/                    # reverse proxy logs + rotation
    <standname>-access.log, <standname>-access.log.N[.gz]
    <standname>-error.log, error.log
kafka/                    # Kafka server: controller.log, server.log, server.properties, kafka.service
ignite/                   # Ignite config
journalservice/           # journal/index service: cluster_health.log, indices_stats.log, instance_indices.log, service_info.log
permissions/              # filesystem permissions dump
```

Log naming convention in `<standname>_logs/`: `<component>_YYYY-MM-DD.log` (a day per file, often two days present: the incident day and the day before).

**`journal.log`** is a dump of the systemd journal (journalctl output). Line format:
```
мес дд чч:мм:сс <hostname> <unit>/<process>[<pid>]: <message>
```
Also contains `-- Logs begin at <date>, end at <date> --`. The platform service is `comindware<standname>.service` (e.g. `comindwarecmwdata.service`). Watch for systemd lifecycle lines: `Main process exited`, `Failed with result`, `SERVICE_STOP`/`SERVICE_START`, `CRON ... systemctl start`.

**`adapter_*_*.log`** — adapter host / integration logs (external and internal adapters). Naming:
- `adapter_external_system_*.log` — external adapter host lifecycle (start/stop, loaded config from `adapterhost.yml`, MQ connection settings);
- `adapter_external_heartbeat_*.log` — periodic adapter host health: data-transfer paths, loaded adapters, process stats (threads, memory);
- `adapter_external_kafkaClient_*.log` — adapter host MQ client;
- `adapter_internal_system_*.log` — internal (in-platform) adapter service operations (deploy of adapter definitions, `HttpListenerAdapter`, `HttpRequestAdapter`, `MySqlClient`, etc.);
- `adapter_internal_system_error_*.log` — adapter errors (endpoint/procedure failures);
- `adapter_<connection-name>_*.log` — logs for a specific named connection (a separate file per connection may appear);
- `integration_raw*` — raw integration data (typically used for OData exchange).

**Error line format in adapter logs:**
```
[<timestamp>][<LEVEL>] PlatformKey: <stand>_<...>. Procedure: <procedure> (procedure.<n>). Endpoint: <endpoint> (endpoint.<n>). Adapter: <AdapterType>  <message>
```
The `Procedure`/`Endpoint`/`Adapter` fields identify exactly which integration route failed. Common errors: `Instance not started: ... Путь передачи данных «<name>» отключен` (data-transfer path disabled / instance down).

**`igniteClient_*.log`** is the log of the embedded Apache Ignite node — **the platform's database** (in-memory + persistent regions, WAL, checkpoints, off-heap memory). It is one of the largest files (can be ~100MB/day) and is mostly periodic metrics. Key content:
- periodic metric blocks: `Metrics for local node`, `Heap [used=XMB, free=Y%]`, `Off-heap memory [used=..., allocated=...]`, `Data storage metrics for local node`, thread pools, `FreeList`, region stats;
- data regions: `Persistent region [type=user, persistence=true, ...]` (disk-backed), `InMemory region [type=user, persistence=false, ...]`, `Default_Region`, internal regions;
- `checkpoint` lines (persistence checkpoint progress; look for stalled/old checkpoints — e.g. "earliest reserved checkpoint" stuck — a sign of write problems);
- WAL/persistence activity (`walMode`, `walFsyncDelay`, etc. in the config block);
- startup blocks: `IgniteConfiguration [igniteInstanceName=<stand>, ..., nodeId=<id>]`, `Configured failure handler: StopNodeOrHaltFailureHandler`;
- noise `WARN`s: "Stack smashing detected", "Consistent ID is not set", "New version is available".
`ignite/Ignite.config` holds the node configuration; `ignite/no_ignite_logs.log` (if present) says the dedicated Ignite log dir was not found.

### 2.2 Select relevant logs by category

Determine which files to look at based on the category. Adapt to the actual stack found in 2.1 (IIS vs nginx, MongoDB vs Ignite/Kafka, etc.):

| Category | Logs to check first |
|-----------|------------------------------|
| Performance | `performance_*.log` (platform perf), `apigateway_*.log` / IIS logs (`time-taken`, response codes), `igniteClient_*.log` (cache latency), `kafkaClient_*.log`, nginx access logs (slow upstream responses, HTTP 500/503) |
| Errors and exceptions | `error_*.log` and `<component>_*_error_*.log`, root `error_*.log`; search for `ERROR`, `Exception`, `FATAL`. Cross-check with `system_*.log` for context. **If `error_*.log` is inconclusive but the platform crashed/hung → `journal.log` (SIGKILL/OOM-killer) and `igniteClient_*.log` (Heap/Off-heap exhaustion, node restarts)** |
| Service crashes | `journal.log` (systemd journal: `Main process exited`, `Failed with result`, `SERVICE_STOP`/`SERVICE_START`, `oom-killer`), `process_*.log` (service lifecycle), `status.log` (services status at dump time), `heartbeat_*.log` (missing heartbeats = down service), `igniteClient_*.log` (node restart/memory exhaustion before crash) |
| Integrations and data exchange | `adapter_internal_system_*.log` + `adapter_internal_system_error_*.log` (endpoint/procedure errors), `adapter_external_system_*.log` + `adapter_external_heartbeat_*.log` (adapter host health), `adapter_<connection-name>_*.log` (specific connections), `integration_raw*` (OData), `kafka/` + `kafkaClient_*.log` (message bus), `apigateway_*.log` (API routing), nginx `error.log` |
| Authentication and licensing | `error_*.log` (login failures, license errors — component `Session`/`ExtAudit`/`Service`), `system_*.log` (AD/directory activity, domain sync) + `idm*` log if present (identity-management / AD details), `apigateway_*.log` (401/403), nginx/IIS access logs (401/403 spikes), `journal.log` (auth-related) |
| Database | **`igniteClient_*.log`** — the platform's database IS Apache Ignite (in-memory + persistent regions, WAL, checkpoints, off-heap memory). Look at `Data storage metrics`, `Heap`/`Off-heap` usage, checkpoint progress, region stats, and error/WARN markers. `journalservice/*` (index/search cluster health, indices stats) if journal-related |

Common fallback: if the component is unclear, start from `system_*.log` (service activity overview), `error_*.log` (all errors), and `status.log` / `journal.log` (what was running at the time).

### 2.2a `performance_*.log` structure (platform request metrics)

This log records per-operation durations for requests/transactions hitting the platform. One JSON object per line. `Duration` is in seconds.

**Fields (top level):**
- `Time` — timestamp (with TZ, e.g. `+03:00`).
- `Duration` — seconds.
- `Id` — unique id of this record.
- `Origin` — id of the root transaction this record belongs to (equals `Id` of the top-level record).
- `Parent` — id of the parent record; **`null` for a top-level transaction root**.
- `Details` — event info (see below).
- `Count` — optional: how many times the operation was aggregated (e.g. `DataSourceService.GetInfo` with `Count: 11` means 11 calls aggregated into one record).

**`Details` common fields:** `Source`, `Action`, `Type`, `Session`, `Account`, `RequestId`.

**Sources (top-level transactions, `Parent: null`):**
| Source | Meaning |
|--------|---------|
| `WebRequest` | HTTP request to platform (e.g. `/api/Dataform/QueryData`, `/DatasetData/Query`, `/Home/Login/`) |
| `MessageBroker` | Message-bus event (e.g. `request_queue_<stand>_...`, `reply_queue_<stand>_...`) |
| `Worker` | Background worker (e.g. `SystemActionQueue`, `UserDefinedActionQueue`, `ClusterCoordinationWorker`) |
| `History` | `EventJournalService` — journal/history writes |
| `Heartbeat` | Service health ping (`OnAggregateSensors`) |
| `FullTextSearch` | Full-text search flush/commit |
| `ServiceMethod` (top-level) | Standalone service call (e.g. `UserCommandExecutionService.Execute`, `AsymCryptService.Decrypt`) |
| `Initialize`, `Timer`, `Backup` | Startup, scheduled timer, backup |

**Transaction nesting:** a top-level record has `Parent: null`. Its children reference it via `Origin`; deeper nesting via `Parent`. Example chain for `/DatasetData/Query`:
```
WebRequest /DatasetData/Query                  (root, Parent=null)
└─ ServiceMethod DatasetService.QueryData      (Parent=root)
   ├─ ServiceMethod DatasetConfigurationService.Get
   │  ├─ ServiceMethod CardConfigurationService.GetAttachedCard
   │  ├─ ServiceMethod DataSourceService.GetInfo      (+ Count: N)
   │  └─ ServiceMethod ToolbarService.GetAttachedToolbar
   └─ ...
```
To measure where time went, compare `Duration` of the root vs its children (children durations are a subset; the difference is overhead/other work).

**`Details.Type` variants and their extra fields:**
| Type | Meaning | Extra `Details` fields |
|------|---------|------------------------|
| `Event` | Generic service/HTTP/broker event | `Source`, `Action`, `Session`, `Account` |
| `Expression` | Evaluation of a business expression/rule | `ExpressionId` (e.g. `formRule.*`, `user.rule.*`, `bruleDef.*`) |
| `DataCommit` | DB transaction commit | `TxId`, `ChangesCount`, `TxParent` (transaction type, e.g. `Comindware.LogicsStorage.Api.Transaction`) |
| `DatasetRequest` | Dataset query | `DatasetId`, `Container`, `PageIndex`, `PageSize` |
| `QueryForm` | Form query | `FormId`, `Form`, `ObjId` |
| `UserCommand` | User command execution | `CommandId` |
| `TriggerAction` | Business trigger fired | `Action` (`ta.*`), `Container` (`pa.*`/`oa.*`), `Kind` (e.g. `ReferenceObjectChange`, `ForeachOperator`) |
| `Token` | Process/workflow token movement | `PID`, `TID` (`ptkn.*`), `PSA` (`psa.*`), `Info` (enter/exit by flow `psf.*`) |
| `NativeQuery` | Raw SQL query to DB | `SqlQueryText`, `RowCount`, `DoQueryTotal` |
| `RequestContextCommit` | Context commit phase | `Name` (e.g. `RequestContext`, `RequestContext.Cache`), `Kind` |
| `WidgetDataReader` / `WidgetDataWriter` | Form/widget data load/save | `Form`, `Container` |
| `HistoryStatistic` | Journal queue statistics | `EventsCount`, `ElapsedSeconds`, `CountInQueue` |
| `Trigger` | Trigger overhead | `Trigger` |
| `DeliverMessage` | Message delivery | `Sender`, `Receiver` |
| `BrainInitialize` | AI/brain init | — |

**How to use for performance analysis:**

The slow transaction itself is rarely the root cause — it is usually a symptom. Analyze the **situation before and after** the slow operation to find what made it slow.

1. **Find the slow top-level transactions** (`Duration` above threshold, e.g. > a few seconds; compare against typical values in the file).
2. **Look at their children** (`Origin` = the slow root's `Id`): which child operation has the largest `Duration` — that's the hotspot.
3. **Analyze what happened BEFORE the slowdown** (this is the key step). Scan the same time window for preceding heavy operations that could be the trigger:
   - `DataCommit` with very high `ChangesCount` (e.g. 5000+ objects) — a large data change right before the slowdown;
   - `Expression`-heavy requests — opening a form/table/list with filter expressions (`ExpressionId` of type `formRule.*`, `user.rule.*`, `bruleDef.*`) that evaluates a lot of rules;
   - `TriggerAction`/`Token` bursts — business logic / process loops;
   - `NativeQuery` with heavy SQL, large `RowCount`;
   - cache-heavy operations (`RequestContextCommit` with `Name: RequestContext.Cache`).
   Correlate the timing: the heavy operation precedes the slowdown by seconds/minutes and its effect propagates to later operations (cache thrash, lock held, transaction volume, DB load).
4. **Check what happened AFTER** — did subsequent operations (even lightweight ones) stay slow? This indicates a systemic effect (DB lock, exhausted pool, memory pressure), not a one-off.
5. **Correlate with other logs**: `error_*.log`, `system_*.log`, DB/journal logs for the same window to confirm the trigger (e.g. a lock, an OOM, a slow query).
6. **Pay attention to `Count: N` aggregated records** — an operation called N times may sum up to more than the root's `Duration`.
7. Note the timestamps are in platform TZ — convert to the window TZ from the interview (1.4).

Pattern to remember: **slow operation is the symptom; the cause is usually a heavy operation just before it (or a sustained load) that degraded the system for everything that followed.** Always analyze the transactions before and after, not just the slow one.

### 2.2b `error_*.log` structure (errors and exceptions)

One logical line per event; message may continue on the next physical lines or contain embedded `\n`.

**Line format (fields may be missing depending on context):**
```
<Date> <Time>,<ms> <LEVEL> <RequestId> <Account> <Host> <Port> <ClientIP> <Component> <Elapsed> <Version>  '<message>'
```

- `<LEVEL>` — `ERROR`, `WARN`, etc.
- `<RequestId>` — the request that failed; `00000000-0000-0000-0000-000000000000` = background/async task without a request.
- `<Account>` — user; `systemAccount` = system/background.
- `<Host> <Port> <ClientIP>` — present for HTTP requests (e.g. `pki.favr.ru 8080 91.212.81.159`).
- `<Component>` — module that logged: `ExtAudit` (audit/ext), `Web` (WebAPI), `Core` (business rules, core services), `Process.Core` (process engine), `TeamNetwork` (tasks/team), etc.
- `<Elapsed>` — operation duration, e.g. `00:00:00.079`.
- `<Version>` — platform version, e.g. `5.0.24234.0`.
- `<message>` — in quotes `'...'`.

**Two formats of messages:**

1. **Inline text** — short human message:
   ```
   'ASYNC: "Authentication error occurred while processing the service request."'
   'Operation "GET" completed with an error: ...'
   'Operation complete with error. '
   'У аккаунта <Name> нет разрешения «<permission>» на доступ к ресурсу «<resource>»'
   'Login failed: unknown username, wrong password or account disabled.'
   'WebAPI exception calling action "Theme":"GetDefaultThemeStyles" on controller "...Controller"'
   'Could not load file or assembly '<hash>, Version=...' or one of its dependencies.'
   'CalculateScript failed: "<script>" in "<path>.dll"'
   ```

2. **Structured exception block** — a `Core` component entry followed by labeled continuation lines:
   ```
   Service name:
      "<ServiceName>"
   Method name:
      "<MethodName>"
   Parameters list:
      "[0]: \"<param>\"" ...
   Stack:
      "  at <frame> \n  at <frame> ..."
   ```

**Key interpretation rules:**

- **One event is often logged multiple times** (same `RequestId`, same second) by different components (`ExtAudit` + `Web` + `Core`). De-duplicate by `RequestId` + message — they are the same failure seen from different layers, not separate errors.
- **`ExtAudit` "ASYNC: Authentication error..." entries are noise** — recurring background authentication failures, usually unrelated to the incident. Check frequency/volume: a burst may still matter, but a steady trickle usually does not.
- The **`Web` entry (`WebAPI exception calling action X on controller Y`)** is the most useful for mapping an error to a user action. Use its `Action`/`Controller` to find the corresponding endpoint.
- The **`Core` structured block** gives the service + method + parameters + stack — use it to pinpoint the failing component (e.g. `SecurityContext.RequireUserObjectPermission` = permissions, `UserTaskService.Get` = tasks, `ProcessObjectService` = processes).
- Messages may contain **Russian text** — keep the original wording, do not paraphrase; translate only when summarizing.
- **Stack frames** reference platform assemblies (`Comindware.*`) and script DLLs under `/var/lib/comindware/<stand>/Database/Scripts/` (Linux) — a failure inside a script DLL means a user-defined script/rule caused it.
- `WARN` lines in this file are often rule/macro validation messages (`Invalid statement operation ...`) — generally harmless noise unless clustered.

**How to use for error analysis:**
1. Filter by the interview window (1.4) and category keywords (errors, exceptions).
2. Group by `RequestId` to reconstruct each failure's full picture across components.
3. Separate noise (`ExtAudit` ASYNC authentication, `WARN` rule-validation) from real failures.
4. Prioritize by: frequency in window, severity (`FATAL` > `ERROR` > `WARN`), affected user (`systemAccount` = background), and whether the same signature repeats.
5. Cross-reference with `performance_*.log` (did this error coincide with a slowdown?), `system_*.log` (what was happening), and nginx/IIS (HTTP status for the same request).
6. Match the `Web`/`Core` entries to identify the endpoint, service method, and the exact failing frame in the stack.

**When `error_*.log` is inconclusive but the platform crashed or hung:**

Some failures never leave a meaningful trace in `error_*.log`:
- a process killed externally (e.g. SIGKILL by the OOM-killer) does not get a chance to write an exception — `error_*.log` will be empty or contain only unrelated noise at that moment;
- a hang/leak shows up in memory metrics, not as exceptions.

In those cases look **beyond** `error_*.log`:

1. **`journal.log` (systemd journal / journalctl dump)** — find the service crash:
   - `Main process exited, code=killed, status=9/KILL` → SIGKILL (typically OOM-killer or manual kill). Correlate with OOM-killer entries (`oom-killer`, `Out of memory`, `killed process`).
   - `Failed with result 'signal'` / unit entered `failed` state → crash signal.
   - `SERVICE_STOP` with `res=failed`, then `SERVICE_START` with `res=success` → crash + automatic/scheduled restart.
   - `CRON ... systemctl start comindwarecmwdata` → a scheduled watchdog restart — record the restart cycle times.
   - Look for `oom-killer` / `Out of memory` / `Killed process ... total-vm` entries near the crash time — they confirm OOM.
2. **`igniteClient_*.log` (Apache Ignite embedded node)** — a crash/hang often originates in the in-memory DB:
   - `Heap [used=XMB, free=Y%]` and `Off-heap memory [used=..., allocated=...]` metric blocks — track memory growth before the crash (rising `used`, falling `free%` → leak/exhaustion).
   - a new `IgniteConfiguration [igniteInstanceName=<stand>, ..., nodeId=<new id>]` after a crash = node restart; count restarts to see the crash cycle.
   - `WARN` "Stack smashing detected", "Consistent ID is not set", "New version is available" — startup noise, ignore.
   - `Configured failure handler: StopNodeOrHaltFailureHandler` — node halts on failure; a halt is not logged in `error_*.log`.
3. **`system_*.log`** — last activity before the crash (what the platform was doing when it died).
4. **`status.log`** — services status at dump time (which service is up/down).

Pattern to remember: **no error in `error_*.log` + platform crash/hang = look at `journal.log` for SIGKILL/OOM and `igniteClient_*.log` for memory exhaustion / node restarts.**

### 2.2c Integrations and data exchange — analysis approach

Integration failures are rooted in the **adapter logs**, not the platform `error_*.log`. The user will likely report "обмен не работает" / "интеграция упала" / "OData не отвечает".

1. **Start with `adapter_internal_system_error_*.log`** — it has the exact route that failed:
   ```
   [ts][ERROR] PlatformKey: ... Procedure: <proc> (procedure.<n>). Endpoint: <ep> (endpoint.<n>). Adapter: <Type>  <message>
   ```
   Record the `Procedure`/`Endpoint`/`Adapter` — that names the failing integration route and the adapter type (`HttpRequestAdapter`, `HttpListenerAdapter`, `MySqlClient`, `MySqlListener`, etc.).
2. **Check `adapter_external_heartbeat_*.log`** — is the adapter host healthy? Look at "Загруженные адаптеры", "Созданные экземпляры путей передачи данных", process memory/thread stats. Empty/`Отсутствуют` instances = nothing deployed → integration can't work.
3. **Check `adapter_external_system_*.log`** — did the adapter host restart recently (`AdapterHostService::StopAsync` → `Host::Host` load config → `AdapterHostService::StartAsync`)? Correlate restart times with the reported failure. The loaded config shows MQ connection (`mq.server`, `mq.name`) — check it matches the Kafka config.
4. **For a specific connection**, look for `adapter_<connection-name>_*.log` if present; `integration_raw*` holds raw integration payloads (usually for OData) — use it to verify what was actually sent/received.
5. **Correlate with the message bus**: `kafkaClient_*.log` (adapter-side) and `kafka/` + `kafkaClient_*.log` in `<standname>_logs/` (platform-side) — if MQ is down, all adapters fail at once (look for a wall of adapter errors at the same timestamp).
6. **Common failure signatures**:
   - `Instance not started: ... Путь передачи данных «<name>» отключен` → data-transfer path is disabled (config/instance issue), not a network error;
   - a burst of the same adapter error right after an adapter-host restart → deployment/config problem;
   - adapter errors only for one `Endpoint` → that connection (external system side, credentials, endpoint address);
   - all adapter errors at the same moment → MQ/Kafka down or adapter host down (correlate with `journal.log` and Kafka logs).
7. **Check the other side**: for HTTP adapters, nginx `error.log` / `apigateway_*.log` show the inbound side; for the external system errors, look for connection-refused/timeout signatures in the adapter message text.

### 2.2d Authentication and licensing — analysis approach

Authentication and licensing problems are found **primarily in `error_*.log`** (not in dedicated log files). The relevant component names there are `Session`, `ExtAudit`, `Service`, `Core`.

1. **Login failures** — search `error_*.log` for:
   - `Login failed: unknown username, wrong password or account disabled.` — the standard login-rejection message (check `Web` component entry for the source IP; `ExtAudit` `Operation "POST" completed with an error: Login failed...` is the same event logged by another component);
   - `Password cannot be changed: directory server authentication mode is being used.` (`Session` component, stack in `AuthenticationService.ResetPasswordRequest` / `SessionIdentityService`) — password reset blocked because the account authenticates via an external directory (AD/LDAP);
   - component `Session` entries around `SessionIdentityService` — session creation/validation errors.
2. **License errors** — search for license keywords (English and Russian):
   - `licen`, `license`, `activation`, `expired`, `invalid token`, `лиценз`;
   - licensing often surfaces as DI-activation exceptions (`An exception was thrown while activating λ:Comindware.Platform.Api.Services.IThemeService -> ...`) or `Operation "GET" completed with an error` — trace the underlying cause in the stack, not the wrapper.
3. **Correlate with HTTP status codes**: `401`/`403` in `apigateway_*.log` and nginx/IIS access logs — a spike of `401` = mass login/session failures; `403` = permissions (see category "Errors": `У аккаунта ... нет разрешения ...`).
4. **Distinguish signal from noise**:
   - `ExtAudit` `ASYNC: "Authentication error occurred while processing the service request."` is recurring background noise — count them but do not treat a steady trickle as the incident;
   - a **burst** of `Login failed` from many different IPs in a short window = brute-force attempt;
   - repeated `Login failed` from the same IP = targeted attempt or misconfigured automation.
5. **Check the account state** in the failing records: `IsActive`, `IsDisabled`, `IsAnonymous`, `AuthenticationMethod` fields appear in some `Web`/DI-activation error payloads — they tell whether the account is active and how it authenticates (0 = internal, directory-based modes differ).
6. **For directory (AD/LDAP) auth problems** — AD-related data lives in two more places:
   - **`system_*.log`** — AD/directory activity (sync, authentication against the domain, user/group import) plus platform lifecycle; search for AD/LDAP/domain keywords (`ldap`, `active directory`, `домен`, `domain`, `forest`, directory server);
   - **`idm*` log** (e.g. `idm_*.log` or an `idm` component file) — identity-management service log with AD sync/authentication details; if present, check it alongside `system_*.log` for AD auth/sync errors. (If no `idm*` file exists in the dump, say so — it may be missing from the dump.)
   - Errors like "directory server authentication mode is being used" point at external directory config; correlate with directory-server availability in `system_*.log`/`journal.log`.

### 2.2e Database (Apache Ignite) — analysis approach

The platform's database is **Apache Ignite** (embedded node, log file `igniteClient_*.log`). There is no MongoDB on this stack. Ignite problems look like: slow data operations, checkpoints not completing, memory exhaustion, node restarts, or "database" errors surfacing in `error_*.log`/`system_*.log`.

1. **Memory pressure** — track `Heap [used=XMB, free=Y%]` and `Off-heap memory [used=..., allocated=...]` across the window:
   - rising `used` + falling `free%` toward a crash = leak/exhaustion → correlate with node restart and OOM;
   - `allocated` fixed but `free%` dropping = physical memory consumed by data regions.
2. **Regions** — `Persistent region [type=user, persistence=true, maxSize=...]` is the disk-backed data; `InMemory region` is volatile. Region `maxSize` values show the configured DB size limits.
3. **Checkpoints** — periodic checkpoint lines (`checkpointBeforeLockTime`, `checkpointLockHoldTime`); if checkpoints stop completing or "earliest reserved checkpoint" stays old for a long time → writes are stalled (disk slow/full, WAL issues).
4. **WAL/persistence** — from the config block: `walMode`, `walFsyncDelay`, `walSegmentSize`, `walPath`, `checkpointFreq` — gives the persistence behavior.
5. **Slow data operations** — correlate with `performance_*.log` `DataCommit`/`NativeQuery` and `error_*.log` database-related exceptions; Ignite stores the app data (caches like `cmwdatadatatdb_*` seen in `system_*.log`).
6. **Node health/restarts** — new `IgniteConfiguration ... nodeId=<new id>` blocks = node restarts; `StopNodeOrHaltFailureHandler` halts the node on failure.
7. **Data storage metrics** — `Data storage metrics for local node` blocks contain per-cache storage usage — use to see which cache grows.
8. **If MongoDB is present** (only on stacks that have it — `mongod.log`), apply the standard MongoDB markers; on this Linux stack it is Ignite instead.

### 2.2f Platform freezes / hangs (зависания) — analysis approach

When users report "платформа зависла" (freezes at specific times), the service is not crashed — it is **stuck**: requests pile up and time out (`throttled due to execution timeout - 00:00:30`, nginx `499`, `Соединение закрыто преждевременно` in `apigateway_*.log`). The root cause is usually **not visible in `system_*.log`/`error_*.log` alone** — you must also analyze **user behavior** (`audit_*.log`) and the exact operations in `performance_*.log`. Do not stop at the first long-duration system event.

1. **Build the degradation curve first** (`performance_*.log`). Produce a per-minute table of count / max / avg `Duration` in the window: the freeze is not instantaneous — operations degrade over minutes, then stall. Compute the **effective start** of each long operation as `Time − Duration`; the earliest such start in the window is the onset.
2. **Check whether the onset is global.** If several **unrelated users** (different `Account` values) have operations that start hanging in the **same second** — it is a systemic/global stall (memory/GC, lock, pool exhaustion), NOT a single user's action. One user opening a list or pressing a button cannot simultaneously stall 6–10 other users.
3. **Analyze user behavior in `audit_*.log`** (JSON lines: `time`, `sessionId`, `sessionUser`, `url`) in the minutes BEFORE the onset:
   - repeated opens of the same list — `ToolbarApi/GetListToolbar/oa.*/lst.N`, `PersonalDatasetConfigurationApi/lst.N` (e.g. tens of opens of one list per 15 min = auto-refresh/retry loop, a **load amplifier**);
   - form operations — `Dataform/QueryForm`, `PersonalFormApi/Put`;
   - button/command executions — `Records/Execute`, `UserCommandExecutionService.Execute`, `event.N` in `UserCommandConfigurationApi/GetOperationData`, which can run for minutes and hold locks.
   Heavy repeated operations that start before the degradation and continue through it are both the **precipitating load** and the **amplifier** (timeouts trigger browser/UI retries → more load → deeper stall).
4. **Do not read the long-duration list as a cause list.** A long `Heartbeat OnAggregateSensors` is a periodic background ticker that merely completes late when the platform is already frozen — a **symptom, never the root cause**. Same for a `system_*.log` stream of a single repeated `Get cache cmwdatadatatdb_*` message: it means a background thread is alive while request threads are blocked.
5. **Track memory.** `heartbeat_*.log` `PerformanceHelper.Perform` lines carry `TotalProcessMemory`/`TotalGCMemory`/`DeltaProcessMemory`; `status.log` carries process RSS at dump time; `igniteClient_*.log` `Heap [used=...]` tells whether the embedded DB is the memory hog or not. If .NET/mono memory grows **monotonically between service restarts** and freezes recur at high values → the freeze is memory-driven (GC stall on a huge heap); a service restart is only a band-aid that resets the counter.
6. **Correlate with `journal.log`**: service `Main process exited, code=killed, status=9/KILL` + restart cycles + `Consumed <N>h ... CPU time` (how long the process ran before it had to be killed). Repeated kill/restart cycles every 2–3 hours confirm a **recurring** freeze pattern.
7. **Recommendations**: a restart stabilizes temporarily but does NOT solve a recurring freeze. Always (a) snapshot the .NET/mono heap before the freeze to see what accumulates (caches/sessions/script assemblies); (b) review the heavy lists/events behind the identified `lst.*`/`event.*` operations (their datasets/queries and the scripts behind buttons); (c) review GC settings / cache limits / platform version. Treat the recurring kill cycle as evidence of an underlying growth problem, not as a fixable-by-restart issue.

### 2.3 Search methodology

Work systematically and record findings. **The main constraint is the period from the interview (1.4): read and analyze only the records inside the from–to window.**

1. **Filter files by period.** Before reading, determine which log files intersect the window:
   - files with the date in the name (`comindware_YYYY-MM-DD.log`, `u_exYYMMDD.log`, etc.) — read only those whose date overlaps the window, skip the rest;
   - files without a date in the name — check the first/last record to see whether the file intersects the window; if not, do not read it.
2. **Time summary.** Mind the time zone (state TZ both in logs and in the report). If the log TZ differs from the user's window TZ, convert.
3. **Targeted search instead of full reads.** Do not read whole files. First do a selective search within the window using key markers:
   - filter lines by the window's dates/times (Grep by time prefix, date range);
   - for large files, read only the found fragments via offset/limit, not the whole file;
   - key markers (each category has its own set):
     - errors: `ERROR`, `Exception`, `FATAL`, `Stack trace`, `Unhandled exception`;
     - performance: `timeout`, `timed out`, `slow`, `deadlock`, `lock`, `OOM`, `OutOfMemory`, `HTTP 500/503`, long `time-taken` in IIS, slow upstream in nginx (`upstream_response_time`), `slow query`; freezes/hangs additionally: `throttled due to execution timeout`, nginx `499`, `Соединение закрыто преждевременно`, repeated `Get cache`, long `TotalProcessMemory`/`TotalGCMemory` in `heartbeat_*.log`;
     - services: `Service stopped`, `Service started`, `restart`, `crash`, `exit code`, systemd `Main process exited`, `Failed to start`;
     - integrations: `connection refused`, `404/401/403` to external APIs, `webhook`, `retry`, `endpoint`, `kafka.*error`, `failed to connect`, `consumer group`;
     - authentication/licensing: `login failed`, `unauthorized`, `license`, `activation`, `expired`, `invalid token`, `401`, `403`;
     - database: on the Linux stack (Ignite) — `OutOfMemory`, `Heap`, `Off-heap`, `checkpoint`, `StopNodeOrHalt`, `nodeId` restarts; if MongoDB is present — `connection`, `replica`, `secondary`, `not primary`, `slow query`, `aborting`; `journalservice` health markers (`cluster_health`, `unassigned shards`, `red`/`yellow` status).
4. **Collect excerpts.** For each found event record:
   - exact timestamp (with TZ) — it must fall inside the window;
   - level and message text;
   - log file name and line number;
   - context: 2–4 lines before and after (the real cause is often in neighboring lines, not in the ERROR line itself).
5. **Rebuild the timeline.** Order the key events by time within the window. Cross-reference events across log types (platform components ↔ web server/IIS or nginx ↔ DB/journal ↔ system journal) to find the root cause — e.g., a service crash after `OutOfMemory`, followed by a wave of connection errors.
6. **Separate cause from effect.** The first exception in the window is often the cause; subsequent errors (`Can't connect`, `Service unavailable`) are consequences.

### 2.4 Completeness check

If the logs do not produce a coherent picture, check:
- whether the incident period is fully covered (the from–to window from 1.4, with no date gaps inside it);
- whether a log level was filtered out (e.g., only WARN/ERROR without the INFO context);
- whether relevant logs sit in subfolders you have not looked at.

If data is insufficient, tell the user exactly which logs are missing and for which period.

## Step 3. Report

Write a detailed report. Structure:

### 3.1 Summary (brief)
- Problem type (from the interview).
- Root cause (if established) in one or two sentences.
- Severity: critical / high / medium / low.
- Affected components (platform components, stand name, web server/nginx or IIS, DB, Kafka/Ignite if involved).

### 3.2 Incident timeline
A table of events in time order: `Time (TZ) | Source (file:line) | Level | Event`. Mark the "cause → effect" chains.

### 3.3 Analysis and log excerpts
For each significant event:
- what happened and why it matters;
- a log excerpt (code block) with timestamp, file, and line number;
- relation to other events.

Do not overload the report: include only meaningful excerpts; list the rest briefly.

### 3.4 Possible solutions
For each problem — concrete administrator actions, from simple to complex:
- immediate (quick stabilization: restart the service, free disk space, clear cache);
- mid-term (configuration: platform settings, IIS/DB limits, application configuration);
- strategic (architecture: scaling, version upgrade, custom development).
- For each solution — the expected effect and signs that it worked.

### 3.5 What to request when data is incomplete
If data was insufficient, list specifically which logs/metrics and for which period are needed to confirm the root cause.

## Step 4. Learn and improve (self-training)

The skill can improve itself: at any point, the user may ask to save what was learned during the current analysis. Record those lessons so future runs benefit from them.

### 4.1 Trigger phrases

Recognize requests like:
- "зафиксируй действия" / "зафиксируй" / "запомни это" / "сохрани в скилл"
- "record this" / "save to the skill" / "remember this action"
- or similar phrasing meaning "store what we just did so the skill improves".

When you hear this, do NOT just summarize. Commit the lesson to the skill files (see 4.3).

### 4.2 What to capture

From the current session, capture lessons that are reusable beyond this single incident:

1. **New root-cause patterns** — e.g., "crash preceded by OutOfMemory in service X" or "timeout caused by slow query pattern Y".
2. **Log markers / file locations** — specific log files, sections, or message signatures that turned out to be the smoking gun (e.g., a marker string in `comindware_*.log` that signals a licensing failure).
3. **Correlations** — cross-log correlations that were useful (platform ↔ IIS ↔ DB ↔ system).
4. **Fix verification** — signs that a fix worked, or traps to avoid (e.g., "do not trust ERROR alone, root cause is 2 lines above").
5. **New methodology** — a search step or technique that proved valuable.

Ignore incident-specific details (exact timestamps, user names, machine names, data values) — generalize the lesson instead.

### 4.3 Where to save

Lessons are appended to **`LEARNINGS.md`** in the skill directory (`/home/alesha/gh-repos/comindware-log-analysis/LEARNINGS.md`). Each entry:

```markdown
### YYYY-MM-DD — Short lesson title

- **Category**: performance | errors | service crashes | integrations | auth/licensing | database | methodology
- **Context**: when does this lesson apply
- **Lesson**: what was learned (generalized, no incident-specific data)
- **Evidence**: the log signature / marker / correlation that reveals it
- **Action**: what to do differently next time
```

Rules:
- Append new entries at the **end** of the file; keep existing entries untouched.
- If an identical lesson already exists, merge or skip — do not duplicate.
- If a lesson refines an existing one, update that entry instead of adding a new one.

### 4.4 Structural improvements (ask first)

If a lesson is big or recurring, you may also propose changes to `SKILL.md` itself (new markers, new step, better wording). **Never edit `SKILL.md` on your own — always propose the change and wait for the user's approval.** Small `LEARNINGS.md` entries do not require approval.

### 4.5 Using past lessons

Before starting a new analysis (in Step 2), if `LEARNINGS.md` exists in the skill directory, read it and apply relevant lessons: use the recorded markers, correlations, and methodologies from the start.

## Practical notes

- Work with files in the given directory using search tools (Glob/Grep/Read); do not copy logs anywhere unless necessary.
- Strictly limit the analysis to the from–to window from the interview: do not read records outside the period; skip log files that do not intersect the window.
- For large logs, use selective reads of found lines and time filtering instead of reading whole files. If a file is too large even for targeted reads, narrow the window or ask the user to provide an extract.
- If log files are not UTF-8 (e.g., Windows-1251), account for that when reading.
- Do not invent facts from the logs: back every conclusion in the report with an excerpt or mark it as "needs confirmation".
- Platform versions and specific messages may vary — if needed, confirm with the user which terms match their environment.
