# Asterisk.Sdk — Next Level Design Spec

**Autor:** Harol Reina
**Fecha:** 2026-03-16
**Estado:** Estable v1.5.0 — Fases 1-7 ✅ completas, Quality Sprint ✅ (SourceLink, analyzers, PublicAPI.Shipped.txt, 1,379 unit tests), PbxAdmin separado a repo propio (2026-03-24)
**Objetivo:** Definir la estrategia de evolución del SDK hacia una plataforma de telecomunicaciones líder con modelo open-core (MIT + PRO comercial).

---

## 1. Visión

Posicionar **Asterisk.Sdk** como:

> **The .NET Telephony Platform SDK for Asterisk**

Un SDK que permite construir contact centers, CPaaS, voice bots, IVR avanzados y plataformas SIP completamente en .NET, con rendimiento competitivo frente a implementaciones en Go y Java.

### Modelo de negocio: Open-Core

- **MIT (público, nuget.org):** Protocol clients, runtime state, session engine, telemetry, examples. Atrae adopción y comunidad.
- **PRO (comercial, feed privado):** Cluster orchestration, predictive dialer, advanced analytics, multi-tenant, event sourcing, skill-based routing. Genera ingresos.

### Dos repositorios

- `Asterisk.Sdk` — MIT, público en GitHub. Contiene todo el código open source.
- `Asterisk.Sdk.Pro` — Comercial, repo privado. Consume los paquetes MIT via NuGet. Código PRO nunca es público.

---

## 2. Auditoría del Estado Actual

### Production-Ready

| Capa | Componente | Evidencia |
|------|-----------|-----------|
| AMI | Connection, Protocol Parser, Event Pump, Correlation, Source Generators, Metrics, Thread Safety | Pipelines zero-copy, AmiStringPool, 4 source generators AOT, SemaphoreSlim write lock, ConcurrentDictionary correlation, 148 actions, 278 events, 633 tests |
| AGI | FastAgiServer, AgiChannel, Mapping Strategies, 54 Commands | Pipelines, 3 estrategias thread-safe, cobertura completa de comandos, AgiHostedService, 180 tests |
| ARI | AriClient, WebSocket, JSON, Audio Subsystem, 10 Resources | Source-generated JSON (~90 models), 46 event types, 94 endpoints, AudioSocket+WebSocket, reconnect con backoff, 306 tests |
| Live | AsteriskServer, ChannelManager, QueueManager, AgentManager, ServerPool | Dual indices O(1), ConcurrentDictionary, per-entity Lock, copy-on-write observers, reconnect+reload |
| VoiceAi | Audio, AudioSocket, VoiceAi, Stt, Tts, OpenAiRealtime, Testing | Polyphase FIR resampler, VAD, dual-loop pipeline, barge-in, 4 STT + 2 TTS providers, 223 tests |
| Core | Interfaces, Attributes, Enums, Exceptions | Zero reflection, async-first, clean hierarchy |
| Hosting | AddAsterisk(), Options Validation | DI correcto, [OptionsValidator] AOT-safe |
| Infra | Build, Packages, EditorConfig, Benchmarks, Quality Analyzers | TreatWarningsAsErrors, SourceLink, deterministic builds, PackageValidation, PublicAPI.Shipped.txt, 0 trim warnings |

### Needs Maturing

| Componente | Problema |
|------------|----------|
| ActivityBase | `CancelAsync` no cancela la operación real; sin guards de re-entrada; transiciones no atómicas |
| Activity Implementations | No capturan DIALSTATUS/QUEUESTATUS; argumentos incorrectos en Park/Queue; BlindTransfer simulado |
| LiveMetrics | Solo 6 gauges; sin counters de eventos, sin métricas por cola |
| Config | Semicolons naive, ConfigParseException no hereda AsteriskException, sin resolución #include |
| ARI Error Handling | AriNotFoundException/AriConflictException existen pero nunca se lanzan |

### Missing

| Componente | Impacto |
|------------|---------|
| Health Checks (IHealthCheck) | Crítico para K8s/producción |
| IConfiguration binding | Sin soporte appsettings.json |
| IHostedService para AMI/Live | Usuario maneja lifecycle manualmente (AGI ya tiene AgiHostedService como patrón a seguir) |
| AMI Heartbeat/Keepalive | No detecta conexiones TCP half-open |
| AGI Metrics | Inconsistente con AMI/ARI |
| DefaultEventTimeout en AMI | Event-generating actions cuelgan indefinidamente |
| Session Engine | No existe abstracción CallSession sobre channels/bridges |
| Examples | No hay ejemplos funcionales |

---

## 3. Arquitectura MIT

### Estructura del repo Asterisk.Sdk

```
Asterisk.Sdk (MIT — nuget.org)
│
├── Protocol Layer (existente, production-ready)
│   ├── Asterisk.Sdk.Ami
│   ├── Asterisk.Sdk.Ari
│   └── Asterisk.Sdk.Agi
│
├── Runtime Layer (existente, hardening requerido)
│   ├── Asterisk.Sdk.Live
│   └── Asterisk.Sdk.Activities
│
├── Domain Layer (NUEVO)
│   ├── Asterisk.Sdk.Sessions
│   └── Asterisk.Sdk.Telemetry
│
├── Infrastructure
│   ├── Asterisk.Sdk (core)
│   ├── Asterisk.Sdk.Config
│   └── Asterisk.Sdk.Hosting
│
└── Examples (NUEVO)
    ├── Asterisk.Sdk.Examples.SimpleCall
    ├── Asterisk.Sdk.Examples.Ivr
    └── Asterisk.Sdk.Examples.QueueRouting
```

### Hardening requerido

**Protocol fixes:**
- AMI: heartbeat/keepalive configurable (Ping cada 30s, reconnect on timeout)
- AMI: aplicar `DefaultEventTimeout` en `SendEventGeneratingActionAsync`
- AMI: fix observable gauge leak on reconnect
- AMI: fix drop counter (cambiar a DropWrite o remover dead code)
- ARI: usar `AriNotFoundException`/`AriConflictException` en todos los resources
- AGI: detección status 511 → `AgiHangupException`
- AGI: per-connection timeout configurable
- AGI: `AgiMetrics` (connections, scripts, errors)

**Activities rebuild:**
- `ActivityBase`: `CancellationTokenSource` wired a `ExecuteAsync`
- `ActivityBase`: re-entrance guard, transiciones atómicas
- `DialActivity`: captura `DIALSTATUS`
- `QueueActivity`: captura `QUEUESTATUS`, fix argument order
- `ParkActivity`: fix argument construction
- `HangupActivity`: pasar `CauseCode`

**Hosting + Telemetry + Config:**
- `IHostedService` para AMI connection + Live server lifecycle
- `IConfiguration` binding (`.BindConfiguration("Asterisk")`)
- `Asterisk.Sdk.Telemetry`: `AmiHealthCheck`, `AriHealthCheck`, `AgiHealthCheck`
- `Asterisk.Sdk.Telemetry`: OpenTelemetry `ActivitySource` para call lifecycle tracing
- `ConfigParseException` → hereda de `AsteriskException`
- `ConfigFileReader`: semicolons dentro de quoted values
- `LiveMetrics`: event counters, per-queue callers gauge, queue wait histogram

### Nuevos módulos MIT

**Asterisk.Sdk.Sessions:**
- `CallSession`: modelo con Participants, Channels, Bridge, State, Metadata
- `CallSessionState`: Created → Dialing → Ringing → Connected → OnHold → Transferring → Conference → Completed → Failed → TimedOut
- `CallSessionManager`: crea/actualiza/finaliza sesiones
- Correlación automática: channelId → CallSession, bridgeId → CallSession
- Event transformation: AMI events → domain events (CallStarted, CallConnected, CallTransferred, CallEnded)
- `IObservable<CallSessionEvent>` para PRO consumers
- `AgentSession`: estado extendido con current CallSession y statistics
- `QueueSession`: wait time tracking, SLA calculation
- **Reconciliación de sesiones huérfanas:** sweep periódico que verifica si los channels aún existen; sesiones sin channels activos se marcan como Failed
- **Timeout handling:** Dialing sin respuesta → TimedOut; network failure mid-call → Failed con metadata de causa
- **State transition matrix:** documentar transiciones válidas (e.g., Connected puede ir a OnHold, Transferring, Conference, Completed, Failed; pero Created no puede ir directamente a Completed)

**Event subscription strategy:** `CallSessionManager` se suscribe a los observables de `AsteriskServer` (managers de Live layer), NO a eventos raw AMI. Esto garantiza que Live ya ha procesado y enriquecido los eventos antes de que Sessions los consuma. El orden de dispatch es: AMI raw → Live managers → Session manager.

**Extension points (interfaces en `Sdk.Sessions`, NO en core — evita dependencia circular):**
- `ICallRouter` — default: single-node passthrough
- `IAgentSelector` — default: Asterisk native queue strategy
- `ISessionStore` — default: in-memory ConcurrentDictionary

**Interface evolution strategy:** Extension point interfaces usan abstract base classes con default implementations en lugar de interfaces puras. Esto permite agregar nuevos miembros en versiones futuras sin romper implementaciones PRO existentes. Ejemplo:
```csharp
public abstract class CallRouterBase
{
    public abstract ValueTask<string> SelectNodeAsync(CallSession session, CancellationToken ct);
    public virtual ValueTask<bool> CanRouteAsync(CallSession session, CancellationToken ct) => ValueTask.FromResult(true);
}
```

### Lo que NO va en MIT

- Event sourcing completo (replay, snapshots)
- Cluster orchestration (NodeRegistry, ClusterRouter, call distribution)
- Campaign engine / predictive dialer
- Advanced analytics (dashboards, reporting, ClickHouse)
- Multi-tenant isolation
- Skill-based routing avanzado

---

## 4. Arquitectura PRO

### Estructura del repo Asterisk.Sdk.Pro

```
Asterisk.Sdk.Pro (Comercial — GitHub Packages)
│
├── Cluster Orchestration
│   ├── Asterisk.Sdk.Pro.Cluster
│   └── Asterisk.Sdk.Pro.Cluster.Redis
│
├── Predictive Dialer
│   ├── Asterisk.Sdk.Pro.Dialer
│   └── Asterisk.Sdk.Pro.Dialer.Storage
│
├── Advanced Analytics
│   ├── Asterisk.Sdk.Pro.Analytics
│   └── Asterisk.Sdk.Pro.Analytics.Export
│
├── Enterprise Features
│   ├── Asterisk.Sdk.Pro.MultiTenant
│   ├── Asterisk.Sdk.Pro.EventStore
│   └── Asterisk.Sdk.Pro.Routing
│
└── Asterisk.Sdk.Pro.Hosting
```

### Cluster Orchestration (Pro.Cluster)

Consume `Asterisk.Sdk.Live` (AsteriskServerPool) como base.

```
ClusterManager
 ├── NodeRegistry          — Registro dinámico de nodos Asterisk
 ├── NodeHealthMonitor     — Health checks periódicos (AMI ping + SIP OPTIONS)
 ├── ClusterRouter         — Distribución de llamadas entre nodos
 │    ├── LeastLoadStrategy
 │    ├── RoundRobinStrategy
 │    └── GeographicStrategy
 ├── NodeDrainManager      — Drenar llamadas antes de apagar un nodo
 └── FailoverCoordinator   — Reubicar sesiones si un nodo cae
```

`Pro.Cluster.Redis`: Estado compartido via StackExchange.Redis (pub/sub + key-value) para múltiples instancias del control plane. StackExchange.Redis 2.8+ tiene soporte parcial de AOT/trimming — requiere spike de validación en Fase 3.

### Predictive Dialer (Pro.Dialer)

Componente genérico reutilizable para outbound dialing en contact centers.

```
CampaignEngine
 ├── CampaignManager       — CRUD, lifecycle (Created→Scheduled→Running→Paused→Completed→Archived)
 ├── ContactListManager    — Load/dedup/DNC check, cursor-based iteration
 ├── PacingEngine           — El cerebro del predictive
 │    ├── PredictiveAlgorithm  — Erlang-C, target abandonment rate (3-5%), adaptive
 │    ├── ProgressiveAlgorithm — 1:1 agent-to-call ratio
 │    └── PreviewAlgorithm     — Agent-initiated con preview window
 ├── RetryStrategy          — Configurable per disposition:
 │    busy → retry 5min, no-answer → retry 30min,
 │    voicemail → retry 2h, max 3 attempts, time window 9am-9pm
 ├── CallbackScheduler      — Agent callbacks + system callbacks, timezone-aware
 └── DispositionManager     — Call result → next action mapping
```

`Pro.Dialer.Storage`: Dapper + PostgreSQL, source-generated mappers, migrations.

### Advanced Analytics (Pro.Analytics)

```
AnalyticsEngine
 ├── RealTimeAggregator     — Rolling-window metrics (SLA%, ASA, AHT, abandonment rate)
 ├── CdrEnricher            — Enriquece CDR con datos de sesión, campaign, agent
 ├── IntervalReporter       — Snapshots cada N minutos (half-hour reporting estándar CC)
 └── AlertEngine            — Thresholds configurables → callbacks/webhooks
```

`Pro.Analytics.Export`: Prometheus metrics, ClickHouse bulk insert, Grafana annotations.

### Enterprise Features

- **Pro.MultiTenant**: `TenantContext` via `AsyncLocal<T>`, filtrado automático, métricas aisladas por tenant, routing rules por tenant
- **Pro.EventStore**: Event sourcing sobre CallSession — replay, snapshots, audit trail. PostgreSQL via Dapper
- **Pro.Routing**: Skill-based routing (skills + proficiency levels), priority queues, weighted distribution, overflow rules

### Dependencias PRO → MIT

```
Pro.Cluster     → Sdk.Live (AsteriskServerPool, AsteriskServer)
Pro.Dialer      → Sdk.Sessions (CallSession), Sdk.Ami (OriginateAction)
Pro.Analytics   → Sdk.Sessions (CallSession), Sdk.Telemetry (metrics)
Pro.EventStore  → Sdk.Sessions (CallSession, domain events)
Pro.MultiTenant → Sdk (core interfaces)
Pro.Routing     → Sdk.Live (QueueManager, AgentManager)
Pro.Hosting     → Sdk.Hosting (extends AddAsterisk())
```

Reglas: MIT nunca referencia PRO. PRO solo consume MIT via NuGet packages. Cada paquete PRO es independiente.

---

## 5. Paquetes NuGet

### MIT (nuget.org)

| Paquete | Dependencias |
|---------|-------------|
| `Asterisk.Sdk` | — |
| `Asterisk.Sdk.Ami` | `Sdk` |
| `Asterisk.Sdk.Agi` | `Sdk`, `Sdk.Ami` |
| `Asterisk.Sdk.Ari` | `Sdk` |
| `Asterisk.Sdk.Live` | `Sdk`, `Sdk.Ami` |
| `Asterisk.Sdk.Activities` | `Sdk`, `Sdk.Ami`, `Sdk.Agi`, `Sdk.Live` |
| `Asterisk.Sdk.Sessions` | `Sdk`, `Sdk.Ami`, `Sdk.Live` |
| `Asterisk.Sdk.Telemetry` | `Sdk` (OTel abstractions only) |
| `Asterisk.Sdk.Ami.HealthChecks` | `Sdk.Ami` (IHealthCheck for AMI) |
| `Asterisk.Sdk.Ari.HealthChecks` | `Sdk.Ari` (IHealthCheck for ARI) |
| `Asterisk.Sdk.Agi.HealthChecks` | `Sdk.Agi` (IHealthCheck for AGI) |
| `Asterisk.Sdk.Config` | `Sdk` |
| `Asterisk.Sdk.Hosting` | Meta-package: trae todos los anteriores. Para uso selectivo, referenciar paquetes individuales |
| `Asterisk.Sdk.Audio` ✅ | — (zero deps, pure C#) |
| `Asterisk.Sdk.VoiceAi` ✅ | `Sdk.VoiceAi.AudioSocket` |
| `Asterisk.Sdk.VoiceAi.AudioSocket` ✅ | `Sdk.Audio` |
| `Asterisk.Sdk.VoiceAi.Stt` ✅ | `Sdk.VoiceAi` |
| `Asterisk.Sdk.VoiceAi.Tts` ✅ | `Sdk.VoiceAi` |
| `Asterisk.Sdk.VoiceAi.OpenAiRealtime` ✅ | `Sdk.VoiceAi`, `Sdk.Audio` |
| `Asterisk.Sdk.VoiceAi.Testing` ✅ | `Sdk.VoiceAi` |

### PRO (GitHub Packages — feed privado)

| Paquete | Depende de (MIT) | Depende de (PRO) |
|---------|-----------------|-----------------|
| `Asterisk.Sdk.Pro.Cluster` | `Sdk.Live` | — |
| `Asterisk.Sdk.Pro.Cluster.Redis` | — | `Pro.Cluster` |
| `Asterisk.Sdk.Pro.Dialer` | `Sdk.Sessions`, `Sdk.Ami` | — |
| `Asterisk.Sdk.Pro.Dialer.Storage` | — | `Pro.Dialer` |
| `Asterisk.Sdk.Pro.Analytics` | `Sdk.Sessions`, `Sdk.Telemetry` | — |
| `Asterisk.Sdk.Pro.Analytics.Export` | — | `Pro.Analytics` |
| `Asterisk.Sdk.Pro.MultiTenant` | `Sdk` | — |
| `Asterisk.Sdk.Pro.EventStore` | `Sdk.Sessions` | — |
| `Asterisk.Sdk.Pro.Routing` | `Sdk.Live` | — |
| `Asterisk.Sdk.Pro.Hosting` | `Sdk.Hosting` | Todos Pro.* |

### Extension Points

All extension points live in `Asterisk.Sdk.Sessions` (NOT in core `Asterisk.Sdk`) to avoid circular dependencies — they reference domain types like `CallSession`, `AsteriskAgent`, `AsteriskQueue`.

Uses abstract base classes instead of interfaces for forward-compatible evolution:

```csharp
// Asterisk.Sdk.Sessions (MIT) — abstract classes que PRO extiende
public abstract class CallRouterBase
{
    public abstract ValueTask<string> SelectNodeAsync(CallSession session, CancellationToken ct);
    public virtual ValueTask<bool> CanRouteAsync(CallSession session, CancellationToken ct)
        => ValueTask.FromResult(true);
}

public abstract class AgentSelectorBase
{
    public abstract ValueTask<AsteriskAgent?> SelectAgentAsync(AsteriskQueue queue, CancellationToken ct);
    public virtual ValueTask<IReadOnlyList<AsteriskAgent>> RankAgentsAsync(
        AsteriskQueue queue, IReadOnlyList<AsteriskAgent> candidates, CancellationToken ct)
        => ValueTask.FromResult(candidates);
}

public abstract class SessionStoreBase
{
    public abstract ValueTask SaveAsync(CallSession session, CancellationToken ct);
    public abstract ValueTask<CallSession?> GetAsync(string sessionId, CancellationToken ct);
    public virtual ValueTask DeleteAsync(string sessionId, CancellationToken ct)
        => ValueTask.CompletedTask;
}

// Asterisk.Sdk.Sessions (MIT) — observable events que PRO consume
public interface ICallSessionManager
{
    IObservable<CallSessionEvent> Events { get; }
    IObservable<CallSession> SessionCreated { get; }
}
```

PRO registra implementaciones via DI:
```csharp
services.AddAsteriskPro(options => {
    options.UseCluster(cluster => cluster.UseRedis("redis:6379"));
    options.UseDialer(dialer => dialer.UsePostgres("connstring"));
    options.UseAnalytics(analytics => analytics.UseClickHouse("connstring"));
});
```

MIT funciona completamente solo con implementaciones default (single-node, in-memory, round-robin).

---

## 6. Roadmap

### Fase 1: Hardening MIT (mes 0–2)

**Objetivo:** Convertir lo existente en production-hardened. Publicar `v0.2.0-beta`.

**Sprint 1 (2 sem) — Protocol fixes:** ✅ Completado
- ✅ AMI heartbeat/keepalive configurable
- ✅ AMI DefaultEventTimeout en SendEventGeneratingActionAsync
- ✅ AMI fix observable gauge leak on reconnect
- ✅ AMI fix drop counter
- ✅ ARI domain exceptions en resources
- ✅ AGI status 511 detection
- ✅ AGI per-connection timeout
- ✅ AGI AgiMetrics

**Sprint 2 (2 sem) — Activities rebuild:** ✅ Completado
- ✅ ActivityBase cancel real con CancellationTokenSource
- ✅ Re-entrance guard, transiciones atómicas
- ✅ DialActivity captura DIALSTATUS
- ✅ QueueActivity captura QUEUESTATUS, fix args
- ✅ ParkActivity fix args
- ✅ HangupActivity pasar CauseCode
- ✅ Tests: failure, cancellation, concurrency

**Sprint 3 (2 sem) — Hosting + Telemetry + Config:** ✅ Completado
- ✅ IHostedService para AMI/Live lifecycle
- ✅ IConfiguration binding (AOT-safe manual binding en AddAsterisk(IConfiguration))
- ✅ Health checks: AmiHealthCheck, AgiHealthCheck, AriHealthCheck (en sus respectivos proyectos)
- ✅ OTel ActivitySource: AmiActivitySource, AgiActivitySource, AriActivitySource (distributed tracing)
- ✅ ConfigParseException hierarchy fix
- ✅ ConfigFileReader semicolons en quotes
- ✅ LiveMetrics expansion

**Sprint 4 (2 sem) — Examples + Release:** ✅ Completado
- ✅ 10 examples funcionales (superó objetivo de 3)
- ✅ README renovado
- ✅ Integration tests skip graceful
- ✅ Publicado v0.2.0-beta en nuget.org

### Fase 2: Session Engine MIT (mes 2–4)

**Objetivo:** Construir Asterisk.Sdk.Sessions. Publicar `v0.5.0-beta`.

**Sprint 5-6 (4 sem) — CallSession core:** ✅ Completado
- ✅ CallSession modelo + state machine
- ✅ CallSessionManager + correlación automática
- ✅ AMI events → domain events transformation
- ✅ IObservable<CallSessionEvent>
- ✅ Wire LinkEvent/BridgeEnterEvent en Live EventObserver

**Sprint 7-8 (4 sem) — AgentSession + extension points:** ✅ Completado
- ✅ AgentSession con current CallSession
- ✅ QueueSession con SLA calculation
- ✅ ICallRouter, IAgentSelector, ISessionStore interfaces
- ✅ Integration tests con Docker
- ✅ Benchmarks
- ✅ Publicar v0.5.0-beta

**Pre-Sprint 9 (2026-03-18) — Prerequisitos PRO:** ✅ Completado
- ✅ StackExchange.Redis v2.12.1 AOT spike validado (0 trim warnings, 9.3 MB binary)
- ✅ AddAsteriskSessionsMultiServer() API (reemplaza hack de PbxAdmin)
- ✅ 10 integration tests de hardening (heartbeat, health checks AMI/ARI/AGI, live server)
- ✅ 864 tests passing, 0 warnings

### Fase 3: PRO Foundation (mes 4–8)

**Objetivo:** Crear repo Pro. Entregar Cluster + Dialer + EventStore. Publicar `v0.1.0-pro-beta`.

**Sprint 9-10 (4 sem) — Repo setup + Cluster:** ✅ Completado (2026-03-18)
- ✅ Repo Asterisk.Sdk.Pro — Directory.Build.props, Directory.Packages.props, global.json, .editorconfig, nuget.config
- ✅ Pro.Cluster: ClusterManager, NodeRegistry, NodeHealthMonitor, ClusterRouter (weighted strategies, priority tiers), DrainManager, FailoverCoordinator, SessionReconstructor
- ✅ Pro.Cluster.Redis: RedisClusterTransport, ClusterEventSerializer (polymorphic JSON), Lua-scripted locking
- ✅ ClusterTransportBase + InMemoryClusterTransport (testing)
- ✅ 108 tests (82 Cluster + 26 Redis), 0 warnings
- ✅ MIT SDK: RegisterReconstructedSession, AddExistingServer (commit 6a37d34)

**Sprint 11-14 (8 sem) — Predictive Dialer:** ✅ Completado (2026-03-19)
- ✅ Pro.Dialer (64 files, 3,095 LOC): DialerEngine (BackgroundService), 5 activation strategies (EventDriven, TimerDriven, Preview, RateDriven), 3 pacing engines (ErlangC, FixedRatio, Formula)
- ✅ 14 abstract bases (CampaignStore, ContactProvider, OriginateBuilder, OriginateExecutor, RetryStrategy, CallResultHandler, CallResultNotifier, DncChecker, OutboundRouteResolver, DialStringBuilder, CallerIdResolver, HolidayCalendarProvider, PacingEngine, IActivationStrategy)
- ✅ Full compliance pipeline: ComplianceContext (schedule + blend + DNC + frequency + timezone + holiday)
- ✅ OriginateGate (Interlocked CAS, global + per-campaign backpressure), ContactBuffer (Channel<T>, exponential backoff)
- ✅ DefaultOriginateExecutor (circuit breaker E1: 5 failures → 30s), CallbackScheduler (persist-first C3, PriorityQueue + Lock T2)
- ✅ CampaignOperations (clone, TTL, recycling, penetration tracking)
- ✅ Pro.Dialer.Storage.Postgres (9 files, 678 LOC): 7 PostgreSQL implementations (Dapper + Npgsql), 15-table schema, FOR UPDATE SKIP LOCKED
- ✅ 195 tests (194 Dialer + 1 Postgres), 45 commits, 0 warnings
- ✅ MIT SDK: SelectNodeForOriginateAsync (commit 9f61bef), AsteriskServer.Connection (commit df01081)

**Sprint 15 (2 sem) — EventStore:** ✅ Completado (2026-03-19)
- ✅ Pro.EventStore (14 files, 679 LOC): EventStoreSubscriber (IHostedService), SessionCompletionProjector (CDR builder with hold-time pairing), EventSerializer (AOT-safe, 9 event types, TimeSpan↔double ms)
- ✅ Pro.EventStore.Postgres (4 files, 389 LOC): PostgresSessionEventStore, PostgresCompletedSessionStore, partitioned session_events + flat completed_sessions schema
- ✅ 22 tests (21 EventStore + 1 Postgres), 9 commits, 0 warnings
- ✅ Architecture: Event Log + Completion Projection (2 layers — JSONB events for audit/replay, flat CDR for reports/billing)

**Fase 3 Totales:**
- 6 paquetes NuGet: Pro.Cluster, Pro.Cluster.Redis, Pro.Dialer, Pro.Dialer.Storage.Postgres, Pro.EventStore, Pro.EventStore.Postgres
- 130 source files, 8,244 LOC source, 51 test files, 6,179 LOC test = **14,423 LOC total**
- **325 tests**, 63 commits, 0 warnings, AOT-compatible (IsAotCompatible=true, EnableTrimAnalyzer, EnableAotAnalyzer)
- Package version: 0.1.0-beta.1

### Fase 4: PRO Advanced (mes 8–12)

**Objetivo:** Analytics, MultiTenant, Routing. Publicar `v1.0.0` MIT + `v1.0.0-pro`.

**Sprint 17-18 (4 sem) — Analytics:** ✅ Completado (2026-03-19)
- ✅ Pro.Analytics: RealTimeAggregator (in-memory per-queue IntervalBucket, 21 Must-Have CC metrics), LiveStateProvider
- ✅ AlertEngine (threshold callbacks con cooldown), IntervalReporter (clock-aligned 30min snapshots)
- ✅ AnalyticsQueryService (live + historical read API), CdrEnricher, CampaignSnapshotCollector, RestartRecovery
- ✅ Pro.Analytics.Export: PrometheusExporter (7 ObservableGauge instruments, zero-cost)
- ✅ Pro.Analytics.Storage.Postgres: PostgresIntervalSnapshotStore (3 tables: interval_snapshots, agent_snapshots, campaign_snapshots)
- ✅ MIT SDK: CallWrapUpEvent + CallRingNoAnswerEvent (2 new SessionDomainEvent types)
- ✅ 53 tests (52 Analytics + 1 Postgres), 16 commits, 0 warnings

**Sprint 19 (2 sem) — MultiTenant:** ✅ Completado (2026-03-19)
- ✅ Pro.MultiTenant: TenantContext (AsyncLocal), TenantScope (IDisposable), TenantGuard
- ✅ ITenantResolver + 3 defaults (DID, Context, Header), ITenantStore, ITenantLifecycleHandler
- ✅ **BREAKING CHANGE 0.2.0-beta:** All abstract bases (7 Dialer + 2 EventStore + 1 Analytics) gained `string tenantId` as first parameter
- ✅ All 10 PostgreSQL store implementations updated with tenant_id
- ✅ OriginateGate: 3-level CAS (global → tenant → campaign)
- ✅ 65 files changed, all 380 existing tests updated for backward compat (tenantId="")
- ✅ 17 tests (11 core + 6 isolation), 10 commits

**Sprint 20 (2 sem) — Routing:** ✅ Completado (2026-03-19)
- ✅ Pro.Routing: SkillMatcher (inverted index, O(N) AND-logic matching, proficiency scoring)
- ✅ 3 abstract bases (SkillCatalogBase, AgentScorerBase, RoutingMiddlewareBase)
- ✅ RoutingEngine with composable middleware pipeline (4 built-in: SkillMatching, Priority, Occupancy, Overflow)
- ✅ 2 scorers: ProficiencyWeightedScorer + OccupancyAdjustedScorer (fairness)
- ✅ QueueMembershipController (AMI QueueAdd/Remove/SetPenalty), OverflowTimer (per-call bullseye)
- ✅ SkillBasedAgentSelector (extends MIT AgentSelectorBase)
- ✅ RoutingMetrics (7 instruments), InMemorySkillCatalog
- ✅ 36 tests, 7 commits, 0 warnings

**Fase 4 Totales:**
- 5 paquetes nuevos: Pro.Analytics, Pro.Analytics.Export, Pro.Analytics.Storage.Postgres, Pro.MultiTenant, Pro.Routing
- **11 paquetes totales** en Pro repo (+ 6 de Fase 3)
- **433 tests**, 94 commits, 0 warnings, versión 0.2.0-beta
- 1,311 tests MIT + Pro combinados

### Fase 5: Voice AI (mes 12–18)

**Objetivo:** Integración nativa de AI en telefonía. AudioSocket transport, STT/TTS abstracto con providers pluggables, VoiceAi pipeline con turn-taking e interrupciones, OpenAI Realtime bridge, Agent Assist en tiempo real.

**Sprint 21-22 (4 sem) — Audio + AudioSocket + Core Pipeline:** ✅ Completado (2026-03-19)
- ✅ `Asterisk.Sdk.Audio` — pure C# polyphase FIR resampler (12 rate pairs, zero-alloc), PCM16↔float32, gain, RMS energy, silence detection (VAD). Sin dependencias externas. 52 tests.
- ✅ `Asterisk.Sdk.VoiceAi.AudioSocket` — AudioSocket server/client con System.IO.Pipelines, `AudioSocketSession` (bidirectional PCM streaming, backpressure), `AudioSocketClient` (testing sin Asterisk real). 29 tests.
- ✅ `Asterisk.Sdk.VoiceAi` — core abstractions: `ISessionHandler`, `VoiceAiPipeline` (dual-loop: AudioMonitorLoop + PipelineLoop, barge-in via volatile CancellationTokenSource), `VoiceAiSessionBroker` (IHostedService), `IConversationHandler`, `SpeechRecognizer`/`SpeechSynthesizer` abstract bases. 19 tests.
- ✅ `Asterisk.Sdk.VoiceAi.Testing` — FakeSpeechRecognizer, FakeSpeechSynthesizer, FakeConversationHandler. 10 tests.
- ✅ 110 tests totales, 0 warnings, AOT-compatible.

**Sprint 23 (2 sem) — STT + TTS Providers:** ✅ Completado (2026-03-19, merge c3325a2)
- ✅ `Asterisk.Sdk.VoiceAi.Stt` — `DeepgramSpeechRecognizer` (WebSocket streaming), `WhisperSpeechRecognizer`, `AzureWhisperSpeechRecognizer`, `GoogleSpeechRecognizer`. DI: `AddDeepgramSpeechRecognizer()` / `AddWhisperSpeechRecognizer()` / etc. 21 tests.
- ✅ `Asterisk.Sdk.VoiceAi.Tts` — `ElevenLabsSpeechSynthesizer` (WebSocket streaming), `AzureTtsSpeechSynthesizer`. DI: `AddElevenLabsSpeechSynthesizer()` / `AddAzureTtsSpeechSynthesizer()`. 12 tests.
- ✅ `VoiceAiExample` — E2E console app (Deepgram + ElevenLabs + EchoConversationHandler).
- ✅ 62 tests nuevos (21 Stt + 12 Tts + 19 VoiceAi + 10 Testing), 70 archivos nuevos, 0 warnings.

**Sprint 24 (2 sem) — OpenAI Realtime + Virtual Agent Demo:** ✅ Completado (2026-03-20, merge 18ffd5e)
- ✅ `Asterisk.Sdk.VoiceAi.OpenAiRealtime` — `OpenAiRealtimeBridge` (dual-loop: InputLoop 8kHz→24kHz→OpenAI, OutputLoop OpenAI→24kHz→8kHz→Asterisk).
- ✅ `ISessionHandler` abstraction — `VoiceAiPipeline` y `OpenAiRealtimeBridge` intercambiables vía DI. `VoiceAiSessionBroker` inyecta `ISessionHandler`.
- ✅ Function calling: `IRealtimeFunctionHandler` (Name, Description, ParametersSchema, ExecuteAsync) + `RealtimeFunctionRegistry` + `AddFunction<T>()`.
- ✅ Observabilidad: 7 `RealtimeEvent` records via `IObservable<RealtimeEvent>` (speech started/stopped, transcript, response started/ended, function called, error).
- ✅ DI: `AddOpenAiRealtimeBridge()` + `AddFunction<THandler>()`. `VoiceAiSessionBroker` registrado como hosted service.
- ✅ `RealtimeFakeServer` (in-process HttpListener WebSocket) para tests de integración.
- ✅ `OpenAiRealtimeExample` demo — `GetCurrentTimeFunction`, appsettings.json.
- ✅ 24 tests (6 bridge integration + 3 function call bridge + 3 registry unit + 4 DI + 4 JSON/events), 0 warnings, AOT-compatible.

**Sprint 25-26 (4 sem) — Pro.AgentAssist:** ✅ Completado (2026-03-20, tag v0.3.0-beta, branch feature/pro-agent-assist)
- ✅ `Asterisk.Sdk.Pro.AgentAssist` — real-time AI agent assistance during calls. 30 tests.
  - `AgentAssistEngine` (BackgroundService): event subscription, `PassesFilter` (QueueNames/AgentIds/TenantIds), creates `AgentAssistSession` per call
  - `AgentAssistSession` (IAsyncDisposable): ARI dual-stream snoop (`ISnoopChannelPair`), `DualStreamTranscriber` (2 `SpeechRecognizer` instances, `.Synchronize()` for thread safety), 4 `IObservable<T>` streams (Transcripts, Suggestions, Sentiment, ComplianceAlerts), `Task.WhenAll` parallel provider runners with timeouts
  - `WhisperQueue` (bounded Channel, DropOldest, Critical priority→CTS preemption via `Interlocked.Exchange`) + `WhisperDelivery` (TTS→AudioSocket streaming loop)
  - `AgentAssistSupervisor`: `ConcurrentDictionary<string, AgentAssistSession>`, `IObservable<AgentAssistSession>` SessionStarted/SessionEnded
  - `AgentAssistMetrics`: 8 instruments (Counter, Histogram, ObservableGauge), `CreateForTesting()`
  - Pluggable providers: `SuggestionProviderBase`, `SentimentAnalyzerBase`, `ComplianceMonitorBase` + keyword-based implementations
  - DI: `AddProAgentAssist(builder => ...)` with typed builder, eager options validation
- ✅ `Asterisk.Sdk.Pro.AgentAssist.Storage.Postgres` — 3 Dapper stores, 3 tables (`agent_assist_sessions`, `suggestion_log`, `compliance_alerts`), migration SQL `001_AgentAssist_Schema.sql`. 15 integration tests (require PostgreSQL).
- ✅ 45 tests total, 25 commits, 0 warnings, AOT-compatible

**Sprint 27 (2 sem) — Pro.CallAnalytics AI:** 📦 Gestionado en repo `Asterisk.Sdk.Pro`
- `Asterisk.Sdk.Pro.CallAnalytics` — post-call AI processing
- LLM-powered call summarization, automated QA scoring, compliance flags

**Fase 5 Paquetes (entregados a 2026-03-20):**
- MIT ✅: `Asterisk.Sdk.Audio`, `Asterisk.Sdk.VoiceAi`, `Asterisk.Sdk.VoiceAi.AudioSocket`, `Asterisk.Sdk.VoiceAi.Stt`, `Asterisk.Sdk.VoiceAi.Tts`, `Asterisk.Sdk.VoiceAi.OpenAiRealtime`, `Asterisk.Sdk.VoiceAi.Testing`
- PRO ✅: `Pro.AgentAssist` + `Pro.AgentAssist.Storage.Postgres` — completados 2026-03-20 (45 tests, tag v0.3.0-beta)
- PRO 📦: `Pro.CallAnalytics` — gestionado en repo Pro
- Entregados MIT: 7 paquetes VoiceAi nuevos

### Fase 6: Polish + v1.0 (mes 18–20)

**Objetivo:** Estabilizar API pública, publicar documentación oficial y benchmarks, implementar mecanismo de licenciamiento PRO, publicar `v1.0.0` MIT y `v1.0.0-pro` PRO.

**MIT v1.0.0:** ✅ Completado (2026-03-21)
- API review, CHANGELOG.md, SECURITY.md, per-package READMEs, 12 examples
- Publicado en nuget.org (17 paquetes)

**v1.5.0 Quality Sprint:** ✅ Completado (2026-03-24)
- Code quality analyzers (Layers 1-3: Roslyn, Threading, API surface)
- PublicAPI.Shipped.txt populated con API surface v1.5.0
- SourceLink, deterministic builds, PackageValidation baseline
- Coverage push: Ami 82%, Agi 86%, Live 81.6%, Ari 79.4%
- 1,379 unit tests + 640 functional tests
- PbxAdmin separado a repo propio (`Asterisk.Sdk.PbxAdmin`)

**PRO v1.0.0-pro:** 📦 Gestionado en repo `Asterisk.Sdk.Pro`
- API review, Docker E2E integration tests, licensing mechanism

### Release Timeline

| Versión | Contenido | Timeline |
|---------|-----------|----------|
| `v0.2.0-beta` MIT | Hardening, health checks, examples | Mes 2 | ✅ |
| `v0.5.0-beta` MIT | Session Engine, extension points | Mes 4 | ✅ Publicado 2026-03-17 |
| `v0.1.0-pro-beta` PRO | Cluster, Dialer, EventStore | Mes 8 | ✅ Completado 2026-03-19 (6 paquetes, 325 tests) |
| `v0.2.0-beta` PRO | Analytics, MultiTenant, Routing | Mes 10 | ✅ Completado 2026-03-19 (11 paquetes, 433 tests) |
| `v0.6.0-beta` MIT | VoiceAi core, AudioSocket, STT/TTS, OpenAI Realtime | Mes 14 | ✅ Completado 2026-03-20 (Sprint 21-24, 7 paquetes, ~196 tests) |
| `v0.6.0-pro-beta` PRO | AgentAssist, CallAnalytics AI | Mes 17 | ⏳ Parcial: AgentAssist ✅ (Sprint 25-26, v0.3.0-beta), CallAnalytics ⏳ (Sprint 27) |
| `v1.0.0` MIT | API estable, docs, benchmarks | Mes 20 | ✅ Publicado 2026-03-21 |
| `v1.1.0` MIT | Asterisk 22-23 compat, 3 AMI actions, ARI bridges | — | ✅ Publicado 2026-03-22 |
| `v1.2.0` MIT | Sprint A: AMI PJSIP + Bridge + Transfer (20 actions) | — | ✅ Publicado 2026-03-22 |
| `v1.3.0` MIT | Sprint B: ARI Completeness (26 endpoints, 11 models, 2 new resources) | — | ✅ Publicado 2026-03-22 |
| `v1.4.0` MIT | Sprint C: AMI Misc + AudioSocket Ast23 (11 actions + 8 high-rate types) | — | ✅ Publicado 2026-03-22 |
| `v1.5.0` MIT | Quality Sprint: analyzers, SourceLink, PublicAPI, coverage push | — | ✅ Publicado 2026-03-24 |
| `v1.0.0-pro` PRO | API review, integration tests, licensing | — | 📦 Gestionado en repo Pro |

### Fase 7: API Completeness — Asterisk 18-23 Full Coverage (post-v1.0)

**Objetivo:** Cubrir el 100% de la API pública de Asterisk (AMI Actions, ARI Endpoints, ARI Models). Compatibilidad explícita con Asterisk 18, 20, 22 y 23. Plan completo: `docs/superpowers/plans/2026-03-22-api-completeness-plan.md`

**Gap Analysis (actualizado 2026-03-26):**

| Capa | Asterisk | SDK v1.5.0 | Cobertura |
|------|----------|-----------|-----------|
| AMI Actions | 152 | 148 | 97% |
| AMI Events | 180 | 278 | 154% (legacy+compat) |
| AGI Commands | 47 | 54 | 100%+ |
| ARI Endpoints | 98 | 94 | 96% |
| ARI Events | 46 | 46 | 100% |
| ARI Models | 27 | 27 | 100% |

**Sprint A (v1.2.0) — AMI PJSIP + Bridge + Transfer:** ✅ Completado (2026-03-22)
- 11 PJSIP actions (ShowAors, ShowAuths, ShowRegistrations, Register, Unregister, Qualify, Hangup)
- 7 Bridge management actions (Destroy, Info, Kick, List, TechnologyList/Suspend/Unsuspend)
- 2 Transfer actions (BlindTransfer, CancelAtxfer)
- 1 event field: `TechCause` en Hangup events (Asterisk 23)
- ~10 new/updated response events para event-generating actions

**Sprint B (v1.3.0) — ARI Completeness:** ✅ Completado (2026-03-22)
- 3 new resource classes: AriAsteriskResource (16 endpoints), AriMailboxesResource (4), AriEventsResource (1)
- Complete existing: Channels (+7), Bridges (+6), Endpoints (+5), Applications (+3), Recordings (+9)
- 11 new ARI models (AsteriskInfo, Module, LogChannel, Mailbox, RtpStats, etc.)
- Update IAriClient interface with new resource properties
- ARI Asterisk 23 params: `announcer_format`, `recorder_format`

**Sprint C (v1.4.0) — AMI Misc + Asterisk 23 Specifics:** ✅ Completado (2026-03-22)
- 2 Voicemail actions (VoicemailRefresh, VoicemailUserStatus)
- 2 Presence actions (PresenceState, PresenceStateList)
- 2 Queue actions (QueueReload, QueueRule)
- 4 Misc actions (CoreShowChannelMap, SendFlash, DialplanExtensionAdd/Remove, DbGetTree)
- AudioSocket high sample rate message types 0x11-0x18 (Asterisk 23: slin12-slin192)

**Not prioritized (hardware legacy <5% of deployments):**
- DAHDI (8 actions) — TDM hardware only
- PRI (4 actions) — E1/T1 hardware only
- IAX (3 actions) — Obsolete protocol
- Sorcery Cache (5 actions) — Internal admin
- FAX (3 actions), MeetMe (2), AOCMessage, WaitEvent

**Post-completion coverage: ~97% of all Asterisk 23 public APIs.**

### PbxAdmin: Call Flow & UX Improvements (post Fase 7)

**Objetivo:** Unificar la visualización de flujo de llamadas, mejorar la legibilidad del dialplan y rutas outbound, agregar cross-references entre entidades, y proveer un debugger de dialplan visual. Spec: `docs/superpowers/specs/2026-03-23-call-flow-ux-design.md`

**Phase 1 — Foundation:** ✅ Completado (2026-03-23)
- CallFlowService (graph building, health warnings P1, cross-references, cache)
- DialplanHumanizer, DialPatternHumanizer, NumberManipulator helpers
- `/call-flow` page: overview dashboard + two-panel inbound flow
- Nav reorganization: Call Flow en PBX Management, Dialplan movido a System

**Phase 2 — Call Tracer:** ✅ Completado (2026-03-23)
- Call Tracer con date/time picker + override mode selector (Live/None/AllOpen/AllClosed)
- Debugger step-through con Inspect dialplan lines por paso
- Asterisk pattern matching (_NXZ. syntax)
- `/dialplan` mejorado: badges de tipo por contexto, humanización de apps, links bidireccionales

**Phase 3 — Routes & Cross-refs:** ✅ Completado (2026-03-23)
- Outbound routes UX: pattern humanizer, trunk health dots, failover labels, manipulation preview
- Cross-references en TC, IVR ("Usado por: ...", "Referenciado por: ...")
- Inline flow summary en inbound routes
- Warning "Not referenced" en entidades huérfanas

**PbxAdmin — Separado a repo propio (2026-03-24):**

> PbxAdmin se gestiona desde `github.com/Harol-Reina/Asterisk.Sdk.PbxAdmin`. Consume SDK v1.5.0 via NuGet.
> Las fases Future (v1, v2, v3) se planifican y ejecutan desde ese repositorio.

---

## 7. Métricas de Éxito

### MIT (actualizado 2026-03-26)
- ✅ Publicado en nuget.org: v0.2.0-beta → v0.5.0-beta → v1.0.0 → v1.1.0 → v1.2.0 → v1.3.0 → v1.4.0 → v1.5.0
- ✅ 17 paquetes NuGet (9 Core + 7 VoiceAi + 1 SourceGenerators)
- ✅ 12 examples funcionales (superó objetivo de 3)
- ✅ 0 trim warnings AOT, 0 build warnings
- ✅ Quality tooling: SourceLink, deterministic builds, PackageValidation, PublicAPI.Shipped.txt, code analyzers (Layers 1-3)
- ⏳ Benchmarks públicos competitivos vs Go/Java
- ✅ Session Engine funcional con 9 domain events + IObservable
- ✅ Voice AI stack completo: AudioSocket + STT/TTS providers + OpenAI Realtime bridge (7 paquetes, Sprint 21-24)
- ✅ 1,379 unit tests MIT + 640 functional tests (2026-03-26)
- ✅ API Completeness: ~97% de APIs Asterisk 18-23 (148 AMI actions, 278 events, 94 ARI endpoints, 46 ARI event types)
- ✅ PbxAdmin separado a repo propio (2026-03-24), consume SDK v1.5.0 via NuGet

### PRO (gestionado en repo `Asterisk.Sdk.Pro`)
- ✅ Predictive dialer operativo con Erlang-C pacing (5 modos, 3 engines)
- ✅ Cluster support multi-nodo (weighted routing, failover, drain)
- ✅ EventStore con replay funcional (event log + CDR projection)
- ✅ Analytics en tiempo real (21 métricas CC, alertas, snapshots, Prometheus)
- ✅ MultiTenant con AsyncLocal isolation
- ✅ Skill-based routing (inverted index, priority, overflow, fairness scorers)
- ✅ AgentAssist operativo (dual-stream STT, suggestions, sentiment, compliance whisper — Sprint 25-26, 45 tests, v0.3.0-beta)
- 📦 CallAnalytics AI, v1.0.0-pro release, legacy migration — pendientes en repo Pro

---

## 8. Compatibilidad AOT de Dependencias PRO

| Dependencia | Módulo PRO | AOT Status | Notas |
|-------------|-----------|------------|-------|
| Npgsql 9.0.3 | Dialer.Storage, EventStore.Postgres | ✅ Validado | NpgsqlDataSource.Create(), 0 trim warnings |
| Dapper 2.1.66 | Dialer.Storage, EventStore.Postgres | ✅ Validado | Explicit AS aliases (no reflection mapping), 0 warnings |
| StackExchange.Redis 2.12.1 | Cluster.Redis | ✅ Validado | 0 trim warnings, 9.3 MB binary (spike Pre-Sprint 9) |
| ClickHouse.Client | Analytics.Export | Desconocido | Spike necesario en Sprint 17; fallback: HTTP bulk insert directo |
| OpenTelemetry .NET | Telemetry | Soportado | Fully AOT-compatible desde 1.7+ |

Cada dependencia con status "Parcial" o "Desconocido" tiene un spike task asignado en el roadmap antes de su uso.

---

## 9. Estrategia de Migración desde Dialers Legacy

Pro.Dialer proporciona 14 abstract bases que permiten migrar cualquier dialer legacy:

- **Fase 1:** Implementar abstract bases apuntando al storage existente (MySQL, CRM, etc.)
- **Fase 2:** Ejecutar en paralelo — legacy + Pro.Dialer procesando el mismo tráfico
- **Fase 3:** Cutover completo, legacy archivado
- **Gate de migración:** Pro.Dialer debe pasar todos los integration tests del dialer legacy antes de cutover

Los abstract bases cubren todos los puntos de extensión: `CampaignStoreBase`, `ContactProviderBase`, `OriginateBuilderBase`, `CallResultHandlerBase`, `DncCheckerBase`, `PacingEngineBase`, etc. — permitiendo reemplazar storage, CRM integration, dial string format, y lógica de retry sin modificar el engine core.

---

## 10. Licensing PRO

**Mecanismo:** NuGet feed authentication (GitHub Packages con token) como primera barrera. Sin DRM runtime.

**Justificación:** Una validación runtime de licencia complicaría AOT (strings embebidos, reflection para checks), añadiría latencia en hot paths, y los clientes enterprise prefieren trust-based licensing sobre DRM intrusivo. El modelo GitLab EE (código privado + contrato) es el estándar para open-core B2B.

**Evolución futura:** Si el volumen de clientes lo justifica, agregar license key validation en `AddAsteriskPro()` (startup-only check, no hot path). Nunca validar en runtime durante llamadas.

---

## 11. Nota sobre Activities

`Asterisk.Sdk.Activities` se marca como `[Experimental]` en v0.2.0-beta. Los cambios de Sprint 2 (cancel real, transiciones atómicas, result capture) son breaking changes respecto a la API actual. El atributo `[Experimental]` advierte a los consumidores y se removerá en v0.5.0-beta cuando la API se estabilice.

---

## 12. Riesgos y Mitigaciones

| Riesgo | Mitigación |
|--------|-----------|
| Session Engine MIT es demasiado poderoso → canibaliza PRO | Las sesiones MIT son in-memory, sin replay, sin persistencia. PRO agrega durabilidad y escala |
| Complejidad del predictive dialer | Erlang-C es un modelo matemático probado. Empezar con progressive (simple) y agregar predictive incrementalmente |
| Adopción lenta | Examples + benchmarks + docs son la prioridad. Un SDK sin docs no se adopta |
| Breaking changes en MIT rompen PRO | Semantic versioning estricto. PRO CI builds contra MIT latest |
| Scope creep en PRO | Cada módulo es independiente. Se puede entregar Cluster sin Dialer |
| AOT incompatibilidad en dependencias PRO | Spike tasks en roadmap antes de cada integración. Fallbacks definidos (HTTP directo para ClickHouse) |
| Breaking changes en extension points rompen PRO | Abstract base classes con virtual methods (no interfaces). Nuevos métodos se agregan con default impl |
| Legacy dialer coexistiendo con Pro.Dialer | Feature freeze en legacy. Gate de migración con integration tests |
