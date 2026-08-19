# Comindware Log Analysis — LEARNINGS

Lessons captured from past analyses. This file is read by the skill before each new analysis (Step 2.1) and is updated when the user asks to record actions ("зафиксируй действия").

## How to add an entry

Append a new entry at the end of this file (do not edit existing ones):

```markdown
### YYYY-MM-DD — Short lesson title

- **Category**: performance | errors | service crashes | integrations | auth/licensing | database | methodology
- **Context**: when does this lesson apply
- **Lesson**: what was learned (generalized, no incident-specific data)
- **Evidence**: the log signature / marker / correlation that reveals it
- **Action**: what to do differently next time
```

Rules:
- Append at the end; keep existing entries untouched.
- If an identical lesson exists, merge or skip — do not duplicate.
- If a lesson refines an existing one, update that entry instead of adding a new one.
- Small entries need no approval; changes to `SKILL.md` always require user approval.

### 2026-07-29/30 — Structure of `performance_*.log` (confirmed from real incident dump)

- **Category**: performance
- **Context**: performance analysis on any Linux stand (`performance_YYYY-MM-DD.log` in `<standname>_logs/`)
- **Lesson**: each line is a JSON record; `Duration` is in seconds; top-level transactions have `Parent: null`, children reference the root via `Origin` and their parent via `Parent`; `Count` aggregates N identical calls into one record. **The slow transaction itself is a symptom, not the root cause: analyze the situation BEFORE and AFTER it.** A preceding heavy operation is usually the trigger — e.g. a `DataCommit` with a huge `ChangesCount` (5000+ objects), or a form/table open with heavy `Expression` filtering (`formRule.*`/`user.rule.*`/`bruleDef.*`), after which everything becomes slow. Check whether the slowdown persists on subsequent lightweight operations (systemic effect — lock, pool exhaustion, memory) or was a one-off.
- **Evidence**: real dump: `WebRequest /DatasetData/Query` → `DatasetService.QueryData` → `DatasetConfigurationService.Get` → `CardConfigurationService.GetAttachedCard` / `DataSourceService.GetInfo` / `ToolbarService.GetAttachedToolbar`; `DataCommit` has `TxId`/`ChangesCount`/`TxParent`; `NativeQuery` has `SqlQueryText`; `TriggerAction` has `Kind`; `Token` tracks workflow moves (`PID`/`TID`/`PSA`/`Info`); `Expression` has `ExpressionId` (`formRule.*`, `user.rule.*`, `bruleDef.*`).
- **Action**: for performance problems read `performance_*.log` first: find slow roots (seconds+), drill into children by `Origin`, then scan the same window for heavy operations just BEFORE the slowdown (`DataCommit` with high `ChangesCount`, `Expression`-heavy form/table requests, `TriggerAction`/`Token` bursts, heavy `NativeQuery`) and check operations AFTER to judge whether the effect is systemic. Correlate with `error_*.log`/`system_*.log`/DB logs. Respect `Count` aggregation, convert TZ to the interview window.

### 2026-07-29/30 — Structure of `error_*.log` (confirmed from real incident dump)

- **Category**: errors
- **Context**: error analysis on any stand (`error_YYYY-MM-DD.log` in `<standname>_logs/`)
- **Lesson**: line format is `<Date> <Time> <LEVEL> <RequestId> <Account> <Host> <Port> <ClientIP> <Component> <Elapsed> <Version> '<message>'`. One event is logged multiple times by different components (`ExtAudit` + `Web` + `Core`) with the same `RequestId` — de-duplicate, it's one failure. `ExtAudit` "ASYNC: Authentication error..." is recurring noise, not an incident signal. The `Web` entry (`WebAPI exception calling action X on controller Y`) maps the error to an endpoint; the `Core` structured block (`Service name:`/`Method name:`/`Parameters list:`/`Stack:`) pinpoints the service + method + stack. `00000000-...` RequestId = background task; `systemAccount` = system/background user. Stack frames in `/var/lib/comindware/<stand>/Database/Scripts/*.dll` = user-defined script/rule failure. `WARN` `Invalid statement operation` lines are rule-validation noise.
- **Evidence**: `SecurityContext.RequireUserObjectPermission` = permission denial (`У аккаунта ... нет разрешения ...`); `UserTaskService.Get`, `ProcessObjectService.GetReferencedTasks` = tasks/process errors; `Login failed: unknown username, wrong password or account disabled` = auth; `Could not load file or assembly '<hash>'` = script DLL load failure (`CalculateScript failed`).
- **Action**: for error analysis filter by window, group by `RequestId`, strip noise (`ExtAudit` ASYNC auth, `WARN` rule-validation), prioritize by frequency/severity/affected user, then cross-reference with `performance_*.log`, `system_*.log`, and nginx/IIS for the same window.

### 2026-07-29/30 — Inconclusive `error_*.log` + crash/hang → look in `journal.log` and `igniteClient_*.log` (confirmed from real incident dump)

- **Category**: errors, service crashes
- **Context**: platform crashes/hangs where `error_*.log` has no meaningful trace
- **Lesson**: externally killed processes (SIGKILL, OOM-killer) do not write exceptions, so `error_*.log` is empty/noisy at that moment; hangs/leaks show up in memory metrics instead. When `error_*.log` is inconclusive but the platform fell/hung: check `journal.log` for `Main process exited, code=killed, status=9/KILL`, `Failed with result 'signal'`, `oom-killer`/`Out of memory`/`Killed process`, and `SERVICE_STOP res=failed` → `SERVICE_START res=success` restart cycles (incl. `CRON ... systemctl start` watchdog restarts); check `igniteClient_*.log` for `Heap [used=XMB, free=Y%]`/`Off-heap memory` growth before the crash and new `IgniteConfiguration ... nodeId=<new>` blocks (= node restarts). `StopNodeOrHaltFailureHandler` means the Ignite node halts on failure without logging to `error_*.log`.
- **Evidence**: real dump: `comindwarecmwdata.service: Main process exited, code=killed, status=9/KILL` + `Failed with result 'signal'` at 06:01 and 15:13 (29th) and 06:01, 11:10 (30th); `CRON (root) CMD (systemctl start comindwarecmwdata adapterhostcmwdata)` at 06:15 daily; `igniteClient` `Heap [used=...MB, free=...%]` metric blocks + repeated `IgniteConfiguration [igniteInstanceName=cmwdata, ..., nodeId=...]` blocks = multiple node restarts; `ignite/no_ignite_logs.log` = dedicated Ignite log dir absent.
- **Action**: for crashes/hangs always include `journal.log` and `igniteClient_*.log` in the analysis even if the user only reports an error; trace memory growth in igniteClient metrics over the window; correlate SIGKILL times with oom-killer entries and with `system_*.log` last activity.

### 2026-07-30 — Adapter/integration logs structure and analysis approach

- **Category**: integrations
- **Context**: integration and data-exchange problems (обмен не работает, OData)
- **Lesson**: integration failures live in adapter logs, not in platform `error_*.log`. Logs: `adapter_internal_system_*.log` (+ `_error_` = endpoint/procedure errors), `adapter_external_system_*.log` (adapter host lifecycle: `AdapterHostService::StopAsync` → `Host::Host` config load → `StartAsync`), `adapter_external_heartbeat_*.log` (health: loaded adapters, data paths, process memory/threads), `adapter_external_kafkaClient_*.log` (MQ client), `adapter_<connection-name>_*.log` (per-connection), `integration_raw*` (raw payload, usually OData). Error line format: `[ts][ERROR] PlatformKey: ... Procedure: <proc> (procedure.<n>). Endpoint: <ep> (endpoint.<n>). Adapter: <Type>  <message>` — the route triple identifies the failing integration. `Instance not started: ... Путь передачи данных «<name>» отключен` = disabled path, not network.
- **Evidence**: real dump: `adapter_internal_system_error_2026-07-30.log` with `MySqlClient`/`MySqlListener`/`HttpListenerAdapter`/`HttpRequestAdapter` errors `Instance not started ... Путь передачи данных «...» отключен`; `adapter_external_system_2026-07-30.log` shows `AdapterHostService::StopAsync` at 06:01 then `Host::Host` load config + `StartAsync` at 06:15 (restart cycle).
- **Action**: for integration problems start from `adapter_internal_system_error_*.log`, extract Procedure/Endpoint/Adapter, verify adapter host health in `adapter_external_heartbeat_*.log`, correlate restarts with `adapter_external_system_*.log`, check MQ (kafkaClient/Kafka) when all adapters fail at once, and use `integration_raw*`/`adapter_<connection>*` for connection-specific and OData issues.

### 2026-07-29/30 — Auth and licensing: where to look

- **Category**: auth/licensing
- **Context**: login problems, licensing errors, AD/LDAP issues
- **Lesson**: authentication and licensing problems are found primarily in `error_*.log` (components `Session`, `ExtAudit`, `Service`, `Core`), not in dedicated log files. AD/directory data lives in `system_*.log` and, when present, in an `idm*` log (identity-management service). `Login failed: unknown username, wrong password or account disabled.` is the standard rejection (`Web` entry shows source IP; `ExtAudit` `Operation "POST" completed with an error: Login failed...` is the same event). `Password cannot be changed: directory server authentication mode is being used.` (`Session` component, `AuthenticationService.ResetPasswordRequest`) = AD/LDAP-managed account. License errors appear as DI-activation exceptions or `Operation "GET" completed with an error` — trace the underlying cause, not the wrapper. `ExtAudit` ASYNC authentication noise must not be mistaken for an incident; a burst of `Login failed` from many IPs = brute-force. Account state fields (`IsActive`, `IsDisabled`, `IsAnonymous`, `AuthenticationMethod`) appear in some Web/DI-activation payloads.
- **Evidence**: real dump: 18+14 `Login failed` entries; repeated `Password cannot be changed: directory server authentication mode is being used.` in `Session` component; DI-activation exceptions with account JSON (`IsSystemAdministrator`, `IsActive`, `IsDisabled`, `AuthenticationMethod`, `FullName`); AD keywords present in `system_*.log`; no `idm*` file in this particular dump (may be absent from dump).
- **Action**: for auth/licensing start in `error_*.log`, filter by login/license keywords, de-duplicate `ExtAudit`+`Web` same-event entries, cross-check `401`/`403` in apigateway/nginx, then open `system_*.log` and `idm*` for AD/LDAP sync and authentication details.

### 2026-07-29/30 — Database = Apache Ignite (`igniteClient_*.log`)

- **Category**: database
- **Context**: database problems on the Linux stack
- **Lesson**: the platform database is the embedded Apache Ignite node — there is no MongoDB on this stack. `igniteClient_*.log` holds DB info: `Data storage metrics for local node`, `Heap`/`Off-heap memory`, data regions (`Persistent region [persistence=true]` = disk-backed, `InMemory region` = volatile, `Default_Region`), checkpoint lines (`checkpointBeforeLockTime`, `checkpointLockHoldTime`; stale "earliest reserved checkpoint" = stalled writes), WAL/persistence config (`walMode`, `walFsyncDelay`, `walSegmentSize`, `checkpointFreq`). Node restarts show as new `IgniteConfiguration ... nodeId=<new>`; `StopNodeOrHaltFailureHandler` halts the node on failure. App data lives in caches like `cmwdatadatatdb_*` (seen in `system_*.log`).
- **Evidence**: real dump: 407 checkpoint lines/day; regions `Persistent`, `InMemory`, `Default_Region`, `metastoreMemPlc`, `volatileDsMemPlc`; `Off-heap memory [used=7881MB, free=81.92%, allocated=43373MB]`; cache names `cmwdatadatatdb_80003B6A000044AD` in `system_*.log`.
- **Action**: for database problems analyze `igniteClient_*.log`: memory growth across window (`Heap`/`Off-heap` used vs free%), region limits, checkpoint completion (stalled = disk/WAL issue), node restarts, and correlate with `performance_*.log` `DataCommit`/`NativeQuery` and `system_*.log` cache accesses.

### 2026-07-30 — Platform freezes/hangs: analyze user behavior, not just system logs

- **Category**: performance, methodology
- **Context**: users report "платформа зависла" at specific times; the service is not crashed but stuck (requests time out, nginx 499, `Соединение закрыто преждевременно`)
- **Lesson**: the root cause of a freeze is usually NOT visible in `system_*.log`/`error_*.log` alone. First build the degradation curve from `performance_*.log` (per-minute count/max/avg `Duration`; the freeze is gradual, not instantaneous). Compute each long operation's effective start as `Time − Duration` to find the onset. Then check whether the onset is **global**: if several unrelated users' operations start hanging in the same second → systemic/global stall (memory/GC), NOT a single user's action. Then analyze **user behavior** in `audit_*.log` in the minutes BEFORE the onset: repeated opens of the same list (`ToolbarApi/GetListToolbar/oa.*/lst.N`, `PersonalDatasetConfigurationApi/lst.N`), form ops (`QueryForm`, `PersonalFormApi/Put`), button executions (`Records/Execute`, `UserCommandExecutionService.Execute`, `event.N` in `UserCommandConfigurationApi/GetOperationData`). Long `Heartbeat OnAggregateSensors` and a stream of a single repeated `Get cache ...` in `system_*.log` are SYMPTOMS (periodic ticker completing late / background thread alive while request threads blocked), never the cause. If .NET/mono `TotalProcessMemory` grows monotonically between service restarts and freezes recur at high values → memory-driven GC stall; restart is only a band-aid.
- **Evidence**: real dump: all requests (Dataform/QueryData, DatasetData/Query, UserCommandExecutionService.Execute) degraded 100–600 s at ~10:34:30, ~12:36, ~16:02 with several users' operations starting in the same second; `audit_*.log` showed repeated `lst.3730` opens (80+ by one user, 60 in 30 min), `lst.9264` (26 in 15 min), heavy `PersonalFormApi/Put` and `event.17309`/`event.23962` executions right at the onsets; `Heartbeat OnAggregateSensors` 631 s at 10:40 was merely a ticker completing late; `TotalProcessMemory` grew 26→33→37 GB between restarts; RSS 66 GB at dump; service killed (`status=9/KILL`) at 11:10 and 14:26 after ~4.5–5 h CPU.
- **Action**: for freeze/hang reports do NOT stop at the first long-duration system event. Always: (1) build the per-minute degradation curve and the onset via `Time − Duration`; (2) check if the onset is global (same second, multiple unrelated users) — if yes, it is resource-driven, not a trigger operation; (3) correlate the onset with user activity in `audit_*.log` (`lst.*`/`form*`/`event.*`) to find the precipitating load and amplifiers; (4) track memory across restarts (`TotalProcessMemory`, RSS, Ignite Heap) to determine whether it is a recurring growth problem; (5) in recommendations, state explicitly that restart is a temporary band-aid for a recurring freeze and identify what to snapshot/measure to find the actual accumulating resource.




