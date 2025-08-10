## Port Plan: Center Server/Service to Python

### Objectives
- Reimplement the center node in Python with feature parity:
  - TCP hub for shard connections and cross‑server broadcasts.
  - Admin/service API replacing WCF.
  - Timers/jobs (DB saves, scans, notices, world events).
  - DB access to load/update server list and related ops.

### Architecture Choices
- **TCP hub**: asyncio (streams) with a structured binary protocol mirroring current packet codes.
- **Admin API**: FastAPI (HTTP/JSON) or gRPC for typed APIs.
  - If legacy WCF clients must remain, provide a thin .NET bridge that forwards WCF calls to the Python API.
- **Config/logging**: pydantic‑settings + YAML/TOML; logging via structlog or loguru.
- **Database**: SQLAlchemy + async driver (pymssql/pytds). Consider a future move to PostgreSQL if desired.

### Component Mapping
- **CenterServer (C#)** → `asyncio` orchestrator:
  - Socket listener on IP/Port for shards.
  - Startup: load config, init managers, start timers/tasks.
- **ServerClient** → per‑connection handler:
  - RSA handshake (cryptography) and same packet codes.
  - Forward/broadcast helpers.
- **LoginMgr** → in‑memory session registry:
  - Async locks; single‑session by id/name; kick via packet.
- **ServerMgr** → DB‑backed server list manager:
  - Load/refresh lists; compute state from online/total; periodic persistence.
- **WorldMgr** → world events/boss/rank:
  - Counters, timers, rank map, notices loader.
- **CenterService (WCF)** → Python Admin API:
  - Endpoints for CreatePlayer, KitoffUser, GetServerList, Reload, AAS/DailyAward flags, MailNotice, ValidateLoginAndGetID, etc.

### Protocol and Contracts
- Preserve packet codes and payload formats (examples):
  - 0=RSA key, 1=server login, 3=allow user login, 4=user offline batch, 6=query state, 7=AAS, 10=item strengthen,
    11=reload, 12=ping, 72–73=big bugle, 80–89=world boss ops, 117=mail notice, 128/130 consortia ops,
    178=macro drop, 180–186 consortia boss updates, 240=IP/Port.
- Implement Python packet abstraction identical to `GSPacketIn/Out` (endianness, string/binary framing).
- Admin API schemas mirror `ICenterService` operations; expose as REST (OpenAPI) or gRPC.

### Proposed Project Layout (Python)
```
center_py/
  app_config.py           # pydantic settings
  main.py                 # bootstrap and run
  networking/
    server.py             # asyncio TCP listener
    client.py             # per-connection handler (ServerClient)
    packets.py            # encode/decode, opcodes
    rsa.py                # handshake helpers
  managers/
    login_manager.py
    server_manager.py
    world_manager.py
    macro_drop_manager.py
  services/
    api.py                # FastAPI/gRPC
    models.py             # pydantic schemas
  data/
    db.py                 # SQLAlchemy engine/session
    repos.py              # queries matching ServiceBussiness usage
  jobs/
    timers.py             # DB save, logs, scans, notices, world events
  i18n/
    language.txt, system_notice.xml loader
  tests/
    unit/, integration/, protocol/
```

### Milestones
- **M0: Discovery/spec**
  - Enumerate all `ICenterService` methods and payloads; list packet codes/fields; extract DB schema used by `ServiceBussiness`.
  - Deliverable: protocol + API spec document.
- **M1: Core runtime**
  - Config, logging, DB connection, models; skeleton FastAPI/gRPC with health endpoints.
- **M2: TCP hub + handshake**
  - Async TCP server; RSA exchange; minimal packet loop; ping/state update (code 12).
- **M3: Session management**
  - Implement CreatePlayer, TryLoginPlayer, login/offline flows and kicks; ValidateLoginAndGetID.
- **M4: Server list**
  - Start/ReLoad/SaveToDatabase parity; GetServerList API.
- **M5: Broadcasts and feature flags**
  - System notices, mail notice, reload/config flags, battleground/league toggles.
- **M6: World/consortia flows**
  - World boss counters/rank; consortia boss update handlers.
- **M7: Timers/jobs**
  - Save DB, record logs, mail/auction/consortia scans, notices, world events.
- **M8: API parity**
  - Complete all admin endpoints; optional auth.
- **M9: Compatibility/adoption**
  - Option 1: Update in‑repo clients to call new API.
  - Option 2: Ship a .NET shim that exposes original WCF and forwards to Python.
- **M10: Hardening**
  - Load testing, backpressure, graceful shutdown, metrics, structured logs.
- **M11: Packaging**
  - Dockerfile, docker‑compose, k8s manifests; CI pipeline.

### Risks and Mitigations
- Legacy WCF clients: provide a shim or refactor clients to HTTP/gRPC.
- Binary protocol mismatches: build golden tests comparing to C# behavior; replicate `PacketIn` exactly.
- DB drivers on Apple Silicon: prefer Dockerized SQL Server; validate pymssql/pytds; fallback shim for DB if needed.

### Testing Strategy
- Unit tests for packet encode/decode and manager logic.
- Integration tests against SQL Server in Docker.
- End‑to‑end: Python center + mock shard exercising login/ping/broadcast and admin API calls.

### Deployment
- Single container exposing:
  - TCP port (center hub) and HTTP/gRPC port (admin API).
- Config via env vars; mount language/notice files as needed.

### Initial Task List (MVP ~2–3 weeks)
- Week 1: Spec, config/logging/DB skeleton, FastAPI skeleton, TCP listener + RSA, packet abstraction.
- Week 2: Login/Server managers, GetServerList, CreatePlayer flow, ping/state updates, notices, reload/flags.
- Week 3: Timers/jobs, world boss basics, Dockerization, docs and sample client.

### Deliverables
- Python repo with the structure above.
- OpenAPI (or .proto) for the admin service.
- Migration guide for replacing WCF calls or using the .NET bridge.
- Docker scripts to run on macOS M1 (including SQL Server container).


