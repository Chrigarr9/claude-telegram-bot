# paddelino (SP1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build paddelino (SP1 from the design at `docs/superpowers/specs/2026-05-12-sp1-telegram-on-pi-design.md`) — a household-scale Telegram bot powered by pi-agent-core, with per-user conversation continuity, first-run persona bootstrap, and forward-compatible seams for SP2–SP5.

**Architecture:** One Bun process; pi-agent-core in-process; per-user pi `Agent` cached in `AgentManager`; persistence on `agent_end` to `~/.paddelino/users/<id>/conversations/default.json`; identity (system prompt) bootstrapped on first run via a hardcoded meta-prompt + user description; Telegram channel via grammY with consolidated streaming; concurrent inbound handled by `agent.steer()`; allowlist + token-bucket rate limit; no eviction in SP1.

**Tech Stack:** TypeScript, Bun, grammY (Telegram), `@earendil-works/pi-agent-core`, `@earendil-works/pi-ai`, TypeBox (tool schemas — pi default), `bun test` for unit, pi-ai `registerFauxProvider` for integration.

**Implementation directory:** **New repo** at `/mnt/Shared/Code/projects/paddelino/`. This is **not** a worktree of `claude-telegram-bot` — that repo continues to run unchanged as a safety net. paddelino is a fresh `git init` directory.

**Reference repos (read-only):**
- pi monorepo: `/mnt/Shared/Code/projects/pi/` (for API shapes — confirmed during spec phase)
- Current bot: `/mnt/Shared/Code/projects/claude-telegram-bot/` (for verbatim ports of `security.ts`, `formatting.ts`, `process-lock.ts`, audit helper, streaming consolidation pattern)

**Reading the plan:** Tasks are numbered T01–T42 across 14 phases. Each task is one focused commit. Follow TDD where the task is logic; scaffolding tasks just verify `bun run typecheck` passes.

---

## Phase A — Repo and tooling (T01–T04)

### T01: Initialize the paddelino repo

**Files:**
- Create: `/mnt/Shared/Code/projects/paddelino/.gitignore`
- Create: `/mnt/Shared/Code/projects/paddelino/README.md` (stub — fleshed out in T39)

- [ ] **Step 1: Create dir and init git**

```bash
mkdir -p /mnt/Shared/Code/projects/paddelino
cd /mnt/Shared/Code/projects/paddelino
git init -b main
```

- [ ] **Step 2: Add `.gitignore`**

```
node_modules/
dist/
*.log
.env
.env.local
.DS_Store
.claude/settings.local.json
```

- [ ] **Step 3: Stub `README.md`**

```markdown
# paddelino

Household Telegram assistant on pi-agent-core. SP1 — see `docs/sp1.md`.
```

- [ ] **Step 4: Initial commit**

```bash
git add -A
git commit -m "Initial commit: paddelino skeleton"
```

---

### T02: package.json with Bun + pi deps

**Files:**
- Create: `/mnt/Shared/Code/projects/paddelino/package.json`

- [ ] **Step 1: Write `package.json`**

```json
{
  "name": "paddelino",
  "version": "0.1.0",
  "description": "Household Telegram assistant on pi-agent-core",
  "type": "module",
  "main": "src/index.ts",
  "scripts": {
    "start": "bun run src/index.ts",
    "dev": "bun --watch run src/index.ts",
    "typecheck": "tsc --noEmit",
    "test": "bun test",
    "build": "bun build --compile src/index.ts --outfile dist/paddelino",
    "check": "bun run typecheck && bun run test"
  },
  "dependencies": {
    "@earendil-works/pi-agent-core": "*",
    "@earendil-works/pi-ai": "*",
    "@sinclair/typebox": "*",
    "grammy": "*",
    "@grammyjs/runner": "*"
  },
  "devDependencies": {
    "@types/bun": "*",
    "typescript": "*"
  }
}
```

- [ ] **Step 2: Resolve `"*"` to actual current versions**

Run:

```bash
cd /mnt/Shared/Code/projects/paddelino
bun add @earendil-works/pi-agent-core @earendil-works/pi-ai @sinclair/typebox grammy @grammyjs/runner
bun add -d @types/bun typescript
```

This will fill the `*` placeholders with concrete versions in `package.json`.

- [ ] **Step 3: Verify install**

```bash
bun install
ls node_modules/@earendil-works/pi-agent-core/dist/index.js  # should exist
```

- [ ] **Step 4: Commit**

```bash
git add package.json bun.lockb
git commit -m "Add package.json and install pi-agent-core, grammY, TypeBox"
```

---

### T03: TypeScript config

**Files:**
- Create: `/mnt/Shared/Code/projects/paddelino/tsconfig.json`

- [ ] **Step 1: Write `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ESNext"],
    "types": ["bun-types"],
    "allowImportingTsExtensions": false,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "rootDir": ".",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*", "tests/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

- [ ] **Step 2: Verify typecheck passes on empty repo**

```bash
mkdir -p src
echo 'export {};' > src/index.ts
bun run typecheck
```

Expected: no output, exit 0.

- [ ] **Step 3: Commit**

```bash
git add tsconfig.json src/index.ts
git commit -m "Add tsconfig.json (strict mode); stub src/index.ts"
```

---

### T04: Directory skeleton + .env.example

**Files:**
- Create: `/mnt/Shared/Code/projects/paddelino/src/agent/{tools,inputs/processors,router}/`
- Create: `/mnt/Shared/Code/projects/paddelino/src/channels/telegram/handlers/`
- Create: `/mnt/Shared/Code/projects/paddelino/src/shared/`
- Create: `/mnt/Shared/Code/projects/paddelino/tests/{unit,integration}/`
- Create: `/mnt/Shared/Code/projects/paddelino/.env.example`

- [ ] **Step 1: Create dirs (with `.gitkeep` so empty dirs are tracked)**

```bash
cd /mnt/Shared/Code/projects/paddelino
mkdir -p src/agent/tools src/agent/inputs/processors src/agent/router
mkdir -p src/channels/telegram/handlers src/shared
mkdir -p tests/unit tests/integration
mkdir -p scripts/systemd
touch src/agent/tools/.gitkeep src/agent/inputs/processors/.gitkeep src/agent/router/.gitkeep
touch src/channels/telegram/handlers/.gitkeep src/shared/.gitkeep
touch tests/unit/.gitkeep tests/integration/.gitkeep
```

- [ ] **Step 2: Write `.env.example`**

```
# Telegram (required)
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USERS=

# At least one provider key OR an OAuth token (see pi-ai docs)
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
OPENROUTER_API_KEY=
GEMINI_API_KEY=

# Optional overrides
# PADDELINO_HOME=~/.paddelino
# PADDELINO_PROVIDER_PRIORITY=anthropic,openai,openrouter,gemini
# PADDELINO_DEFAULT_PROVIDER=
# PADDELINO_DEFAULT_MODEL=
# PADDELINO_DEFAULT_THINKING_LEVEL=off
# PADDELINO_GREETING=
# PADDELINO_RATE_LIMIT_CAPACITY=10
# PADDELINO_RATE_LIMIT_REFILL_PER_SEC=0.5
```

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "Add directory skeleton and .env.example"
```

---

## Phase B — Shared utilities (T05–T07)

### T05: `src/shared/types.ts` — cross-cutting types

**Files:**
- Create: `src/shared/types.ts`

- [ ] **Step 1: Write shared types**

```ts
// src/shared/types.ts

export type UserId = string;

export interface AppConfig {
  telegram: {
    token: string;
    allowedUsers: Set<UserId>;
  };
  providers: {
    available: ProviderId[];           // ["anthropic", "openai", ...]
    priority: ProviderId[];            // honored from PADDELINO_PROVIDER_PRIORITY
    defaultProvider: ProviderId;       // first available in priority order
    defaultModel: string;              // chosen from the provider-aware default-model table
    defaultThinkingLevel: ThinkingLevel;
  };
  paths: {
    home: string;                      // ~/.paddelino or PADDELINO_HOME
  };
  rateLimit: {
    capacity: number;
    refillPerSec: number;
  };
  greeting?: string;
}

export type ProviderId = "anthropic" | "openai" | "openrouter" | "gemini";

export type ThinkingLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh";
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/shared/types.ts
git commit -m "Add shared types (UserId, AppConfig, ProviderId, ThinkingLevel)"
```

---

### T06: `src/shared/audit.ts` — JSON-per-line audit log

**Files:**
- Create: `src/shared/audit.ts`
- Test: `tests/unit/audit.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/audit.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync, readFileSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { createAudit } from "../../src/shared/audit";

describe("audit", () => {
  let dir: string;
  beforeEach(() => {
    dir = mkdtempSync(join(tmpdir(), "paddelino-audit-"));
  });

  test("appends JSON line per event", async () => {
    const audit = createAudit(join(dir, "audit.log"));
    await audit("auth_denied", "user-1", { reason: "not on allowlist" });
    await audit("prompt_received", "user-1", { length: 42 });

    const content = readFileSync(join(dir, "audit.log"), "utf8");
    const lines = content.trim().split("\n");
    expect(lines).toHaveLength(2);
    const first = JSON.parse(lines[0]!);
    expect(first.kind).toBe("auth_denied");
    expect(first.userId).toBe("user-1");
    expect(first.details.reason).toBe("not on allowlist");
    expect(typeof first.ts).toBe("string");
  });
});
```

- [ ] **Step 2: Run test (should fail — no implementation)**

```bash
bun test tests/unit/audit.test.ts
```

Expected: FAIL (module not found).

- [ ] **Step 3: Write `src/shared/audit.ts`**

```ts
// src/shared/audit.ts
import { appendFileSync, mkdirSync } from "node:fs";
import { dirname } from "node:path";

export type AuditKind =
  | "auth_denied"
  | "rate_limited"
  | "prompt_received"
  | "tool_call"
  | "tool_result"
  | "error"
  | "agent_end"
  | "restart";

export interface AuditFn {
  (kind: AuditKind, userId: string, details: Record<string, unknown>): Promise<void>;
}

export function createAudit(path: string): AuditFn {
  mkdirSync(dirname(path), { recursive: true });
  return async (kind, userId, details) => {
    const line = JSON.stringify({ ts: new Date().toISOString(), kind, userId, details }) + "\n";
    appendFileSync(path, line, "utf8");
  };
}
```

- [ ] **Step 4: Test passes**

```bash
bun test tests/unit/audit.test.ts
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/shared/audit.ts tests/unit/audit.test.ts
git commit -m "Add JSON-per-line audit log writer with test"
```

---

### T07: `src/shared/lock.ts` — PID-based process lock

**Files:**
- Create: `src/shared/lock.ts`
- Test: `tests/unit/lock.test.ts`
- Reference: `/mnt/Shared/Code/projects/claude-telegram-bot/src/process-lock.ts`

- [ ] **Step 1: Read the current bot's lock for reference**

```bash
cat /mnt/Shared/Code/projects/claude-telegram-bot/src/process-lock.ts
```

Adapt: same algorithm, paddelino paths, exported as `acquireLock(path): { release(): void }`.

- [ ] **Step 2: Write failing test**

```ts
// tests/unit/lock.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { acquireLock, LockHeldError } from "../../src/shared/lock";

describe("lock", () => {
  let dir: string;
  beforeEach(() => {
    dir = mkdtempSync(join(tmpdir(), "paddelino-lock-"));
  });

  test("acquires when no lock exists", () => {
    const h = acquireLock(join(dir, ".lock"));
    expect(h).toBeDefined();
    h.release();
  });

  test("throws LockHeldError when a live PID already holds it", () => {
    const path = join(dir, ".lock");
    const h = acquireLock(path);
    expect(() => acquireLock(path)).toThrow(LockHeldError);
    h.release();
  });

  test("reclaims when the previous PID is dead (writes bogus PID)", () => {
    const path = join(dir, ".lock");
    // Manually plant a stale PID (1 == init, but we use 999999 which is unlikely to exist).
    Bun.write(path, "999999");
    const h = acquireLock(path);
    expect(h).toBeDefined();
    h.release();
  });
});
```

- [ ] **Step 3: Write `src/shared/lock.ts`**

```ts
// src/shared/lock.ts
import { existsSync, readFileSync, unlinkSync, writeFileSync } from "node:fs";

export class LockHeldError extends Error {
  constructor(public readonly heldByPid: number, path: string) {
    super(`Lock held by PID ${heldByPid} at ${path}`);
  }
}

export interface LockHandle {
  release(): void;
}

function isAlive(pid: number): boolean {
  try {
    process.kill(pid, 0);
    return true;
  } catch (err: any) {
    return err.code === "EPERM";
  }
}

export function acquireLock(path: string): LockHandle {
  if (existsSync(path)) {
    const raw = readFileSync(path, "utf8").trim();
    const pid = Number.parseInt(raw, 10);
    if (Number.isFinite(pid) && pid > 0 && isAlive(pid)) {
      throw new LockHeldError(pid, path);
    }
    unlinkSync(path);
  }
  writeFileSync(path, String(process.pid), "utf8");
  return {
    release() {
      try { unlinkSync(path); } catch { /* already gone */ }
    },
  };
}
```

- [ ] **Step 4: Tests pass**

```bash
bun test tests/unit/lock.test.ts
```

Expected: PASS (3/3).

- [ ] **Step 5: Commit**

```bash
git add src/shared/lock.ts tests/unit/lock.test.ts
git commit -m "Port process-lock as src/shared/lock.ts with stale-PID reclaim"
```

---

## Phase C — Persistence (T08–T09)

### T08: `src/agent/persistence.ts` — atomic save/load envelope

**Files:**
- Create: `src/agent/persistence.ts`
- Test: `tests/unit/persistence.test.ts`

- [ ] **Step 1: Write failing tests**

```ts
// tests/unit/persistence.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync, readFileSync, writeFileSync, existsSync, readdirSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { createPersistence } from "../../src/agent/persistence";

describe("persistence", () => {
  let home: string;
  beforeEach(() => {
    home = mkdtempSync(join(tmpdir(), "paddelino-persistence-"));
  });

  test("save then load roundtrip", async () => {
    const p = createPersistence(home, "0.1.0", "pi-test");
    const ctx = { systemPrompt: "you are friendly", messages: [{ role: "user", content: "hi", timestamp: 1 }] };
    await p.save("user-1", ctx as any);
    const loaded = await p.load("user-1");
    expect(loaded).toEqual(ctx as any);
  });

  test("missing file returns null", async () => {
    const p = createPersistence(home, "0.1.0", "pi-test");
    const loaded = await p.load("nobody");
    expect(loaded).toBeNull();
  });

  test("corrupt JSON gets archived and returns null", async () => {
    const userDir = join(home, "users", "user-c", "conversations");
    require("node:fs").mkdirSync(userDir, { recursive: true });
    writeFileSync(join(userDir, "default.json"), "not json", "utf8");

    const p = createPersistence(home, "0.1.0", "pi-test");
    const loaded = await p.load("user-c");
    expect(loaded).toBeNull();

    const files = readdirSync(userDir);
    expect(files.some(f => f.startsWith("default.json.corrupt-"))).toBe(true);
  });

  test("incompatible pi version gets archived as .incompatible", async () => {
    const p1 = createPersistence(home, "0.1.0", "pi-OLD");
    await p1.save("user-i", { systemPrompt: "x", messages: [] } as any);

    const p2 = createPersistence(home, "0.1.0", "pi-NEW", { piVersionMatch: "pi-NEW" });
    const loaded = await p2.load("user-i");
    expect(loaded).toBeNull();

    const userDir = join(home, "users", "user-i", "conversations");
    const files = readdirSync(userDir);
    expect(files.some(f => f.startsWith("default.json.incompatible-"))).toBe(true);
  });

  test("atomic write: no .tmp file remains after save", async () => {
    const p = createPersistence(home, "0.1.0", "pi-test");
    await p.save("user-1", { systemPrompt: "x", messages: [] } as any);
    const userDir = join(home, "users", "user-1", "conversations");
    expect(existsSync(join(userDir, "default.json"))).toBe(true);
    expect(existsSync(join(userDir, "default.json.tmp"))).toBe(false);
  });
});
```

- [ ] **Step 2: Write `src/agent/persistence.ts`**

```ts
// src/agent/persistence.ts
import { mkdirSync, existsSync, readFileSync, writeFileSync, renameSync, openSync, fsyncSync, closeSync } from "node:fs";
import { join } from "node:path";

export interface PersistenceOptions {
  piVersionMatch?: string;   // when set, refuse to load envelopes whose piVersion differs
}

export interface Persistence {
  save(userId: string, context: unknown): Promise<void>;
  load(userId: string): Promise<unknown | null>;
}

interface Envelope {
  paddelinoVersion: string;
  piVersion: string;
  savedAt: string;
  context: unknown;
}

export function createPersistence(
  home: string,
  paddelinoVersion: string,
  piVersion: string,
  opts: PersistenceOptions = {},
): Persistence {
  const userDir = (userId: string) => join(home, "users", userId, "conversations");
  const filePath = (userId: string) => join(userDir(userId), "default.json");

  return {
    async save(userId, context) {
      const dir = userDir(userId);
      mkdirSync(dir, { recursive: true });
      const envelope: Envelope = {
        paddelinoVersion,
        piVersion,
        savedAt: new Date().toISOString(),
        context,
      };
      const tmp = filePath(userId) + ".tmp";
      writeFileSync(tmp, JSON.stringify(envelope), "utf8");
      const fd = openSync(tmp, "r");
      try { fsyncSync(fd); } finally { closeSync(fd); }
      renameSync(tmp, filePath(userId));
    },

    async load(userId) {
      const path = filePath(userId);
      if (!existsSync(path)) return null;
      const raw = readFileSync(path, "utf8");

      let envelope: Envelope;
      try {
        envelope = JSON.parse(raw);
      } catch {
        archive(path, "corrupt");
        return null;
      }

      const expectedPi = opts.piVersionMatch;
      if (expectedPi !== undefined && envelope.piVersion !== expectedPi) {
        archive(path, "incompatible");
        return null;
      }
      return envelope.context;
    },
  };
}

function archive(path: string, kind: "corrupt" | "incompatible"): void {
  const ts = new Date().toISOString().replace(/[:.]/g, "-");
  const dest = `${path}.${kind}-${ts}`;
  renameSync(path, dest);
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/persistence.test.ts
```

Expected: PASS (5/5).

- [ ] **Step 4: Commit**

```bash
git add src/agent/persistence.ts tests/unit/persistence.test.ts
git commit -m "Add persistence with atomic write and corrupt/incompatible archival"
```

---

### T09: Inject persistence file path constants

This task ensures the persistence layer is used through a single factory call with the runtime paddelino + pi version pulled from `package.json`.

**Files:**
- Modify: `src/shared/types.ts`

- [ ] **Step 1: Add version helper export to `src/shared/types.ts`**

Append:

```ts
export const PADDELINO_VERSION = "0.1.0";
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/shared/types.ts
git commit -m "Export PADDELINO_VERSION constant"
```

---

## Phase D — Identity and tools (T10–T13)

### T10: `src/agent/identity.ts` — read/write system prompt

**Files:**
- Create: `src/agent/identity.ts`
- Test: `tests/unit/identity.test.ts`

`bootstrapPersona()` is added in T34 — this task is the *read/write* side only, so the bot can use whatever's on disk.

- [ ] **Step 1: Write failing tests**

```ts
// tests/unit/identity.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync, mkdirSync, writeFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { createIdentity } from "../../src/agent/identity";

describe("identity", () => {
  let home: string;
  beforeEach(() => { home = mkdtempSync(join(tmpdir(), "paddelino-id-")); });

  test("hasPersona is false on a fresh dir", () => {
    const id = createIdentity(home);
    expect(id.hasPersona()).toBe(false);
  });

  test("setPersona writes both files and hasPersona becomes true", () => {
    const id = createIdentity(home);
    id.setPersona("user described this", "GENERATED PROMPT");
    expect(id.hasPersona()).toBe(true);
    expect(id.getSystemPrompt()).toBe("GENERATED PROMPT");
  });

  test("getSystemPrompt reads from disk", () => {
    mkdirSync(join(home, "identity"), { recursive: true });
    writeFileSync(join(home, "identity", "system-prompt.md"), "FROM DISK", "utf8");
    const id = createIdentity(home);
    expect(id.getSystemPrompt()).toBe("FROM DISK");
  });

  test("getSystemPrompt throws when no persona is set", () => {
    const id = createIdentity(home);
    expect(() => id.getSystemPrompt()).toThrow();
  });
});
```

- [ ] **Step 2: Write `src/agent/identity.ts`**

```ts
// src/agent/identity.ts
import { existsSync, mkdirSync, readFileSync, writeFileSync } from "node:fs";
import { join } from "node:path";

export interface Identity {
  hasPersona(): boolean;
  getSystemPrompt(): string;
  setPersona(userInput: string, generatedPrompt: string): void;
}

export function createIdentity(home: string): Identity {
  const dir = join(home, "identity");
  const promptPath = join(dir, "system-prompt.md");
  const inputPath = join(dir, "persona-input.md");

  return {
    hasPersona() {
      if (!existsSync(promptPath)) return false;
      return readFileSync(promptPath, "utf8").trim().length > 0;
    },
    getSystemPrompt() {
      if (!existsSync(promptPath)) {
        throw new Error("No persona configured. Run bootstrap or set ~/.paddelino/identity/system-prompt.md");
      }
      return readFileSync(promptPath, "utf8");
    },
    setPersona(userInput, generatedPrompt) {
      mkdirSync(dir, { recursive: true });
      writeFileSync(promptPath, generatedPrompt, "utf8");
      writeFileSync(inputPath, userInput, "utf8");
    },
  };
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/identity.test.ts
```

Expected: PASS (4/4).

- [ ] **Step 4: Commit**

```bash
git add src/agent/identity.ts tests/unit/identity.test.ts
git commit -m "Add identity read/write (bootstrap LLM call added later in T34)"
```

---

### T11: `src/agent/tools/current-time.ts` — TypeBox-defined tool

**Files:**
- Create: `src/agent/tools/current-time.ts`
- Test: `tests/unit/tools/current-time.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/tools/current-time.test.ts
import { describe, test, expect } from "bun:test";
import { currentTimeTool } from "../../../src/agent/tools/current-time";

describe("currentTime tool", () => {
  test("returns an ISO string for the given timezone", async () => {
    const res = await currentTimeTool.execute("id-1", { timezone: "UTC" });
    expect(res.content[0]!.type).toBe("text");
    expect(/\d{4}-\d{2}-\d{2}T\d{2}:\d{2}/.test((res.content[0] as any).text)).toBe(true);
  });

  test("falls back to UTC for invalid timezones", async () => {
    const res = await currentTimeTool.execute("id-1", { timezone: "Bogus/Zone" });
    expect(res.details.usedTimezone).toBe("UTC");
  });
});
```

- [ ] **Step 2: Write `src/agent/tools/current-time.ts`**

```ts
// src/agent/tools/current-time.ts
import { Type, type Static } from "@sinclair/typebox";
import type { AgentTool } from "@earendil-works/pi-agent-core";

const Params = Type.Object({
  timezone: Type.String({ description: "IANA timezone, e.g. 'Europe/Zurich'. UTC if invalid." }),
});

type ParamsT = Static<typeof Params>;

interface Details { usedTimezone: string }

export const currentTimeTool: AgentTool<typeof Params, Details> = {
  name: "current_time",
  description: "Return the current date and time in the requested timezone.",
  label: "Current time",
  parameters: Params,
  async execute(_id, params): Promise<{ content: any[]; details: Details }> {
    let used = params.timezone;
    let formatted: string;
    try {
      formatted = new Intl.DateTimeFormat("sv-SE", {
        timeZone: params.timezone,
        year: "numeric", month: "2-digit", day: "2-digit",
        hour: "2-digit", minute: "2-digit", second: "2-digit",
        hour12: false,
      }).format(new Date());
    } catch {
      used = "UTC";
      formatted = new Date().toISOString();
    }
    return {
      content: [{ type: "text", text: `${formatted} (${used})` }],
      details: { usedTimezone: used },
    };
  },
} as any;
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/tools/current-time.test.ts
```

Expected: PASS (2/2).

- [ ] **Step 4: Commit**

```bash
git add src/agent/tools/current-time.ts tests/unit/tools/current-time.test.ts
git commit -m "Add current_time tool (TypeBox schema, IANA timezone, UTC fallback)"
```

---

### T12: `src/agent/tools/note.ts` — write a note to disk

**Files:**
- Create: `src/agent/tools/note.ts`
- Test: `tests/unit/tools/note.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/tools/note.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync, readdirSync, readFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { createNoteTool } from "../../../src/agent/tools/note";

describe("note tool", () => {
  let home: string;
  beforeEach(() => { home = mkdtempSync(join(tmpdir(), "paddelino-note-")); });

  test("writes a markdown file under users/<id>/notes/", async () => {
    const tool = createNoteTool(home, "user-1");
    const res = await tool.execute("id-1", { content: "remember the milk" });
    const dir = join(home, "users", "user-1", "notes");
    const files = readdirSync(dir);
    expect(files).toHaveLength(1);
    expect(readFileSync(join(dir, files[0]!), "utf8")).toBe("remember the milk");
    expect(res.details.path.endsWith(files[0]!)).toBe(true);
  });
});
```

- [ ] **Step 2: Write `src/agent/tools/note.ts`**

```ts
// src/agent/tools/note.ts
import { mkdirSync, writeFileSync } from "node:fs";
import { join } from "node:path";
import { Type, type Static } from "@sinclair/typebox";
import type { AgentTool } from "@earendil-works/pi-agent-core";

const Params = Type.Object({
  content: Type.String({ description: "Markdown content for the note. Plain text is fine." }),
});

type ParamsT = Static<typeof Params>;

interface Details { path: string }

export function createNoteTool(home: string, userId: string): AgentTool<typeof Params, Details> {
  return {
    name: "note",
    description: "Write a short note that the user (and you, in future) can refer back to.",
    label: "Note",
    parameters: Params,
    async execute(_id, params): Promise<{ content: any[]; details: Details }> {
      const dir = join(home, "users", userId, "notes");
      mkdirSync(dir, { recursive: true });
      const ts = new Date().toISOString().replace(/[:.]/g, "-");
      const path = join(dir, `${ts}.md`);
      writeFileSync(path, params.content, "utf8");
      return {
        content: [{ type: "text", text: `Saved note to ${path}` }],
        details: { path },
      };
    },
  } as any;
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/tools/note.test.ts
```

Expected: PASS (1/1).

- [ ] **Step 4: Commit**

```bash
git add src/agent/tools/note.ts tests/unit/tools/note.test.ts
git commit -m "Add note tool (writes timestamped markdown per user)"
```

---

### T13: Combine tools into a `getTools(home, userId)` helper

**Files:**
- Modify: `src/agent/identity.ts`

- [ ] **Step 1: Add `getTools()` to identity**

Append to `src/agent/identity.ts`:

```ts
// added
import { currentTimeTool } from "./tools/current-time";
import { createNoteTool } from "./tools/note";

export function getDefaultTools(home: string, userId: string) {
  return [currentTimeTool, createNoteTool(home, userId)];
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/agent/identity.ts
git commit -m "Add getDefaultTools(home, userId) — current_time and note"
```

---

## Phase E — Input processors (T14–T17)

### T14: `src/agent/inputs/types.ts` — RawInput discriminated union

**Files:**
- Create: `src/agent/inputs/types.ts`

- [ ] **Step 1: Write the types**

```ts
// src/agent/inputs/types.ts
import type { UserMessage } from "@earendil-works/pi-ai";

export interface Meta {
  userId: string;
  channel: "telegram";   // SP2 adds "display"; SP4 adds "voice"
  timestamp: number;
}

export type RawInput =
  | { kind: "text";     payload: { text: string };                                              meta: Meta }
  | { kind: "photo";    payload: { fileIds: string[]; caption?: string; mimeTypes: string[] };  meta: Meta }
  | { kind: "voice";    payload: { fileId: string };                                            meta: Meta }
  | { kind: "audio";    payload: { fileId: string; filename?: string };                         meta: Meta }
  | { kind: "document"; payload: { fileId: string; filename?: string; mimeType?: string };      meta: Meta }
  | { kind: "video";    payload: { fileId: string };                                            meta: Meta }
  | { kind: "raw_file"; payload: { fileId: string; filename?: string };                         meta: Meta };

export interface ProcessorContext {
  fetchFile(fileId: string): Promise<{ data: Uint8Array; mimeType: string }>;
}

export interface InputProcessor<K extends RawInput["kind"] = RawInput["kind"]> {
  kind: K;
  process(input: Extract<RawInput, { kind: K }>, ctx: ProcessorContext): Promise<UserMessage[]>;
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/agent/inputs/types.ts
git commit -m "Add RawInput discriminated union and InputProcessor interface"
```

---

### T15: `src/agent/inputs/processors/text.ts` — text passthrough

**Files:**
- Create: `src/agent/inputs/processors/text.ts`
- Test: `tests/unit/inputs/text.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/inputs/text.test.ts
import { describe, test, expect } from "bun:test";
import { textProcessor } from "../../../src/agent/inputs/processors/text";

describe("textProcessor", () => {
  test("wraps text into a UserMessage", async () => {
    const out = await textProcessor.process(
      { kind: "text", payload: { text: "hi" }, meta: { userId: "u", channel: "telegram", timestamp: 100 } },
      { fetchFile: async () => ({ data: new Uint8Array(), mimeType: "" }) },
    );
    expect(out).toHaveLength(1);
    expect(out[0]).toMatchObject({ role: "user", content: "hi", timestamp: 100 });
  });
});
```

- [ ] **Step 2: Write `src/agent/inputs/processors/text.ts`**

```ts
// src/agent/inputs/processors/text.ts
import type { InputProcessor } from "../types";

export const textProcessor: InputProcessor<"text"> = {
  kind: "text",
  async process(input) {
    return [{ role: "user", content: input.payload.text, timestamp: input.meta.timestamp }];
  },
};
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/inputs/text.test.ts
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/agent/inputs/processors/text.ts tests/unit/inputs/text.test.ts
git commit -m "Add text input processor (passthrough)"
```

---

### T16: `src/agent/inputs/processors/photo.ts` — multimodal user message

**Files:**
- Create: `src/agent/inputs/processors/photo.ts`
- Test: `tests/unit/inputs/photo.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/inputs/photo.test.ts
import { describe, test, expect } from "bun:test";
import { photoProcessor } from "../../../src/agent/inputs/processors/photo";

describe("photoProcessor", () => {
  test("builds a user message with text + image content blocks", async () => {
    const ctx = {
      async fetchFile(fileId: string) {
        return { data: new Uint8Array([1, 2, 3]), mimeType: "image/jpeg" };
      },
    };
    const out = await photoProcessor.process(
      {
        kind: "photo",
        payload: { fileIds: ["f1", "f2"], caption: "look", mimeTypes: ["image/jpeg", "image/png"] },
        meta: { userId: "u", channel: "telegram", timestamp: 100 },
      },
      ctx,
    );
    expect(out).toHaveLength(1);
    const msg = out[0]!;
    expect(msg.role).toBe("user");
    expect(Array.isArray(msg.content)).toBe(true);
    const blocks = msg.content as any[];
    expect(blocks[0]).toEqual({ type: "text", text: "look" });
    expect(blocks[1]).toMatchObject({ type: "image", mimeType: "image/jpeg" });
    expect(blocks[2]).toMatchObject({ type: "image", mimeType: "image/png" });
  });

  test("omits the text block when caption is empty", async () => {
    const ctx = { async fetchFile() { return { data: new Uint8Array([1]), mimeType: "image/jpeg" }; } };
    const out = await photoProcessor.process(
      { kind: "photo", payload: { fileIds: ["f1"], mimeTypes: ["image/jpeg"] }, meta: { userId: "u", channel: "telegram", timestamp: 1 } },
      ctx,
    );
    const blocks = out[0]!.content as any[];
    expect(blocks[0]!.type).toBe("image");
  });
});
```

- [ ] **Step 2: Write `src/agent/inputs/processors/photo.ts`**

```ts
// src/agent/inputs/processors/photo.ts
import type { InputProcessor } from "../types";

export const photoProcessor: InputProcessor<"photo"> = {
  kind: "photo",
  async process(input, ctx) {
    const blocks: Array<{ type: "text"; text: string } | { type: "image"; data: string; mimeType: string }> = [];
    if (input.payload.caption && input.payload.caption.length > 0) {
      blocks.push({ type: "text", text: input.payload.caption });
    }
    for (let i = 0; i < input.payload.fileIds.length; i++) {
      const fileId = input.payload.fileIds[i]!;
      const fetched = await ctx.fetchFile(fileId);
      blocks.push({
        type: "image",
        data: Buffer.from(fetched.data).toString("base64"),
        mimeType: fetched.mimeType,
      });
    }
    return [{ role: "user", content: blocks, timestamp: input.meta.timestamp }];
  },
};
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/inputs/photo.test.ts
```

Expected: PASS (2/2).

- [ ] **Step 4: Commit**

```bash
git add src/agent/inputs/processors/photo.ts tests/unit/inputs/photo.test.ts
git commit -m "Add photo input processor (base64 + image content blocks, caption-aware)"
```

---

### T17: `src/agent/inputs/registry.ts` — processor dispatch

**Files:**
- Create: `src/agent/inputs/registry.ts`
- Test: `tests/unit/inputs/registry.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/inputs/registry.test.ts
import { describe, test, expect } from "bun:test";
import { InputProcessorRegistry } from "../../../src/agent/inputs/registry";
import { textProcessor } from "../../../src/agent/inputs/processors/text";

describe("InputProcessorRegistry", () => {
  test("dispatches to the registered processor", async () => {
    const reg = new InputProcessorRegistry();
    reg.register(textProcessor);
    const out = await reg.process(
      { kind: "text", payload: { text: "hi" }, meta: { userId: "u", channel: "telegram", timestamp: 1 } },
      { fetchFile: async () => ({ data: new Uint8Array(), mimeType: "" }) },
    );
    expect(out).toHaveLength(1);
  });

  test("throws when no processor is registered for the kind", async () => {
    const reg = new InputProcessorRegistry();
    await expect(
      reg.process(
        { kind: "voice", payload: { fileId: "f" }, meta: { userId: "u", channel: "telegram", timestamp: 1 } },
        { fetchFile: async () => ({ data: new Uint8Array(), mimeType: "" }) },
      ),
    ).rejects.toThrow(/No processor for kind 'voice'/);
  });
});
```

- [ ] **Step 2: Write `src/agent/inputs/registry.ts`**

```ts
// src/agent/inputs/registry.ts
import type { InputProcessor, ProcessorContext, RawInput } from "./types";
import type { UserMessage } from "@earendil-works/pi-ai";

export class InputProcessorRegistry {
  private map = new Map<RawInput["kind"], InputProcessor>();

  register<K extends RawInput["kind"]>(p: InputProcessor<K>): void {
    this.map.set(p.kind, p as InputProcessor);
  }

  has(kind: RawInput["kind"]): boolean {
    return this.map.has(kind);
  }

  async process(input: RawInput, ctx: ProcessorContext): Promise<UserMessage[]> {
    const p = this.map.get(input.kind);
    if (!p) throw new Error(`No processor for kind '${input.kind}'`);
    return p.process(input as any, ctx);
  }
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/inputs/registry.test.ts
```

Expected: PASS (2/2).

- [ ] **Step 4: Commit**

```bash
git add src/agent/inputs/registry.ts tests/unit/inputs/registry.test.ts
git commit -m "Add InputProcessorRegistry with kind-dispatch and 'No processor' error"
```

---

## Phase F — Router (T18–T19)

### T18: `src/agent/router/types.ts`

**Files:**
- Create: `src/agent/router/types.ts`

- [ ] **Step 1: Write the types**

```ts
// src/agent/router/types.ts
import type { Model } from "@earendil-works/pi-ai";
import type { AgentTool } from "@earendil-works/pi-agent-core";
import type { ThinkingLevel } from "../../shared/types";
import type { RawInput } from "../inputs/types";

export interface RoutingDecision {
  identityName: string;
  model: Model<any>;
  tools: AgentTool<any>[];
  thinkingLevel: ThinkingLevel;
}

export interface RoutingContext {
  recentMessageCount: number;
}

export interface RoutingRule {
  match(input: RawInput, userId: string, ctx: RoutingContext): Promise<RoutingDecision | null>;
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/agent/router/types.ts
git commit -m "Add Router types (RoutingDecision, RoutingRule)"
```

---

### T19: `src/agent/router/router.ts` — empty rule list returns default

**Files:**
- Create: `src/agent/router/router.ts`
- Test: `tests/unit/router.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/router.test.ts
import { describe, test, expect } from "bun:test";
import { Router } from "../../src/agent/router/router";

describe("Router", () => {
  const defaultDecision = {
    identityName: "default",
    model: { provider: "anthropic", id: "haiku" } as any,
    tools: [],
    thinkingLevel: "off" as const,
  };

  test("with zero rules, decide returns the default", async () => {
    const r = new Router(defaultDecision);
    const out = await r.decide(
      { kind: "text", payload: { text: "x" }, meta: { userId: "u", channel: "telegram", timestamp: 1 } },
      "u",
      { recentMessageCount: 0 },
    );
    expect(out).toBe(defaultDecision);
  });

  test("first matching rule wins; non-matching rules return null", async () => {
    const r = new Router(defaultDecision);
    const override = { ...defaultDecision, identityName: "override" };
    r.register({ async match() { return null; } });
    r.register({ async match() { return override; } });
    const out = await r.decide(
      { kind: "text", payload: { text: "x" }, meta: { userId: "u", channel: "telegram", timestamp: 1 } },
      "u",
      { recentMessageCount: 0 },
    );
    expect(out.identityName).toBe("override");
  });
});
```

- [ ] **Step 2: Write `src/agent/router/router.ts`**

```ts
// src/agent/router/router.ts
import type { RawInput } from "../inputs/types";
import type { RoutingContext, RoutingDecision, RoutingRule } from "./types";

export class Router {
  private rules: RoutingRule[] = [];

  constructor(private readonly defaultDecision: RoutingDecision) {}

  register(rule: RoutingRule): void {
    this.rules.push(rule);
  }

  async decide(input: RawInput, userId: string, ctx: RoutingContext): Promise<RoutingDecision> {
    for (const rule of this.rules) {
      const decision = await rule.match(input, userId, ctx);
      if (decision) return decision;
    }
    return this.defaultDecision;
  }
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/router.test.ts
```

Expected: PASS (2/2).

- [ ] **Step 4: Commit**

```bash
git add src/agent/router/router.ts tests/unit/router.test.ts
git commit -m "Add Router with zero-rules default and first-match-wins ordering"
```

---

## Phase G — Agent factory and manager (T20–T22)

### T20: `src/agent/factory.ts` — createAgent

**Files:**
- Create: `src/agent/factory.ts`

- [ ] **Step 1: Write `src/agent/factory.ts`**

```ts
// src/agent/factory.ts
import { Agent, type AgentMessage, type BeforeToolCallContext, type AfterToolCallContext } from "@earendil-works/pi-agent-core";
import type { Model } from "@earendil-works/pi-ai";
import type { AgentTool } from "@earendil-works/pi-agent-core";
import type { ThinkingLevel } from "../shared/types";
import type { AuditFn } from "../shared/audit";

export interface CreateAgentArgs {
  userId: string;
  systemPrompt: string;
  model: Model<any>;
  tools: AgentTool<any>[];
  thinkingLevel: ThinkingLevel;
  initialMessages?: AgentMessage[];
  audit: AuditFn;
}

export function createAgent(args: CreateAgentArgs): Agent {
  return new Agent({
    initialState: {
      systemPrompt: args.systemPrompt,
      model: args.model,
      tools: args.tools,
      thinkingLevel: args.thinkingLevel,
      messages: args.initialMessages ?? [],
    },
    beforeToolCall: async (_ctx: BeforeToolCallContext) => {
      // SP1: no-op. SP5 fills in autonomy levels.
      return undefined;
    },
    afterToolCall: async (ctx: AfterToolCallContext) => {
      await args.audit("tool_result", args.userId, {
        toolName: ctx.toolName,
        isError: ctx.isError,
      });
      return undefined;
    },
    steeringMode: "one-at-a-time",
    followUpMode: "one-at-a-time",
  });
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0. If pi-agent-core's exported types differ from `BeforeToolCallContext` / `AfterToolCallContext` names, fix the imports — see `node_modules/@earendil-works/pi-agent-core/dist/index.d.ts`.

- [ ] **Step 3: Commit**

```bash
git add src/agent/factory.ts
git commit -m "Add createAgent factory wiring identity, model, tools, audit hook"
```

---

### T21: `src/agent/manager.ts` — per-user Agent cache

**Files:**
- Create: `src/agent/manager.ts`
- Test: `tests/unit/manager.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/manager.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { registerFauxProvider, getModel } from "@earendil-works/pi-ai";
import { AgentManager } from "../../src/agent/manager";
import { createPersistence } from "../../src/agent/persistence";
import { createIdentity } from "../../src/agent/identity";
import { createAudit } from "../../src/shared/audit";

describe("AgentManager", () => {
  let home: string;
  let manager: AgentManager;

  beforeEach(() => {
    home = mkdtempSync(join(tmpdir(), "paddelino-mgr-"));
    registerFauxProvider({ provider: "faux", id: "test" });
    const identity = createIdentity(home);
    identity.setPersona("desc", "system prompt");
    const persistence = createPersistence(home, "0.1.0", "pi-test");
    const audit = createAudit(join(home, "audit.log"));
    manager = new AgentManager({
      home,
      identity,
      persistence,
      audit,
      defaultModel: getModel("faux" as any, "test" as any),
      defaultThinkingLevel: "off",
    });
  });

  test("getAgent returns the same Agent for the same user", async () => {
    const a1 = await manager.getAgent("user-1");
    const a2 = await manager.getAgent("user-1");
    expect(a1).toBe(a2);
  });

  test("evictUser drops the cached Agent", async () => {
    const a1 = await manager.getAgent("user-1");
    manager.evictUser("user-1");
    const a2 = await manager.getAgent("user-1");
    expect(a2).not.toBe(a1);
  });

  test("evictAll drops every cached Agent", async () => {
    const a1 = await manager.getAgent("user-1");
    const b1 = await manager.getAgent("user-2");
    manager.evictAll();
    expect(await manager.getAgent("user-1")).not.toBe(a1);
    expect(await manager.getAgent("user-2")).not.toBe(b1);
  });
});
```

- [ ] **Step 2: Write `src/agent/manager.ts`**

```ts
// src/agent/manager.ts
import type { Agent } from "@earendil-works/pi-agent-core";
import type { Model } from "@earendil-works/pi-ai";
import type { Persistence } from "./persistence";
import type { Identity } from "./identity";
import type { AuditFn } from "../shared/audit";
import type { ThinkingLevel } from "../shared/types";
import { createAgent } from "./factory";
import { getDefaultTools } from "./identity";

export interface AgentManagerOptions {
  home: string;
  identity: Identity;
  persistence: Persistence;
  audit: AuditFn;
  defaultModel: Model<any>;
  defaultThinkingLevel: ThinkingLevel;
}

export class AgentManager {
  private cache = new Map<string, Agent>();

  constructor(private readonly opts: AgentManagerOptions) {}

  async getAgent(userId: string): Promise<Agent> {
    const cached = this.cache.get(userId);
    if (cached) return cached;

    const loaded = await this.opts.persistence.load(userId);
    const initialMessages = loaded != null ? (loaded as any).messages ?? undefined : undefined;

    const agent = createAgent({
      userId,
      systemPrompt: this.opts.identity.getSystemPrompt(),
      model: this.opts.defaultModel,
      tools: getDefaultTools(this.opts.home, userId),
      thinkingLevel: this.opts.defaultThinkingLevel,
      initialMessages,
      audit: this.opts.audit,
    });

    agent.subscribe(async (event) => {
      if (event.type === "agent_end") {
        try {
          await this.opts.persistence.save(userId, {
            systemPrompt: agent.state.systemPrompt,
            messages: agent.state.messages,
          });
          await this.opts.audit("agent_end", userId, { messageCount: agent.state.messages.length });
        } catch (err) {
          await this.opts.audit("error", userId, { phase: "persistence.save", err: String(err) });
        }
      }
    });

    this.cache.set(userId, agent);
    return agent;
  }

  evictUser(userId: string): void {
    this.cache.delete(userId);
  }

  evictAll(): void {
    this.cache.clear();
  }

  has(userId: string): boolean {
    return this.cache.has(userId);
  }
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/manager.test.ts
```

Expected: PASS (3/3).

- [ ] **Step 4: Commit**

```bash
git add src/agent/manager.ts tests/unit/manager.test.ts
git commit -m "Add AgentManager with per-user caching, evictUser/All, agent_end persistence"
```

---

### T22: `src/agent/factory.ts` — Telegram-side helper getSavedSessionList

We need a way to ask "does any persisted user exist?" without instantiating Agents. Skipping for SP1 — `AgentManager.has(userId)` is enough. **No code change.** Mark this task as done by leaving a comment in `manager.ts`:

- [ ] **Step 1: Add comment to `src/agent/manager.ts`**

Append:

```ts
// SP1 does not need a "list all persisted users" helper — Telegram message
// arrival drives user discovery. If SP3 (memory) or SP5 (admin commands)
// needs enumeration, add it to AgentManager then.
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/agent/manager.ts
git commit -m "Document SP1 scope: no persisted-user enumeration in AgentManager"
```

---

## Phase H — Channel base, security, formatting (T23–T25)

### T23: `src/channels/ChannelInterface.ts`

**Files:**
- Create: `src/channels/ChannelInterface.ts`

- [ ] **Step 1: Write the interface**

```ts
// src/channels/ChannelInterface.ts
export interface Channel {
  start(): Promise<void>;
  stop(): Promise<void>;
}
```

- [ ] **Step 2: Commit**

```bash
git add src/channels/ChannelInterface.ts
git commit -m "Add ChannelInterface.start/stop seam"
```

---

### T24: `src/channels/telegram/security.ts` — allowlist + token bucket

**Files:**
- Create: `src/channels/telegram/security.ts`
- Test: `tests/unit/security.test.ts`
- Reference: `/mnt/Shared/Code/projects/claude-telegram-bot/src/security.ts`

- [ ] **Step 1: Skim the current bot's `security.ts`**

```bash
cat /mnt/Shared/Code/projects/claude-telegram-bot/src/security.ts
```

Port the token-bucket logic; drop the path-validation and command-safety helpers (SP1 doesn't expose tools that need them).

- [ ] **Step 2: Write failing test**

```ts
// tests/unit/security.test.ts
import { describe, test, expect } from "bun:test";
import { Allowlist, RateLimiter } from "../../src/channels/telegram/security";

describe("Allowlist", () => {
  test("denies unknown user", () => {
    const al = new Allowlist(new Set(["1", "2"]));
    expect(al.allow("3")).toBe(false);
    expect(al.allow("1")).toBe(true);
  });
});

describe("RateLimiter (token bucket)", () => {
  test("allows up to capacity, then denies until refill", () => {
    const rl = new RateLimiter({ capacity: 3, refillPerSec: 1, now: () => 0 });
    expect(rl.check("u")).toBe(true);
    expect(rl.check("u")).toBe(true);
    expect(rl.check("u")).toBe(true);
    expect(rl.check("u")).toBe(false);
  });

  test("refills over time", () => {
    let now = 0;
    const rl = new RateLimiter({ capacity: 1, refillPerSec: 1, now: () => now });
    expect(rl.check("u")).toBe(true);
    expect(rl.check("u")).toBe(false);
    now = 1500;  // 1.5s later → 1 token refilled (capped at capacity)
    expect(rl.check("u")).toBe(true);
  });
});
```

- [ ] **Step 3: Write `src/channels/telegram/security.ts`**

```ts
// src/channels/telegram/security.ts

export class Allowlist {
  constructor(private readonly users: ReadonlySet<string>) {}
  allow(userId: string): boolean { return this.users.has(userId); }
}

export interface RateLimiterOptions {
  capacity: number;
  refillPerSec: number;
  now?: () => number;
}

interface Bucket { tokens: number; lastRefillMs: number; }

export class RateLimiter {
  private buckets = new Map<string, Bucket>();
  private readonly now: () => number;

  constructor(private readonly opts: RateLimiterOptions) {
    this.now = opts.now ?? (() => Date.now());
  }

  check(userId: string): boolean {
    const t = this.now();
    let b = this.buckets.get(userId);
    if (!b) {
      b = { tokens: this.opts.capacity, lastRefillMs: t };
      this.buckets.set(userId, b);
    } else {
      const elapsedSec = (t - b.lastRefillMs) / 1000;
      const refill = elapsedSec * this.opts.refillPerSec;
      b.tokens = Math.min(this.opts.capacity, b.tokens + refill);
      b.lastRefillMs = t;
    }
    if (b.tokens >= 1) { b.tokens -= 1; return true; }
    return false;
  }
}
```

- [ ] **Step 4: Tests pass**

```bash
bun test tests/unit/security.test.ts
```

Expected: PASS (3/3).

- [ ] **Step 5: Commit**

```bash
git add src/channels/telegram/security.ts tests/unit/security.test.ts
git commit -m "Add Allowlist + RateLimiter (token bucket, injectable clock)"
```

---

### T25: `src/channels/telegram/formatting.ts` — Markdown → Telegram-safe HTML

**Files:**
- Create: `src/channels/telegram/formatting.ts`
- Test: `tests/unit/formatting.test.ts`
- Reference: `/mnt/Shared/Code/projects/claude-telegram-bot/src/formatting.ts`

- [ ] **Step 1: Read the current bot's formatting**

```bash
cat /mnt/Shared/Code/projects/claude-telegram-bot/src/formatting.ts
```

Port verbatim (it's ~200 lines, already well-tested via the current bot). Drop the table renderer — SP1 doesn't need it.

- [ ] **Step 2: Write a small regression test**

```ts
// tests/unit/formatting.test.ts
import { describe, test, expect } from "bun:test";
import { markdownToTelegramHtml } from "../../src/channels/telegram/formatting";

describe("markdownToTelegramHtml", () => {
  test("preserves plain text", () => {
    expect(markdownToTelegramHtml("hello world")).toBe("hello world");
  });

  test("converts bold and italic", () => {
    expect(markdownToTelegramHtml("**bold** and *italic*")).toBe("<b>bold</b> and <i>italic</i>");
  });

  test("escapes raw HTML characters", () => {
    expect(markdownToTelegramHtml("<b>x</b>")).toContain("&lt;");
  });

  test("renders inline code", () => {
    expect(markdownToTelegramHtml("use `foo` here")).toBe("use <code>foo</code> here");
  });
});
```

- [ ] **Step 3: Port the formatter** (use the current bot's source as the basis; rename the exported function to `markdownToTelegramHtml` for clarity).

- [ ] **Step 4: Tests pass**

```bash
bun test tests/unit/formatting.test.ts
```

Expected: PASS (4/4). If the original implementation differs in escape behavior, adjust the test rather than changing the port — preserve original semantics.

- [ ] **Step 5: Commit**

```bash
git add src/channels/telegram/formatting.ts tests/unit/formatting.test.ts
git commit -m "Port markdown→Telegram HTML formatter from claude-telegram-bot"
```

---

## Phase I — Streaming state (T26)

### T26: `src/channels/telegram/streaming.ts` — consolidated callback

**Files:**
- Create: `src/channels/telegram/streaming.ts`
- Test: `tests/unit/streaming.test.ts`
- Reference: `/mnt/Shared/Code/projects/claude-telegram-bot/src/handlers/streaming.ts`

- [ ] **Step 1: Read the current bot's streaming pattern**

```bash
cat /mnt/Shared/Code/projects/claude-telegram-bot/src/handlers/streaming.ts
```

Adapt the *pattern* — pi-agent-core has different event types than the Claude SDK (we confirmed `agent_start`, `turn_start`, `message_start`, `message_update`, `message_end`, `tool_execution_start/end`, `turn_end`, `agent_end`).

- [ ] **Step 2: Write failing test (consolidation behavior)**

```ts
// tests/unit/streaming.test.ts
import { describe, test, expect } from "bun:test";
import { createStreamingState } from "../../src/channels/telegram/streaming";

describe("StreamingState", () => {
  test("consolidates multiple text deltas into one debounced edit", async () => {
    const edits: string[] = [];
    const now = { value: 0 };
    const state = createStreamingState({
      sendInitial: async () => 42,
      edit: async (_msgId, text) => { edits.push(text); },
      debounceMs: 500,
      now: () => now.value,
    });

    await state.start();
    now.value = 100; await state.onTextDelta("hel");
    now.value = 200; await state.onTextDelta("lo");
    // Within debounce window — no edit yet
    expect(edits).toHaveLength(0);

    now.value = 700; await state.onTextDelta(" world");
    // Past debounce window — one consolidated edit
    expect(edits).toHaveLength(1);
    expect(edits[0]).toContain("hello world");

    await state.finalize("hello world.");
    expect(edits[edits.length - 1]).toContain("hello world.");
  });

  test("tool_execution events become status lines under the streamed text", async () => {
    const edits: string[] = [];
    const state = createStreamingState({
      sendInitial: async () => 1,
      edit: async (_id, text) => { edits.push(text); },
      debounceMs: 0,
      now: () => 0,
    });
    await state.start();
    await state.onToolStart("current_time");
    await state.onToolEnd("current_time", false);
    expect(edits.some(e => e.includes("Calling current_time"))).toBe(true);
    expect(edits.some(e => e.includes("Done: current_time"))).toBe(true);
  });
});
```

- [ ] **Step 3: Write `src/channels/telegram/streaming.ts`**

```ts
// src/channels/telegram/streaming.ts

export interface StreamingStateOptions {
  sendInitial: () => Promise<number>;            // returns Telegram message_id of the "Thinking..." stub
  edit: (msgId: number, text: string) => Promise<void>;
  debounceMs?: number;
  now?: () => number;
}

export interface StreamingState {
  start(): Promise<void>;
  onTextDelta(delta: string): Promise<void>;
  onToolStart(name: string): Promise<void>;
  onToolEnd(name: string, isError: boolean): Promise<void>;
  finalize(finalText: string): Promise<void>;
}

export function createStreamingState(opts: StreamingStateOptions): StreamingState {
  const debounce = opts.debounceMs ?? 500;
  const now = opts.now ?? (() => Date.now());

  let msgId: number | null = null;
  let buffer = "";
  let statusLines: string[] = [];
  let lastEditMs = 0;

  const render = () => {
    const parts = [];
    if (buffer.length > 0) parts.push(buffer);
    if (statusLines.length > 0) parts.push("\n" + statusLines.join("\n"));
    return parts.join("");
  };

  const maybeEdit = async () => {
    if (msgId == null) return;
    if (now() - lastEditMs < debounce) return;
    await opts.edit(msgId, render() || "Thinking...");
    lastEditMs = now();
  };

  return {
    async start() {
      msgId = await opts.sendInitial();
      lastEditMs = now();
    },

    async onTextDelta(delta) {
      buffer += delta;
      await maybeEdit();
    },

    async onToolStart(name) {
      statusLines.push(`Calling ${name}...`);
      await maybeEdit();
    },

    async onToolEnd(name, isError) {
      const idx = statusLines.findIndex(l => l === `Calling ${name}...`);
      const label = isError ? `Failed: ${name}` : `Done: ${name}`;
      if (idx >= 0) statusLines[idx] = label; else statusLines.push(label);
      await maybeEdit();
    },

    async finalize(finalText) {
      if (msgId == null) return;
      buffer = finalText;
      lastEditMs = 0;  // force final edit through
      await maybeEdit();
    },
  };
}
```

- [ ] **Step 4: Tests pass**

```bash
bun test tests/unit/streaming.test.ts
```

Expected: PASS (2/2).

- [ ] **Step 5: Commit**

```bash
git add src/channels/telegram/streaming.ts tests/unit/streaming.test.ts
git commit -m "Add StreamingState: debounced edit, tool status lines, finalize forces flush"
```

---

## Phase J — Telegram handlers (T27–T31)

### T27: `src/channels/telegram/handlers/stub.ts` — polite stubs

**Files:**
- Create: `src/channels/telegram/handlers/stub.ts`

- [ ] **Step 1: Write the per-type stub map**

```ts
// src/channels/telegram/handlers/stub.ts
import type { RawInput } from "../../../agent/inputs/types";

const stubs: Record<Exclude<RawInput["kind"], "text" | "photo">, string> = {
  voice: "Voice isn't supported yet — that's coming in a later version. Send me text and I'll help.",
  audio: "Audio files aren't supported yet. Text works.",
  document: "Documents aren't supported yet. Paste the text and I'll help.",
  video: "Video isn't supported yet. Send text or a photo.",
  raw_file: "Raw file forwarding isn't supported yet.",
};

export function stubFor(kind: Exclude<RawInput["kind"], "text" | "photo">): string {
  return stubs[kind];
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/channels/telegram/handlers/stub.ts
git commit -m "Add per-input-type polite stub map (no emojis)"
```

---

### T28: `src/channels/telegram/handlers/text.ts` — text → agent.prompt (or steer)

**Files:**
- Create: `src/channels/telegram/handlers/text.ts`

This handler embodies the **central message-handling pattern**: auth → rate limit → kind-specific processing → router → AgentManager → either `prompt` or `steer` depending on `state.isStreaming`. T29 (photo) and the bootstrap handler (T34) follow the same shape.

- [ ] **Step 1: Write `src/channels/telegram/handlers/text.ts`**

```ts
// src/channels/telegram/handlers/text.ts
import type { Context } from "grammy";
import type { InputProcessorRegistry } from "../../../agent/inputs/registry";
import type { AgentManager } from "../../../agent/manager";
import type { Router } from "../../../agent/router/router";
import type { Allowlist, RateLimiter } from "../security";
import type { AuditFn } from "../../../shared/audit";
import { createStreamingState } from "../streaming";
import { markdownToTelegramHtml } from "../formatting";

export interface TextHandlerDeps {
  allowlist: Allowlist;
  rateLimit: RateLimiter;
  audit: AuditFn;
  registry: InputProcessorRegistry;
  router: Router;
  agents: AgentManager;
}

export function makeTextHandler(deps: TextHandlerDeps) {
  return async (ctx: Context) => {
    const userId = ctx.from?.id != null ? String(ctx.from.id) : null;
    const text = ctx.message?.text;
    if (!userId || !text) return;

    if (!deps.allowlist.allow(userId)) {
      await deps.audit("auth_denied", userId, { update: ctx.update.update_id });
      return;  // silent drop
    }
    if (!deps.rateLimit.check(userId)) {
      await deps.audit("rate_limited", userId, {});
      await ctx.reply("Slow down — try again in a few seconds.");
      return;
    }

    const ts = (ctx.message?.date ?? Math.floor(Date.now() / 1000)) * 1000;
    const rawInput = { kind: "text" as const, payload: { text }, meta: { userId, channel: "telegram" as const, timestamp: ts } };

    const userMessages = await deps.registry.process(rawInput, {
      async fetchFile() { throw new Error("text processor doesn't fetch files"); },
    });
    const decision = await deps.router.decide(rawInput, userId, { recentMessageCount: 0 });

    const agent = await deps.agents.getAgent(userId);
    // (systemPrompt refresh after /persona is handled by AgentManager.evictAll(),
    //  which forces getAgent to rebuild from identity.getSystemPrompt() on next call.)
    agent.state.model = decision.model;
    agent.state.tools = decision.tools;
    agent.state.thinkingLevel = decision.thinkingLevel;

    if (agent.state.isStreaming) {
      agent.steer(userMessages[0]!);
      await ctx.reply("Added — will fold into the current reply.");
      return;
    }

    await deps.audit("prompt_received", userId, { length: text.length });

    const stream = createStreamingState({
      sendInitial: async () => (await ctx.reply("Thinking...")).message_id,
      edit: async (msgId, t) => { await ctx.api.editMessageText(ctx.chat!.id, msgId, t, { parse_mode: "HTML" }); },
    });
    await stream.start();

    const unsub = agent.subscribe(async (ev) => {
      if (ev.type === "message_update" && (ev as any).assistantMessageEvent?.type === "text_delta") {
        const delta = (ev as any).assistantMessageEvent.delta as string;
        await stream.onTextDelta(delta);
      } else if (ev.type === "tool_execution_start") {
        await stream.onToolStart((ev as any).toolName);
      } else if (ev.type === "tool_execution_end") {
        await stream.onToolEnd((ev as any).toolName, (ev as any).isError);
      } else if (ev.type === "agent_end") {
        const lastAssistant = agent.state.messages.slice().reverse().find((m: any) => m.role === "assistant");
        const finalText = lastAssistant ? extractText(lastAssistant) : "";
        await stream.finalize(markdownToTelegramHtml(finalText));
      }
    });

    try {
      await agent.prompt(userMessages[0]!);
    } catch (err) {
      await deps.audit("error", userId, { phase: "prompt", err: String(err) });
      await ctx.reply(`Provider error: ${(err as any).message}. Try /retry or /reset.`);
    } finally {
      unsub();
    }
  };
}

function extractText(m: any): string {
  if (typeof m.content === "string") return m.content;
  if (Array.isArray(m.content)) {
    return m.content.filter((b: any) => b.type === "text").map((b: any) => b.text).join("");
  }
  return "";
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0 (warnings about pi-agent-core event shapes are fine if any). If the assistantMessageEvent shape differs, consult `node_modules/@earendil-works/pi-agent-core/dist/index.d.ts`.

- [ ] **Step 3: Commit**

```bash
git add src/channels/telegram/handlers/text.ts
git commit -m "Add text handler: auth, rate-limit, route, agent.prompt or steer, streaming"
```

---

### T29: `src/channels/telegram/handlers/photo.ts` — photo with media-group buffering

**Files:**
- Create: `src/channels/telegram/handlers/photo.ts`
- Reference: `/mnt/Shared/Code/projects/claude-telegram-bot/src/handlers/photo.ts` (for media-group buffering pattern)

The structure is identical to T28's text handler — auth, rate limit, registry, router, AgentManager, steer-or-prompt — *except* the input building step buffers media-group photos for 1 second before processing.

- [ ] **Step 1: Read the current bot's photo handler**

```bash
cat /mnt/Shared/Code/projects/claude-telegram-bot/src/handlers/photo.ts
```

Note the `media_group_id` buffering: when present, wait 1 second for additional photos in the group before kicking off processing.

- [ ] **Step 2: Write `src/channels/telegram/handlers/photo.ts`**

```ts
// src/channels/telegram/handlers/photo.ts
import type { Context } from "grammy";
import type { TextHandlerDeps } from "./text";
import { createStreamingState } from "../streaming";
import { markdownToTelegramHtml } from "../formatting";

interface PendingGroup {
  fileIds: string[];
  mimeTypes: string[];
  caption?: string;
  ts: number;
  timer: ReturnType<typeof setTimeout>;
}

const pending = new Map<string, PendingGroup>();
const MEDIA_GROUP_BUFFER_MS = 1000;

export function makePhotoHandler(deps: TextHandlerDeps) {
  return async (ctx: Context) => {
    const userId = ctx.from?.id != null ? String(ctx.from.id) : null;
    if (!userId) return;

    if (!deps.allowlist.allow(userId)) { await deps.audit("auth_denied", userId, {}); return; }
    if (!deps.rateLimit.check(userId)) { await deps.audit("rate_limited", userId, {}); await ctx.reply("Slow down — try again in a few seconds."); return; }

    const photos = ctx.message?.photo ?? [];
    if (photos.length === 0) return;
    const largest = photos[photos.length - 1]!;
    const fileId = largest.file_id;
    const caption = ctx.message?.caption ?? undefined;
    const mediaGroupId = ctx.message?.media_group_id;
    const ts = (ctx.message?.date ?? Math.floor(Date.now() / 1000)) * 1000;

    if (mediaGroupId) {
      const key = `${userId}:${mediaGroupId}`;
      let entry = pending.get(key);
      if (!entry) {
        entry = { fileIds: [], mimeTypes: [], ts, timer: null as any };
        entry.timer = setTimeout(() => {
          pending.delete(key);
          fire(entry!.fileIds, entry!.mimeTypes, entry!.caption, entry!.ts);
        }, MEDIA_GROUP_BUFFER_MS);
        pending.set(key, entry);
      } else {
        clearTimeout(entry.timer);
        entry.timer = setTimeout(() => {
          pending.delete(key);
          fire(entry!.fileIds, entry!.mimeTypes, entry!.caption, entry!.ts);
        }, MEDIA_GROUP_BUFFER_MS);
      }
      entry.fileIds.push(fileId);
      entry.mimeTypes.push("image/jpeg");
      if (caption && !entry.caption) entry.caption = caption;
      return;
    }

    await fire([fileId], ["image/jpeg"], caption, ts);

    async function fire(fileIds: string[], mimeTypes: string[], cap: string | undefined, t: number) {
      const rawInput = {
        kind: "photo" as const,
        payload: { fileIds, caption: cap, mimeTypes },
        meta: { userId: userId!, channel: "telegram" as const, timestamp: t },
      };

      const userMessages = await deps.registry.process(rawInput, {
        async fetchFile(fid) {
          const file = await ctx.api.getFile(fid);
          const url = `https://api.telegram.org/file/bot${(ctx.api as any).token}/${file.file_path}`;
          const res = await fetch(url);
          const buf = new Uint8Array(await res.arrayBuffer());
          return { data: buf, mimeType: "image/jpeg" };
        },
      });
      const decision = await deps.router.decide(rawInput, userId!, { recentMessageCount: 0 });

      const agent = await deps.agents.getAgent(userId!);
      agent.state.model = decision.model;
      agent.state.tools = decision.tools;
      agent.state.thinkingLevel = decision.thinkingLevel;

      if (agent.state.isStreaming) {
        agent.steer(userMessages[0]!);
        await ctx.reply("Added — will fold into the current reply.");
        return;
      }

      await deps.audit("prompt_received", userId!, { kind: "photo", count: fileIds.length });

      const stream = createStreamingState({
        sendInitial: async () => (await ctx.reply("Thinking...")).message_id,
        edit: async (msgId, txt) => { await ctx.api.editMessageText(ctx.chat!.id, msgId, txt, { parse_mode: "HTML" }); },
      });
      await stream.start();

      const unsub = agent.subscribe(async (ev) => {
        if (ev.type === "message_update" && (ev as any).assistantMessageEvent?.type === "text_delta") {
          await stream.onTextDelta((ev as any).assistantMessageEvent.delta);
        } else if (ev.type === "tool_execution_start") await stream.onToolStart((ev as any).toolName);
        else if (ev.type === "tool_execution_end") await stream.onToolEnd((ev as any).toolName, (ev as any).isError);
        else if (ev.type === "agent_end") {
          const lastAssistant = agent.state.messages.slice().reverse().find((m: any) => m.role === "assistant");
          const finalText = lastAssistant ? extractText(lastAssistant) : "";
          await stream.finalize(markdownToTelegramHtml(finalText));
        }
      });

      try { await agent.prompt(userMessages[0]!); }
      catch (err) { await deps.audit("error", userId!, { phase: "prompt", err: String(err) }); await ctx.reply(`Provider error: ${(err as any).message}. Try /retry or /reset.`); }
      finally { unsub(); }
    }
  };
}

function extractText(m: any): string {
  if (typeof m.content === "string") return m.content;
  if (Array.isArray(m.content)) return m.content.filter((b: any) => b.type === "text").map((b: any) => b.text).join("");
  return "";
}
```

- [ ] **Step 3: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 4: Commit**

```bash
git add src/channels/telegram/handlers/photo.ts
git commit -m "Add photo handler with 1s media-group buffer and streaming"
```

---

### T30: Refactor text/photo handlers to share the streaming subscription wiring

Both T28 and T29 contain near-identical `subscribe → stream.onX → unsub` wiring. Extract once.

**Files:**
- Create: `src/channels/telegram/handlers/stream-bridge.ts`
- Modify: `src/channels/telegram/handlers/text.ts`
- Modify: `src/channels/telegram/handlers/photo.ts`

- [ ] **Step 1: Extract the bridge**

```ts
// src/channels/telegram/handlers/stream-bridge.ts
import type { Agent } from "@earendil-works/pi-agent-core";
import type { StreamingState } from "../streaming";
import { markdownToTelegramHtml } from "../formatting";

function extractText(m: any): string {
  if (typeof m.content === "string") return m.content;
  if (Array.isArray(m.content)) return m.content.filter((b: any) => b.type === "text").map((b: any) => b.text).join("");
  return "";
}

export function bridge(agent: Agent, stream: StreamingState): () => void {
  return agent.subscribe(async (ev) => {
    if (ev.type === "message_update" && (ev as any).assistantMessageEvent?.type === "text_delta") {
      await stream.onTextDelta((ev as any).assistantMessageEvent.delta);
    } else if (ev.type === "tool_execution_start") {
      await stream.onToolStart((ev as any).toolName);
    } else if (ev.type === "tool_execution_end") {
      await stream.onToolEnd((ev as any).toolName, (ev as any).isError);
    } else if (ev.type === "agent_end") {
      const lastAssistant = agent.state.messages.slice().reverse().find((m: any) => m.role === "assistant");
      const finalText = lastAssistant ? extractText(lastAssistant) : "";
      await stream.finalize(markdownToTelegramHtml(finalText));
    }
  });
}
```

- [ ] **Step 2: Replace the inline subscribe calls in `text.ts` and `photo.ts` with `bridge(agent, stream)`. Remove the local `extractText` helpers.**

- [ ] **Step 3: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 4: Commit**

```bash
git add src/channels/telegram/handlers/{text.ts,photo.ts,stream-bridge.ts}
git commit -m "Extract agent→stream bridge to a shared helper used by text+photo"
```

---

### T31: `src/channels/telegram/handlers/commands.ts` — /start /reset /stop /retry /status /model /persona

**Files:**
- Create: `src/channels/telegram/handlers/commands.ts`

The `/persona` body that *runs the bootstrap LLM call* depends on T34's `bootstrapPersona` — wire it as a callback for now, fill in the wiring in T35.

- [ ] **Step 1: Write the command handlers**

```ts
// src/channels/telegram/handlers/commands.ts
import type { Context } from "grammy";
import type { AgentManager } from "../../../agent/manager";
import type { Persistence } from "../../../agent/persistence";
import type { Identity } from "../../../agent/identity";
import type { Allowlist } from "../security";
import type { AppConfig } from "../../../shared/types";
import { getModel, getModels } from "@earendil-works/pi-ai";

export interface CommandDeps {
  allowlist: Allowlist;
  agents: AgentManager;
  persistence: Persistence;
  identity: Identity;
  config: AppConfig;
  regeneratePersona: (description: string) => Promise<string>;  // wired in T35
  beginPersonaDialog: (userId: string) => void;                  // wired in T35
}

export function registerCommands(bot: any /* grammY Bot */, deps: CommandDeps) {
  const userIdOf = (ctx: Context) => ctx.from?.id != null ? String(ctx.from.id) : null;
  const allow = (ctx: Context) => {
    const id = userIdOf(ctx);
    return id && deps.allowlist.allow(id) ? id : null;
  };

  bot.command("start", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    await ctx.reply(deps.config.greeting ?? "Hi.");
  });

  bot.command("reset", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    // Archive default.json by reading and writing-to a renamed path
    // (delegated to persistence in a later refactor; for SP1 we just evict)
    deps.agents.evictUser(id);
    await ctx.reply("Conversation cleared.");
  });

  bot.command("stop", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    const agent = await deps.agents.getAgent(id);
    agent.abort();
    await ctx.reply("Stopped.");
  });

  bot.command("retry", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    const agent = await deps.agents.getAgent(id);
    try { await agent.continue(); } catch (e) { await ctx.reply(`Can't retry: ${(e as Error).message}`); }
  });

  bot.command("status", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    const agent = await deps.agents.getAgent(id);
    const lines = [
      `Provider: ${(agent.state.model as any).provider}`,
      `Model: ${(agent.state.model as any).id}`,
      `Messages: ${agent.state.messages.length}`,
      `Streaming: ${agent.state.isStreaming}`,
    ];
    await ctx.reply(lines.join("\n"));
  });

  bot.command("model", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    const arg = (ctx.message?.text ?? "").replace(/^\/model\s*/, "").trim();
    if (!arg) {
      const provider = deps.config.providers.defaultProvider;
      const list = getModels(provider as any).map((m: any) => m.id).slice(0, 30);
      await ctx.reply(`Models for ${provider}:\n${list.join("\n")}`);
      return;
    }
    const [provider, modelId] = arg.split(/\s+/, 2);
    if (!provider || !modelId) { await ctx.reply("Usage: /model <provider> <id>"); return; }
    try {
      const m = getModel(provider as any, modelId as any);
      const agent = await deps.agents.getAgent(id);
      agent.state.model = m;
      await ctx.reply(`Model set to ${provider}/${modelId}.`);
    } catch {
      const candidates = getModels(provider as any).map((m: any) => m.id);
      const close = closestMatch(modelId, candidates);
      await ctx.reply(close ? `Unknown model '${modelId}'. Did you mean '${close}'?` : `Unknown model '${modelId}'.`);
    }
  });

  bot.command("persona", async (ctx: Context) => {
    const id = allow(ctx); if (!id) return;
    const inline = (ctx.message?.text ?? "").replace(/^\/persona\s*/, "").trim();
    if (inline.length > 0) {
      const prompt = await deps.regeneratePersona(inline);
      await ctx.reply(`New persona:\n\n${prompt}`);
    } else {
      deps.beginPersonaDialog(id);
      await ctx.reply("Describe what kind of assistant you'd like — 1–2 sentences.");
    }
  });
}

function closestMatch(input: string, candidates: string[]): string | null {
  if (candidates.length === 0) return null;
  let best = candidates[0]!;
  let bestD = levenshtein(input, best);
  for (const c of candidates.slice(1)) {
    const d = levenshtein(input, c);
    if (d < bestD) { bestD = d; best = c; }
  }
  return bestD <= Math.max(2, Math.floor(input.length / 3)) ? best : null;
}

function levenshtein(a: string, b: string): number {
  if (a === b) return 0;
  const m = a.length, n = b.length;
  if (m === 0) return n; if (n === 0) return m;
  let prev = new Array(n + 1).fill(0).map((_, i) => i);
  for (let i = 1; i <= m; i++) {
    const cur = [i];
    for (let j = 1; j <= n; j++) {
      cur[j] = a[i - 1] === b[j - 1] ? prev[j - 1]! : 1 + Math.min(prev[j]!, prev[j - 1]!, cur[j - 1]!);
    }
    prev = cur;
  }
  return prev[n]!;
}
```

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/channels/telegram/handlers/commands.ts
git commit -m "Add /start /reset /stop /retry /status /model /persona command handlers"
```

---

## Phase K — Config, bot wiring, index (T32–T33)

### T32: `src/config.ts` — env loading + provider-aware default model

**Files:**
- Create: `src/config.ts`
- Test: `tests/unit/config.test.ts`

- [ ] **Step 1: Write failing test**

```ts
// tests/unit/config.test.ts
import { describe, test, expect } from "bun:test";
import { loadConfig } from "../../src/config";

describe("loadConfig", () => {
  test("picks anthropic+haiku-4-5 when only ANTHROPIC_API_KEY is set", () => {
    const cfg = loadConfig({
      TELEGRAM_BOT_TOKEN: "t",
      TELEGRAM_ALLOWED_USERS: "111,222",
      ANTHROPIC_API_KEY: "x",
    });
    expect(cfg.providers.defaultProvider).toBe("anthropic");
    expect(cfg.providers.defaultModel).toBe("claude-haiku-4-5-20251001");
    expect(cfg.telegram.allowedUsers.has("111")).toBe(true);
  });

  test("honors PADDELINO_PROVIDER_PRIORITY", () => {
    const cfg = loadConfig({
      TELEGRAM_BOT_TOKEN: "t",
      TELEGRAM_ALLOWED_USERS: "111",
      ANTHROPIC_API_KEY: "x",
      OPENAI_API_KEY: "y",
      PADDELINO_PROVIDER_PRIORITY: "openai,anthropic",
    });
    expect(cfg.providers.defaultProvider).toBe("openai");
  });

  test("throws on missing TELEGRAM_BOT_TOKEN", () => {
    expect(() => loadConfig({ TELEGRAM_ALLOWED_USERS: "1", ANTHROPIC_API_KEY: "x" })).toThrow(/TELEGRAM_BOT_TOKEN/);
  });

  test("throws when no provider key is set", () => {
    expect(() => loadConfig({ TELEGRAM_BOT_TOKEN: "t", TELEGRAM_ALLOWED_USERS: "1" })).toThrow(/provider/);
  });
});
```

- [ ] **Step 2: Write `src/config.ts`**

```ts
// src/config.ts
import { homedir } from "node:os";
import { join } from "node:path";
import type { AppConfig, ProviderId } from "./shared/types";

const DEFAULT_MODEL: Record<ProviderId, string> = {
  anthropic: "claude-haiku-4-5-20251001",
  openai: "gpt-5-mini",
  openrouter: "deepseek/deepseek-chat-v3",
  gemini: "gemini-2.5-flash",
};

const PROVIDER_FROM_ENV: Record<ProviderId, string> = {
  anthropic: "ANTHROPIC_API_KEY",
  openai: "OPENAI_API_KEY",
  openrouter: "OPENROUTER_API_KEY",
  gemini: "GEMINI_API_KEY",
};

const DEFAULT_PRIORITY: ProviderId[] = ["anthropic", "openai", "openrouter", "gemini"];

export function loadConfig(env: Record<string, string | undefined>): AppConfig {
  const token = env.TELEGRAM_BOT_TOKEN;
  if (!token) throw new Error("Missing TELEGRAM_BOT_TOKEN");

  const allowedRaw = env.TELEGRAM_ALLOWED_USERS ?? "";
  const allowedUsers = new Set(allowedRaw.split(",").map(s => s.trim()).filter(Boolean));
  if (allowedUsers.size === 0) throw new Error("Missing TELEGRAM_ALLOWED_USERS");

  const priorityRaw = env.PADDELINO_PROVIDER_PRIORITY ?? DEFAULT_PRIORITY.join(",");
  const priority = priorityRaw.split(",").map(s => s.trim() as ProviderId).filter(p => p in DEFAULT_MODEL);

  const available = priority.filter(p => Boolean(env[PROVIDER_FROM_ENV[p]]));
  if (available.length === 0) {
    throw new Error("No provider configured. Set at least one of: " + Object.values(PROVIDER_FROM_ENV).join(", "));
  }

  const defaultProvider = (env.PADDELINO_DEFAULT_PROVIDER as ProviderId) ?? available[0]!;
  const defaultModel = env.PADDELINO_DEFAULT_MODEL ?? DEFAULT_MODEL[defaultProvider];
  const defaultThinkingLevel = (env.PADDELINO_DEFAULT_THINKING_LEVEL as any) ?? "off";

  const home = env.PADDELINO_HOME ?? join(homedir(), ".paddelino");

  return {
    telegram: { token, allowedUsers },
    providers: { available, priority, defaultProvider, defaultModel, defaultThinkingLevel },
    paths: { home },
    rateLimit: {
      capacity: Number(env.PADDELINO_RATE_LIMIT_CAPACITY ?? "10"),
      refillPerSec: Number(env.PADDELINO_RATE_LIMIT_REFILL_PER_SEC ?? "0.5"),
    },
    greeting: env.PADDELINO_GREETING,
  };
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/config.test.ts
```

Expected: PASS (4/4).

- [ ] **Step 4: Commit**

```bash
git add src/config.ts tests/unit/config.test.ts
git commit -m "Add loadConfig with provider-aware default model selection"
```

---

### T33: `src/channels/telegram/bot.ts` and `src/index.ts` — wire everything

**Files:**
- Create: `src/channels/telegram/bot.ts`
- Modify: `src/index.ts`

- [ ] **Step 1: Write `src/channels/telegram/bot.ts`**

```ts
// src/channels/telegram/bot.ts
import { Bot } from "grammy";
import { run } from "@grammyjs/runner";
import type { Channel } from "../ChannelInterface";
import type { TextHandlerDeps } from "./handlers/text";
import { makeTextHandler } from "./handlers/text";
import { makePhotoHandler } from "./handlers/photo";
import { stubFor } from "./handlers/stub";
import type { CommandDeps } from "./handlers/commands";
import { registerCommands } from "./handlers/commands";

export interface TelegramChannelDeps extends TextHandlerDeps, CommandDeps {
  token: string;
  onAnyMessage?: (userId: string, ctx: any) => Promise<boolean>;  // returns true if handled (used for bootstrap)
}

export function createTelegramChannel(deps: TelegramChannelDeps): Channel {
  const bot = new Bot(deps.token);

  registerCommands(bot, deps);

  const textHandler = makeTextHandler(deps);
  const photoHandler = makePhotoHandler(deps);

  bot.on("message:text", async (ctx) => {
    if (ctx.message.text.startsWith("/")) return;  // commands handled above
    const userId = ctx.from?.id != null ? String(ctx.from.id) : null;
    if (userId && deps.onAnyMessage) {
      const handled = await deps.onAnyMessage(userId, ctx);
      if (handled) return;
    }
    await textHandler(ctx);
  });

  bot.on("message:photo", async (ctx) => {
    const userId = ctx.from?.id != null ? String(ctx.from.id) : null;
    if (userId && deps.onAnyMessage) {
      const handled = await deps.onAnyMessage(userId, ctx);
      if (handled) return;
    }
    await photoHandler(ctx);
  });

  bot.on("message:voice", async (ctx) => { await ctx.reply(stubFor("voice")); await deps.audit("auth_denied", String(ctx.from?.id ?? "?"), { kind: "voice", stub: true }); });
  bot.on("message:audio", async (ctx) => { await ctx.reply(stubFor("audio")); });
  bot.on("message:document", async (ctx) => { await ctx.reply(stubFor("document")); });
  bot.on("message:video", async (ctx) => { await ctx.reply(stubFor("video")); });
  bot.on("message:video_note", async (ctx) => { await ctx.reply(stubFor("video")); });

  let handle: ReturnType<typeof run> | null = null;
  return {
    async start() { handle = run(bot); },
    async stop() { if (handle) await handle.stop(); },
  };
}
```

- [ ] **Step 2: Write `src/index.ts`**

```ts
// src/index.ts
import { existsSync, mkdirSync } from "node:fs";
import { join } from "node:path";
import { getModel } from "@earendil-works/pi-ai";
import { loadConfig } from "./config";
import { acquireLock, LockHeldError } from "./shared/lock";
import { createAudit } from "./shared/audit";
import { createIdentity } from "./agent/identity";
import { createPersistence } from "./agent/persistence";
import { AgentManager } from "./agent/manager";
import { Router } from "./agent/router/router";
import { InputProcessorRegistry } from "./agent/inputs/registry";
import { textProcessor } from "./agent/inputs/processors/text";
import { photoProcessor } from "./agent/inputs/processors/photo";
import { Allowlist, RateLimiter } from "./channels/telegram/security";
import { createTelegramChannel } from "./channels/telegram/bot";
import { PADDELINO_VERSION } from "./shared/types";

async function main() {
  const config = loadConfig(process.env);

  // Ensure home dir exists
  mkdirSync(config.paths.home, { recursive: true });

  let lock;
  try {
    lock = acquireLock(join(config.paths.home, ".lock"));
  } catch (err) {
    if (err instanceof LockHeldError) {
      console.error(err.message);
      process.exit(2);
    }
    throw err;
  }

  process.on("SIGINT", () => { lock.release(); process.exit(0); });
  process.on("SIGTERM", () => { lock.release(); process.exit(0); });

  const audit = createAudit(join(config.paths.home, "audit.log"));
  await audit("restart", "system", { version: PADDELINO_VERSION });

  const identity = createIdentity(config.paths.home);
  const persistence = createPersistence(config.paths.home, PADDELINO_VERSION, "unknown");

  const defaultModel = getModel(config.providers.defaultProvider as any, config.providers.defaultModel as any);

  const agents = new AgentManager({
    home: config.paths.home,
    identity,
    persistence,
    audit,
    defaultModel,
    defaultThinkingLevel: config.providers.defaultThinkingLevel,
  });

  const registry = new InputProcessorRegistry();
  registry.register(textProcessor);
  registry.register(photoProcessor);

  const router = new Router({
    identityName: "default",
    model: defaultModel,
    tools: [],   // populated by AgentManager via getDefaultTools — see manager.ts
    thinkingLevel: config.providers.defaultThinkingLevel,
  });

  const allowlist = new Allowlist(config.telegram.allowedUsers);
  const rateLimit = new RateLimiter({ capacity: config.rateLimit.capacity, refillPerSec: config.rateLimit.refillPerSec });

  // Bootstrap state — wired fully in T35
  const channel = createTelegramChannel({
    token: config.telegram.token,
    allowlist,
    rateLimit,
    audit,
    registry,
    router,
    agents,
    persistence,
    identity,
    config,
    regeneratePersona: async (_d) => { throw new Error("Bootstrap not wired yet (T35)"); },
    beginPersonaDialog: () => { throw new Error("Bootstrap not wired yet (T35)"); },
    onAnyMessage: async (_u, _ctx) => false,
  });

  await channel.start();
  console.log("paddelino started");
}

main().catch((err) => { console.error(err); process.exit(1); });
```

- [ ] **Step 3: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 4: Commit**

```bash
git add src/channels/telegram/bot.ts src/index.ts
git commit -m "Wire grammY bot, handlers, AgentManager into a Channel-backed entrypoint"
```

---

## Phase L — First-run persona bootstrap (T34–T36)

### T34: `bootstrapPersona()` LLM call

**Files:**
- Modify: `src/agent/identity.ts`
- Test: `tests/unit/identity-bootstrap.test.ts`

- [ ] **Step 1: Write failing test using registerFauxProvider**

```ts
// tests/unit/identity-bootstrap.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { registerFauxProvider, getModel } from "@earendil-works/pi-ai";
import { createIdentity, bootstrapPersona } from "../../src/agent/identity";

describe("bootstrapPersona", () => {
  let home: string;
  beforeEach(() => { home = mkdtempSync(join(tmpdir(), "paddelino-boot-")); });

  test("calls the LLM with meta-prompt template and writes result to identity dir", async () => {
    registerFauxProvider({
      provider: "faux",
      id: "test",
      respond: (req: any) => "POLISHED SYSTEM PROMPT",
    });
    const id = createIdentity(home);
    const result = await bootstrapPersona({
      identity: id,
      userInput: "friendly cook helper, brief",
      model: getModel("faux" as any, "test" as any),
    });
    expect(result).toBe("POLISHED SYSTEM PROMPT");
    expect(id.hasPersona()).toBe(true);
    expect(id.getSystemPrompt()).toBe("POLISHED SYSTEM PROMPT");
  });
});
```

- [ ] **Step 2: Add `bootstrapPersona` to `src/agent/identity.ts`**

Append to the file:

```ts
import type { Model } from "@earendil-works/pi-ai";
import { Agent } from "@earendil-works/pi-agent-core";

const META_PROMPT = `You are a prompt engineer. Write a complete, polished system prompt for a household assistant that will run as a Telegram bot.

The user has described the assistant they want as:
"{USER_INPUT}"

The prompt you write MUST:
- Establish the persona (tone, perspective, optional name).
- Define scope and boundaries (what to help with; what to politely decline).
- Set style guidelines (verbosity, formality, response length).
- Note that responses appear in Telegram — keep markdown simple (no tables, no complex nesting).
- Avoid hardcoding the date or other time-sensitive facts (the assistant is told the current date separately).

Output ONLY the system prompt itself, with no preamble, no explanation, and no surrounding quotes.`;

export interface BootstrapArgs {
  identity: Identity;
  userInput: string;
  model: Model<any>;
}

export async function bootstrapPersona(args: BootstrapArgs): Promise<string> {
  const agent = new Agent({
    initialState: {
      systemPrompt: META_PROMPT.replace("{USER_INPUT}", args.userInput),
      model: args.model,
      tools: [],
      thinkingLevel: "off",
      messages: [],
    },
  });

  await agent.prompt("Generate the system prompt now.");

  const last = agent.state.messages.slice().reverse().find((m: any) => m.role === "assistant");
  if (!last) throw new Error("Bootstrap LLM call returned no assistant message");
  const text = typeof last.content === "string"
    ? last.content
    : (last.content as any[]).filter(b => b.type === "text").map(b => b.text).join("").trim();
  if (text.length === 0) throw new Error("Bootstrap LLM returned empty content");

  args.identity.setPersona(args.userInput, text);
  return text;
}
```

- [ ] **Step 3: Tests pass**

```bash
bun test tests/unit/identity-bootstrap.test.ts
```

Expected: PASS. If pi's `registerFauxProvider` API differs from this sketch (e.g., requires a different shape for `respond`), adjust to match the real signature — confirmed during spec phase that the function exists.

- [ ] **Step 4: Commit**

```bash
git add src/agent/identity.ts tests/unit/identity-bootstrap.test.ts
git commit -m "Add bootstrapPersona: one-shot meta-prompt LLM call writes system-prompt.md"
```

---

### T35: Wire bootstrap into `src/index.ts` and `commands.ts`

**Files:**
- Modify: `src/index.ts`

- [ ] **Step 1: Wire bootstrap state and onAnyMessage hook**

In `src/index.ts`, replace the bootstrap-placeholder section with:

```ts
// State map: userId → "awaiting persona input"
const personaDialogUsers = new Set<string>();

const beginPersonaDialog = (userId: string) => { personaDialogUsers.add(userId); };

const regeneratePersona = async (description: string): Promise<string> => {
  const prompt = await bootstrapPersona({ identity, userInput: description, model: defaultModel });
  agents.evictAll();
  return prompt;
};

const onAnyMessage = async (userId: string, ctx: any): Promise<boolean> => {
  const text = ctx.message?.text;

  // (1) First-run: no persona yet → intercept first allowlisted message
  if (!identity.hasPersona() && text) {
    await ctx.reply("Hi! Before we start, describe what kind of assistant you'd like me to be — 1–2 sentences. Tone, focus, anything important. (e.g. 'A friendly home assistant focused on cooking and reminders. Casual tone, brief replies.')");
    personaDialogUsers.add(userId);
    return true;
  }

  // (2) /persona (no args) put user in dialog mode → next message becomes description
  if (personaDialogUsers.has(userId) && text) {
    personaDialogUsers.delete(userId);
    try {
      const prompt = await regeneratePersona(text);
      await ctx.reply(`Got it. Here's the persona I'll use:\n\n${prompt}\n\nIf you'd like to change it, run /persona. Otherwise — how can I help?`);
    } catch (err) {
      await ctx.reply(`Couldn't generate the persona right now — ${(err as Error).message}. Try again, or set ~/.paddelino/identity/system-prompt.md by hand.`);
    }
    return true;
  }

  return false;
};
```

Replace the channel construction's `regeneratePersona`, `beginPersonaDialog`, and `onAnyMessage` placeholders with the values defined above. Add the import: `import { bootstrapPersona } from "./agent/identity";`.

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/index.ts
git commit -m "Wire persona bootstrap dialog and /persona regenerate into entrypoint"
```

---

### T36: Persistence-error handling — surface to user

**Files:**
- Modify: `src/agent/manager.ts`

The current `agent_end` listener swallows save errors into the audit log. Per §8, we also want a user-facing reply. Since the manager doesn't have a reply channel, the cleanest path is to expose a `onSaveError` callback.

- [ ] **Step 1: Add `onSaveError` to `AgentManagerOptions` and call it**

```ts
// in AgentManagerOptions
onSaveError?: (userId: string, err: unknown) => Promise<void>;

// in agent.subscribe callback, replace the catch:
} catch (err) {
  await this.opts.audit("error", userId, { phase: "persistence.save", err: String(err) });
  if (this.opts.onSaveError) await this.opts.onSaveError(userId, err);
}
```

Wire it in `src/index.ts`:

```ts
const agents = new AgentManager({
  // ...
  onSaveError: async (userId, _err) => {
    // Send a one-shot Telegram message via the bot api. We need a handle to the api;
    // expose a global reference, or move this wiring to after channel construction.
    // For SP1: log to audit and skip the user message (the next `agent_end` retries).
  },
});
```

Decision for SP1: log to audit only (as currently). User reply is best-effort and would need a richer Telegram API hook. Document this in a comment and skip the user reply path. Update §8's "Persistence error" row accordingly: **"audit-logged silently; retry on next `agent_end`. User-facing reply deferred."**

- [ ] **Step 2: Typecheck**

```bash
bun run typecheck
```

Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add src/agent/manager.ts src/index.ts
git commit -m "Persistence-save errors audit-logged; user-facing reply deferred"
```

---

## Phase M — Distribution (T37–T39)

### T37: `scripts/install.sh`

**Files:**
- Create: `scripts/install.sh`

- [ ] **Step 1: Write `install.sh`**

```bash
#!/usr/bin/env bash
# scripts/install.sh — paddelino installer
set -euo pipefail

ARCH="$(uname -m)"
case "$ARCH" in
  x86_64|amd64) BUN_TARGET="bun-linux-x64" ;;
  aarch64|arm64) BUN_TARGET="bun-linux-arm64" ;;
  *) echo "Unsupported arch: $ARCH" >&2; exit 1 ;;
esac

INSTALL_DIR="${HOME}/.local/bin"
mkdir -p "$INSTALL_DIR"

# Placeholder URL — replace at release time.
RELEASE_URL="${PADDELINO_RELEASE_URL:-https://github.com/USER/paddelino/releases/latest/download/paddelino-${BUN_TARGET}}"

echo "Downloading paddelino for $BUN_TARGET ..."
curl -fsSL "$RELEASE_URL" -o "$INSTALL_DIR/paddelino"
chmod +x "$INSTALL_DIR/paddelino"

mkdir -p "$HOME/.paddelino"

cat <<EOF

paddelino installed to $INSTALL_DIR/paddelino

Next steps:
  1. Add to PATH if needed:    export PATH="\$HOME/.local/bin:\$PATH"
  2. Create env file:           cp .env.example ~/.paddelino/.env  (then edit)
  3. Run:                       paddelino
  4. Optional service install:  see scripts/systemd/paddelino.service.template

EOF
```

- [ ] **Step 2: Make executable and commit**

```bash
chmod +x scripts/install.sh
git add scripts/install.sh
git commit -m "Add install.sh (arch-detect, downloads binary, prints next steps)"
```

---

### T38: `scripts/systemd/paddelino.service.template`

**Files:**
- Create: `scripts/systemd/paddelino.service.template`

- [ ] **Step 1: Write the template**

```ini
[Unit]
Description=paddelino — household Telegram assistant
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=%h/.local/bin/paddelino
Restart=on-failure
RestartSec=5
EnvironmentFile=%h/.paddelino/.env
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

Document install: `cp scripts/systemd/paddelino.service.template ~/.config/systemd/user/paddelino.service && systemctl --user enable --now paddelino`.

- [ ] **Step 2: Commit**

```bash
git add scripts/systemd/paddelino.service.template
git commit -m "Add systemd user-service template"
```

---

### T39: Flesh out `README.md`

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Write the README**

```markdown
# paddelino

Household Telegram assistant on pi-agent-core. SP1.

## Quick start

```bash
curl -fsSL https://example.invalid/install.sh | sh
cp .env.example ~/.paddelino/.env
# edit ~/.paddelino/.env: set TELEGRAM_BOT_TOKEN, TELEGRAM_ALLOWED_USERS, and one provider key
paddelino
```

On first run, message your bot. It will ask you to describe the assistant in 1–2 sentences and use that to bootstrap its system prompt.

## Commands

- `/start` — greeting
- `/status` — model, provider, message count
- `/reset` — clear conversation
- `/stop` — interrupt in-flight response
- `/retry` — continue without a new user message
- `/model <provider> <id>` — switch model
- `/persona [description]` — regenerate the assistant's persona

## Develop

```bash
bun install
bun run dev
```

## See also

- Design spec: `docs/sp1.md` (in this repo or upstream)
- Powered by: [pi-agent-core](https://github.com/earendil-works/pi)
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "Flesh out README with install/usage/commands"
```

---

## Phase N — Integration tests and smoke (T40–T42)

### T40: Integration test — text round-trip with faux provider

**Files:**
- Create: `tests/integration/text-roundtrip.test.ts`

- [ ] **Step 1: Write the test**

```ts
// tests/integration/text-roundtrip.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { registerFauxProvider, getModel } from "@earendil-works/pi-ai";
import { createIdentity } from "../../src/agent/identity";
import { createPersistence } from "../../src/agent/persistence";
import { createAudit } from "../../src/shared/audit";
import { AgentManager } from "../../src/agent/manager";

describe("integration: text round-trip", () => {
  let home: string;
  beforeEach(() => {
    home = mkdtempSync(join(tmpdir(), "paddelino-int-"));
    registerFauxProvider({ provider: "faux", id: "test", respond: () => "Hello back." });
  });

  test("prompt → assistant message persisted", async () => {
    const identity = createIdentity(home);
    identity.setPersona("desc", "you are friendly");
    const persistence = createPersistence(home, "0.1.0", "pi-test");
    const audit = createAudit(join(home, "audit.log"));
    const agents = new AgentManager({
      home, identity, persistence, audit,
      defaultModel: getModel("faux" as any, "test" as any),
      defaultThinkingLevel: "off",
    });

    const agent = await agents.getAgent("user-1");
    await agent.prompt("Hello?");

    // give agent_end's persist a beat
    await new Promise(r => setTimeout(r, 50));

    const loaded = await persistence.load("user-1") as any;
    expect(loaded).not.toBeNull();
    expect(loaded.messages.length).toBeGreaterThanOrEqual(2);
    const last = loaded.messages[loaded.messages.length - 1];
    expect(last.role).toBe("assistant");
  });

  test("restart preserves conversation", async () => {
    const identity = createIdentity(home); identity.setPersona("d", "p");
    const persistence = createPersistence(home, "0.1.0", "pi-test");
    const audit = createAudit(join(home, "audit.log"));
    const mk = () => new AgentManager({
      home, identity, persistence, audit,
      defaultModel: getModel("faux" as any, "test" as any),
      defaultThinkingLevel: "off",
    });

    let agents = mk();
    let agent = await agents.getAgent("user-1");
    await agent.prompt("first message");
    await new Promise(r => setTimeout(r, 50));

    agents = mk();   // simulate restart
    agent = await agents.getAgent("user-1");
    expect(agent.state.messages.length).toBeGreaterThanOrEqual(2);
  });
});
```

- [ ] **Step 2: Run**

```bash
bun test tests/integration/text-roundtrip.test.ts
```

Expected: PASS (2/2).

- [ ] **Step 3: Commit**

```bash
git add tests/integration/text-roundtrip.test.ts
git commit -m "Integration: text round-trip + restart preserves conversation"
```

---

### T41: Integration test — concurrent message uses steer

**Files:**
- Create: `tests/integration/steer.test.ts`

- [ ] **Step 1: Write the test**

```ts
// tests/integration/steer.test.ts
import { describe, test, expect, beforeEach } from "bun:test";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { registerFauxProvider, getModel } from "@earendil-works/pi-ai";
import { createIdentity } from "../../src/agent/identity";
import { createPersistence } from "../../src/agent/persistence";
import { createAudit } from "../../src/shared/audit";
import { AgentManager } from "../../src/agent/manager";

describe("integration: steer", () => {
  let home: string;
  beforeEach(() => {
    home = mkdtempSync(join(tmpdir(), "paddelino-steer-"));
    registerFauxProvider({
      provider: "faux", id: "slow",
      respond: async () => { await new Promise(r => setTimeout(r, 200)); return "ok"; },
    });
  });

  test("second message during streaming gets steered, not lost", async () => {
    const identity = createIdentity(home); identity.setPersona("d", "p");
    const persistence = createPersistence(home, "0.1.0", "pi-test");
    const audit = createAudit(join(home, "audit.log"));
    const agents = new AgentManager({
      home, identity, persistence, audit,
      defaultModel: getModel("faux" as any, "slow" as any),
      defaultThinkingLevel: "off",
    });

    const agent = await agents.getAgent("user-1");
    const first = agent.prompt({ role: "user", content: "first", timestamp: 1 });
    // Schedule a steer before first completes
    setTimeout(() => agent.steer({ role: "user", content: "ALSO this", timestamp: 2 } as any), 50);
    await first;
    await new Promise(r => setTimeout(r, 250));

    const userMsgs = agent.state.messages.filter((m: any) => m.role === "user");
    expect(userMsgs.some((m: any) => typeof m.content === "string" && m.content.includes("ALSO this"))).toBe(true);
  });
});
```

- [ ] **Step 2: Run**

```bash
bun test tests/integration/steer.test.ts
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add tests/integration/steer.test.ts
git commit -m "Integration: steer queues mid-stream message"
```

---

### T42: Manual smoke checklist

**Files:**
- Create: `docs/smoke-checklist.md`

- [ ] **Step 1: Write the checklist**

```markdown
# paddelino — Manual smoke checklist (SP1)

Run on a clean machine after `install.sh` + provider env vars are set.

- [ ] `paddelino` starts without errors; `~/.paddelino/.lock` is created.
- [ ] Sending any text from an allowlisted Telegram user triggers the bootstrap dialog.
- [ ] Replying with "A friendly home assistant, brief replies." creates `~/.paddelino/identity/system-prompt.md` and the bot echoes the generated prompt.
- [ ] Sending "Hello?" produces a streaming response that reflects the persona.
- [ ] Sending a second text while the first is streaming returns "Added — will fold into the current reply." and the second message appears in the next assistant turn.
- [ ] Sending a photo with caption triggers a vision-aware response (requires a vision-capable model).
- [ ] `/stop` interrupts an in-flight response; `/retry` continues without a new user message.
- [ ] `/reset` clears the conversation; next message starts fresh (but persona is preserved).
- [ ] `/persona A grumpy librarian.` overwrites the prompt and subsequent replies match the new persona.
- [ ] `/model anthropic <typo>` returns "Did you mean ...?".
- [ ] An un-allowlisted user account is silently ignored; an audit line appears in `~/.paddelino/audit.log`.
- [ ] Sending a voice / audio / document / video produces the corresponding plain-text stub.
- [ ] Killing and restarting `paddelino` preserves conversation; first message after restart shows the same persona.
- [ ] Starting a second instance fails fast with a lock error.
```

- [ ] **Step 2: Commit**

```bash
git add docs/smoke-checklist.md
git commit -m "Add manual smoke checklist for SP1 acceptance"
```

---

## End-of-plan checks

After T42, run the full suite:

```bash
cd /mnt/Shared/Code/projects/paddelino
bun run check    # typecheck + bun test
```

All unit + integration tests must pass.

Manual checklist (`docs/smoke-checklist.md`) is the final SP1 acceptance gate.

---

## Spec-coverage self-check (post-plan author note)

| Spec section | Covered by |
|---|---|
| §3 Goals: pi-agent-core runtime end-to-end | T20–T22, T28–T29, T33 |
| §3 Goals: multi-provider, provider-aware default | T32 |
| §3 Goals: per-user conversation continuity | T08, T21, T40 |
| §3 Goals: streaming with consolidated status | T26, T28, T29, T30 |
| §3 Goals: install.sh on Linux | T37, T38 |
| §3 Goals: forward-compatible seams | T14–T19 (Input, Router), T23 (Channel), T20 (before/after hooks) |
| §4.1 Per-user Agent, shared identity | T21 |
| §4.4 Five seams | T18 (Router), T19, T14 (Input), T23 (Channel), T20 (factory hooks), T10 (Identity) |
| §5.3 Module responsibilities | T05–T31 |
| §5.4 First-run persona bootstrap | T34, T35 |
| §6 Lifecycle: text and photo, /commands | T28–T31 |
| §6 Concurrent steer | T28, T29, T41 |
| §6 Polite stubs per type | T27, T33 |
| §6 /model with did-you-mean | T31 |
| §6 /persona dialog and inline forms | T31, T35 |
| §7 Persistence atomic write + envelope | T08 |
| §7 Schema mismatch → archive | T08 |
| §7 Audit log | T06 |
| §8 Error replies (no emojis) | T28, T29, T31 |
| §8 Abort + steer semantics | T31 (/stop), T41 (steer) |
| §9 Build / dist / configuration | T02, T03, T32, T37, T38 |
| §10 Verbatim ports | T07 (lock), T24 (security), T25 (formatting), T26 (streaming pattern) |
| §12 Acceptance criteria | T40, T41, T42 |

**Open in plan:** none.

**Deferred from §8 in T36:** persistence-save error → user-facing Telegram reply. Audit-logged only in SP1 (documented inline).
