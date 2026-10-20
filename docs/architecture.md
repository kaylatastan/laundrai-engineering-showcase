# Architecture

This document describes LaundrAI as implemented in the private source. Paths refer to that source tree; they are cited for traceability and are not included in this repository.

## Components

| Component | Private source location | Responsibility |
| --- | --- | --- |
| Flutter Web client | `lib/` | UI, Firebase sign-in, live Firestore reads, backend calls for every reservation write and chat message |
| Firebase Authentication | external service | Email/password accounts; issues ID tokens |
| Cloud Firestore | external service | `machines`, `users`, `reservations`, `reservationRequests` |
| FastAPI backend | `backend_service/app/` | Token verification, reservation transactions, agent hosting |
| LangChain agent | `backend_service/app/agent/` | Tool-calling agent on Groq (`llama-3.3-70b-versatile`) |
| Sensor / simulator | not part of the application | Writes machine occupancy (`status`) to `machines` |

## System Overview

```mermaid
flowchart LR
    subgraph Client
        F[Flutter Web]
    end
    subgraph Firebase
        A[Firebase Auth]
        DB[(Cloud Firestore)]
    end
    subgraph Backend[FastAPI backend]
        D[get_current_user<br/>verify_id_token]
        R["POST /reservations/reserve<br/>POST /reservations/cancel"]
        C["POST /chat"]
        AG[AgentExecutor]
        T[build_tools: get_available_machines, reserve_machine]
        S[FirebaseService transactions]
    end
    LLM[Groq: Llama 3.3 70B]
    SIM[External sensor / simulator]

    F -->|sign in| A
    A -->|ID token| F
    F -->|direct reads| DB
    F -->|write own users/uid| DB
    F -->|Bearer token| D
    D --> R
    D --> C
    C --> AG
    AG <-->|tool calls| LLM
    AG --> T
    T --> S
    R --> S
    S -->|Admin SDK transactions| DB
    SIM -.->|status| DB
```

## Read and Write Paths

The design separates a fast, read-only path from a narrow, authoritative write path.

```mermaid
flowchart TB
    subgraph Reads["Read path (client → Firestore)"]
        R1["machines — snapshots()"]
        R2["reservations where userId == uid — snapshots()"]
        R3["users/{uid} — snapshots()"]
    end
    subgraph Writes["Write path"]
        W1["users/{uid} profile — client"]
        W2["reserve / cancel — backend transaction only"]
    end
    F[Flutter client] --> Reads
    F --> W1
    F -->|Bearer ID token| API[FastAPI]
    API --> W2
```

- **Direct reads.** The client subscribes with `snapshots()` to `machines` (dashboard, machines and schedule screens), to `reservations` filtered by `userId == <uid>` (history, schedule and dashboard), and to its own `users/{uid}` document (profile screen and the app-level `UserNameNotifier`).
- **Client writes.** Only the user's own `users/{uid}`: at sign-up (`lib/services/auth_service.dart`) and on profile edit (`lib/screens/profile_screen.dart`).
- **Reservation writes.** Never written by the client. `lib/services/reservation_service.dart` sends every reserve and cancel request to the backend with a fresh Firebase ID token.

## Data Model

| Collection | Written by | Fields used by the code |
| --- | --- | --- |
| `machines/{machineId}` | backend (reservations), external simulator (`status`) | `status` (`idle`, `reserved`, `running`, `maintenance`, `error`), `name`, `location`, `reservedBy`, `reservedByUid`, `activeReservationId`, `startTime` |
| `reservations/{reservationId}` | backend | `machineId`, `userId`, `userName`, `status` (`active` / `cancelled`), `reservedAt`, `cancelledAt`, `completedAt` |
| `users/{uid}` | client (profile), backend (counter) | `fullName`, `email`, `dormitory`, `note`, `activeReservationCount` |
| `reservationRequests/{sha256}` | backend | `userUid`, `machineId`, `reservationId`, `result`, `createdAt` |

```mermaid
erDiagram
    USERS ||--o{ RESERVATIONS : "userId"
    MACHINES ||--o| RESERVATIONS : "activeReservationId"
    USERS ||--o{ RESERVATION_REQUESTS : "userUid"
    RESERVATION_REQUESTS ||--|| RESERVATIONS : "reservationId"
    USERS {
        string fullName
        int activeReservationCount "cache, recomputed by backend"
    }
    MACHINES {
        string status "idle | reserved | running | maintenance | error"
        string reservedByUid
        string activeReservationId
        timestamp startTime
    }
    RESERVATIONS {
        string machineId
        string userId
        string status "active | cancelled"
        timestamp reservedAt
    }
    RESERVATION_REQUESTS {
        string machineId
        map result "replayed on retry"
        timestamp createdAt
    }
```

Invariants the code relies on:
- A machine is available only when `status == "idle"`. The UI derives every displayed state from `status` (`lib/models/machine_view.dart`).
- On a reserved machine, `activeReservationId` points to exactly one `reservations` document, and `reservedByUid` names its owner.
- `activeReservationCount` is a cache. The backend recomputes it from machine documents on every reserve and cancel rather than incrementing it.

## Authentication and UID Binding

```mermaid
sequenceDiagram
    participant F as Flutter
    participant A as Firebase Auth
    participant API as FastAPI
    F->>A: signInWithEmailAndPassword
    A-->>F: ID token
    F->>API: POST /reservations/reserve<br/>Authorization: Bearer ID token
    API->>A: verify_id_token(token)
    A-->>API: claims {uid, ...}
    Note over API: uid from claims only.<br/>Body user_uid is ignored.
    API->>API: reserve_machine(machine_id, uid, ...)
```

- `get_current_user` (`backend_service/app/dependencies.py`) verifies the bearer token with the Firebase Admin SDK. Expired, invalid and unverifiable tokens receive distinct 401 responses. See [excerpt 03](../excerpts/03-token-verification.md).
- The routers take the UID only from verified claims. `ReservationRequest.user_uid` is accepted for client compatibility but ignored.
- `ChatRequest` forbids extra fields, so a chat request body cannot carry identity at all.

## Client Structure

- An auth-state router in `lib/main.dart` gates the app shell.
- One shared shell (`lib/widgets/dashboard_navigation.dart`) provides desktop, tablet and compact breakpoints.
- `users/{uid}.fullName` is the single source for the displayed name. `UserNameNotifier` (`lib/services/user_name_notifier.dart`) is provided once at the app root and feeds the profile screen, the sidebar and the dashboard greeting.
- `BACKEND_URL` is a compile-time `--dart-define`. `lib/services/backend_url.dart` rejects a missing, relative or non-`http(s)` value and does not echo the invalid value in its error.

## Known Architectural Gaps

- No backend lifecycle for `reserved` → `running` → `idle`; the external component owns it.
- No Firestore security rules, CI, container setup or deployment configuration in the application repository.
- Reservation-route 500 responses include exception text, and CORS allows all origins.
