# @rivergen/witness

[![npm](https://img.shields.io/npm/v/@rivergen/witness)](https://www.npmjs.com/package/@rivergen/witness)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

Type contracts for [RiverGen](https://github.com/Mithun-Chandar/rivergen) field continuity audits.

---

## What this is

RiverGen's gates (#1–#11) verify that your realtime **pipeline is wired**: mutations go through EventFactory, events reach listeners, broadcasts reach projections, projections update the cache.

Gate #12 — **Witness** — verifies that the **data survived the pipeline**. Every field. Every hop. Every ghost reconciliation.

This package exports the TypeScript types that every generated `<domain>.witness.ts` file is built on. It has zero runtime code — types only. You fill the generated scaffold; `rivergen verify` runs it.

---

## Install

```bash
npm install @rivergen/witness
```

Install alongside [`@rivergen/cli`](https://www.npmjs.com/package/@rivergen/cli):

```bash
npm install -g @rivergen/cli
npm install @rivergen/witness
```

---

## How it works

`rivergen gen specs/task.json` generates a `task.witness.ts` scaffold that imports from this package. You fill in the assertions; `rivergen verify` runs them as Gate #12.

Gate #12 has three layers:

| Layer | What it checks |
|-------|----------------|
| **Layer 1** — Required fields | Every event in `requiredFields` has all listed keys in its `testPayloads` entry |
| **Layer 2** — Schema/broadcast contract | Required fields align with the registered Zod schema and broadcast shape |
| **Layer 3** — Projection proof | Your `lifecycle()` and `signals{}` assertions pass — field values survive create/update/delete/ghost reconciliation |

Layer 1 and 2 pass immediately after generation. Layer 3 is a stub — it shows as a warning until you fill `lifecycle()`.

---

## Usage

A generated witness looks like this. You fill in `lifecycle()` and `signals{}`:

```ts
import type { DomainWitness, WitnessAssertion } from "@rivergen/witness";

export interface TaskPayload {
  taskId: string;
  title: string;
  projectId: string;
  assigneeId: string | null;
  status: "todo" | "in-progress" | "done";
  createdAt: string;
  updatedAt: string;
  clientTempId: string | null;
}

type MinimalQC = {
  prefetchQuery(opts: { queryKey: unknown[]; queryFn: () => unknown }): Promise<void>;
  getQueryData<T>(key: unknown[]): T | undefined;
  setQueryData(key: unknown[], data: unknown): void;
};

const LIST_KEY = ["tasks", "list", "project-id-here"];

export const taskWitness: DomainWitness<TaskPayload> = {
  domain: "task",
  events: ["task.created", "task.updated", "task.deleted"],

  requiredFields: {
    "task.created": ["taskId", "title", "projectId", "assigneeId", "status", "createdAt", "updatedAt", "clientTempId"],
    "task.updated": ["taskId", "title", "projectId", "assigneeId", "status", "updatedAt", "clientTempId"],
    "task.deleted": ["taskId", "projectId"],
  },

  testPayloads: {
    "task.created": {
      taskId: "test-task-001",
      title: "Build the witness",
      projectId: "proj-001",
      assigneeId: null,
      status: "todo",
      createdAt: "2026-01-01T00:00:00.000Z",
      updatedAt: "2026-01-01T00:00:00.000Z",
      clientTempId: null,
      _meta: {
        resourceId: "test-task-001",
        actor: { id: "user-001", type: "user" },
        context: { realmId: "proj-001" },
        correlationId: "corr-001",
        eventVersion: "1.0",
      },
    },
    // ... other events
  },

  async lifecycle(queryClient): Promise<WitnessAssertion[]> {
    const qc = queryClient as MinimalQC;
    const assertions: WitnessAssertion[] = [];

    // Seed the cache
    await qc.prefetchQuery({ queryKey: LIST_KEY, queryFn: () => [] });

    // Simulate create
    qc.setQueryData(LIST_KEY, [{ id: "test-task-001", title: "Build the witness" }]);
    const afterCreate = qc.getQueryData<{ id: string; title: string }[]>(LIST_KEY) ?? [];
    const created = afterCreate.find((x) => x.id === "test-task-001");
    assertions.push({ name: "task.created lands in list", ok: !!created });
    assertions.push({ name: "task.created.title preserved", ok: created?.title === "Build the witness" });

    // Simulate delete
    qc.setQueryData(LIST_KEY, afterCreate.filter((x) => x.id !== "test-task-001"));
    const afterDelete = qc.getQueryData<{ id: string }[]>(LIST_KEY) ?? [];
    assertions.push({ name: "task.deleted removes from list", ok: !afterDelete.find((x) => x.id === "test-task-001") });

    return assertions;
  },

  signals: {},
};
```

### The `queryClient` passed to `lifecycle()`

The gate runner passes a minimal in-memory query client. It supports `getQueryData`, `setQueryData`, `prefetchQuery`, `setQueriesData`, and `invalidateQueries`. Use `setQueryData` to simulate cache mutations directly — this is intentional. The witness proves field continuity in the projection layer, not the network layer (gates #1–#4 verify the wiring).

### Ghost reconciliation

Optimistic UIs create a ghost entry while the mutation is in flight. When the confirmed WS event arrives, the ghost is replaced. Witness the full cycle:

```ts
// Ghost appears
const ghostId = "ghost-temp-001";
qc.setQueryData(LIST_KEY, [{ id: ghostId, title: "...", clientTempId: ghostId }]);

// Confirmed entity arrives — ghost gone, real entity present
const prev = qc.getQueryData<{ id: string }[]>(LIST_KEY) ?? [];
qc.setQueryData(LIST_KEY, prev.filter((x) => x.id !== ghostId).concat({ id: "real-001", title: "..." }));

const after = qc.getQueryData<{ id: string }[]>(LIST_KEY) ?? [];
assertions.push({ name: "ghost replaced by confirmed entity", ok: !after.find((x) => x.id === ghostId) && !!after.find((x) => x.id === "real-001") });
```

---

## Types

### `DomainWitness<TPayload>`

The shape every `<domain>.witness.ts` export must satisfy.

```ts
interface DomainWitness<TPayload> {
  domain: string;
  events: string[];
  requiredFields: Record<string, (keyof TPayload)[]>;
  testPayloads: Record<string, TPayload & { _meta: WitnessMeta }>;
  lifecycle(queryClient: unknown): Promise<WitnessAssertion[]>;
  signals: Record<string, (queryClient: unknown) => Promise<WitnessAssertion[]>>;
}
```

- `domain` — matches `domain.key` in the spec
- `events` — all events the domain broadcasts
- `requiredFields` — per-event list of payload keys that must survive to the cache (Layer 1)
- `testPayloads` — representative payload + `_meta` envelope for each event (Layer 2)
- `lifecycle()` — receives an in-memory QueryClient; simulate create/update/delete/ghost; return assertions (Layer 3)
- `signals{}` — additional Layer 3 assertions keyed by event name (for events that don't follow the main CRUD path)

### `WitnessAssertion`

```ts
interface WitnessAssertion {
  name: string;      // human-readable assertion label shown in verify output
  ok: boolean;       // pass or fail
  detail?: string;   // optional failure detail
}
```

### `LayerResult`

```ts
interface LayerResult {
  ok: boolean;
  violations: { file: string; message: string; severity: "error" | "warning" }[];
}
```

### `DomainWitnessReport`

```ts
interface DomainWitnessReport {
  domain: string;
  layer1: LayerResult;
  layer2: LayerResult;
  layer3: LayerResult;
}
```

### `WitnessResult`

```ts
interface WitnessResult {
  ok: boolean;
  reports: DomainWitnessReport[];
}
```

---

## Related

| | |
|---|---|
| **CLI** | [`@rivergen/cli`](https://www.npmjs.com/package/@rivergen/cli) — scaffold + all 12 gates |
| **GitHub (CLI)** | [Mithun-Chandar/rivergen](https://github.com/Mithun-Chandar/rivergen) |
| **GitHub (Witness)** | [Mithun-Chandar/rivergen-witness](https://github.com/Mithun-Chandar/rivergen-witness) |

---

## License

Apache 2.0

