# wavelink-cli — Architecture & Codebase Guide

> Audience: a developer comfortable in **.NET / C#** with working knowledge of
> **TypeScript** and **Node.js**. This document maps the repository's concepts
> onto familiar .NET terminology where useful, and walks through how the code is
> organized, built, and executed.

---

## 1. What this project is

`@raphiiko/wavelink-cli` is a small **command-line interface (CLI)** for
controlling [Elgato Wave Link](https://www.elgato.com/wave-link) — the audio
mixer software that ships with Elgato's Wave microphones (and, as of 3.0, with
just about any audio hardware).

The CLI lets you script the mixer from a terminal: list devices/mixes/channels,
assign outputs to mixes, change volumes, mute/unmute, and toggle gain on
inputs.

Conceptually, think of the project as:

| Layer                          | Analogy in .NET land                                |
| ------------------------------ | --------------------------------------------------- |
| `wavelink-cli` (this repo)     | A `dotnet tool` (e.g. `dotnet ef`) — a thin CLI    |
| `@raphiiko/wavelink-ts` (dep.) | The underlying SDK / client library                 |
| Wave Link app (Elgato)         | The running service this CLI talks to (over WS)     |

This repo does **not** speak directly to Wave Link's WebSocket / JSON-RPC API.
It depends on a separate npm package — [`@raphiiko/wavelink-ts`](https://www.npmjs.com/package/@raphiiko/wavelink-ts)
— which exports a `WaveLinkClient` class. This CLI is essentially a Commander.js
front-end around that client.

---

## 2. About "Wave Link 3.0 Beta"

> You asked whether Wave Link 3.0 is real or whether this is just a beta-access
> thing. Quick clarification:

Wave Link **3.0 is a real major version**, but it was still in **public beta**
when this CLI's most recent release (`0.0.7`, January 2026) was cut.

- Elgato launched the **Wave Link 3.0 Public Beta on October 16, 2025**. It is
  open to anyone — no special early-access invite needed; you opt in via the
  Elgato installer / beta channel.
- 3.0 was a redesign with a new routing table, up to five output mixes,
  VST3/AU plugin hosting, and — notably — it dropped the requirement to own
  Elgato hardware.
- The CLI's `README.md` and `CHANGELOG.md` specifically pin compatibility to
  **"Wave Link 3.0 Beta Update 4"** (see `CHANGELOG.md` line 11). The upstream
  `@raphiiko/wavelink-ts` client targets the 3.0 protocol, which is different
  from the 2.x WebSocket API.

So: if you only have Wave Link 2.x installed, this CLI won't work — the
underlying client speaks the new protocol. You'd need to install the 3.0 Beta
build from Elgato. (At time of writing, 3.0 has since gone GA, so the README
note about "things might break with future updates" is the standard caveat.)

---

## 3. Tech stack at a glance

| Concern         | Choice                                             | .NET analogue                         |
| --------------- | -------------------------------------------------- | ------------------------------------- |
| Language        | TypeScript 5 (strict mode), ES modules             | C# with nullable + warnings-as-errors |
| Runtime target  | Node.js 18+ **or** Bun 1.0+                        | .NET runtime                          |
| Package manager | Bun (with `bun.lock`)                              | NuGet                                 |
| CLI framework   | [Commander.js](https://github.com/tj/commander.js) | `System.CommandLine`                  |
| SDK             | `@raphiiko/wavelink-ts` ^1.3.0                     | A NuGet client library                |
| Bundler         | `bun build` (see `scripts/build.js`)               | `dotnet publish` to a single file     |
| Lint / format   | ESLint 9 (flat config) + Prettier                  | `dotnet format` + analyzers           |
| Type checking   | `tsc --noEmit`                                     | `dotnet build` (compile-only)         |

### Notable TypeScript / tooling details

- `package.json` declares `"type": "module"` — everything is ESM. Imports in
  `.ts` files use `.js` extensions (e.g. `./commands/info.js`) because that's
  what the emitted bundle will resolve. This is normal for ESM-first TS
  projects; it's not a bug.
- `tsconfig.json` enables `strict`, `noUncheckedIndexedAccess`, and
  `noImplicitOverride`. `noEmit: true` — Bun does the actual bundling.
- The bundler step in `scripts/build.js` inlines the package version into the
  binary by `--define`-ing `process.env.PKG_VERSION`. The runtime then reads
  it in `src/index.ts`.

---

## 4. Repository layout

```
wavelink-cli/
├── package.json          # npm metadata, scripts, deps
├── bun.lock              # Bun's lockfile (analogue: packages.lock.json)
├── tsconfig.json         # TS compiler options
├── eslint.config.js      # ESLint flat config
├── .prettierrc           # Prettier config
├── CHANGELOG.md          # Keep-a-Changelog format
├── README.md             # User-facing usage docs
├── scripts/
│   └── build.js          # Bun-based bundler entry point
└── src/
    ├── index.ts          # CLI entry point — wires up commander
    ├── commands/         # One file per top-level command group
    │   ├── info.ts
    │   ├── output.ts
    │   ├── mix.ts
    │   ├── channel.ts
    │   └── input.ts
    ├── services/         # Thin wrappers around the wavelink-ts SDK
    │   ├── client.ts     # `withClient` connection helper
    │   └── finders.ts    # Resolve "id-or-name" -> entity, with "require*" guards
    ├── types/
    │   └── index.ts      # Local DTOs (MixInfo, OutputInfo, InputInfo, ChannelInfo)
    └── utils/
        ├── error.ts      # `exitWithError(msg): never`
        ├── format.ts     # Display helpers (percent, muted, channel name)
        └── validation.ts # `parsePercent` for 0–100 inputs
```

This is a deliberately flat, layered structure. Roughly:

- **`commands/`** = the *presentation* layer (Commander wiring + console output).
- **`services/`** = the *application* layer (talking to the SDK, resolving identities).
- **`types/`**, **`utils/`** = cross-cutting helpers.

There's no DI container, no test project, no Result-type wrapper — for a CLI
this small (~10 files), the indirection isn't worth it.

---

## 5. How a command actually runs

Let's trace what happens when you run:

```bash
wavelink-cli output set-volume "Headphones (Arctis Nova Pro Wireless)" 75
```

### 5.1 Entry point: `src/index.ts`

The shebang `#!/usr/bin/env node` makes the bundled `dist/index.js` directly
executable when installed as a global npm bin. The file is tiny: it creates a
single `Command` (Commander's root program), calls each `register*Commands(program)`
function to attach subcommands, and finally `await program.parseAsync()`.

The top-level `await` works because the file is ESM and Node 18+ supports
top-level await in modules.

### 5.2 Command registration: `src/commands/output.ts`

`registerOutputCommands` creates a `program.command("output")` sub-command and
attaches verbs (`list`, `assign`, `set-volume`, `mute`, `toggle-mute`, …) using
Commander's fluent API. Each `.action(...)` callback:

1. Parses/validates raw string arguments (e.g. `parsePercent(volume, "Volume")`
   returns a `number` or exits the process with an error message).
2. Calls `withClient(client => doTheThing(client, ...))`.

### 5.3 Connection lifecycle: `src/services/client.ts`

`withClient<T>(action)` is the *only* place that constructs a `WaveLinkClient`
and manages its connection. In C# terms it's a "using-style" higher-order
function:

```ts
const client = new WaveLinkClient({ autoReconnect: false });
try {
  await client.connect();
  return await action(client);
} catch (err) { /* log and process.exit(1) */ }
finally { client.disconnect(); }
```

This pattern ensures every command:

- always connects before running,
- always disconnects (success or failure) — so the Node process exits cleanly,
- produces uniform error output and exit codes.

If you've used `Func<TClient, Task<T>>`-style helpers in C# for `HttpClient` or
database connections, this is exactly that.

### 5.4 Entity resolution: `src/services/finders.ts`

Wave Link identifies devices/outputs/channels/mixes by long opaque IDs (look at
the README examples — `{0.0.0.00000000}.{abc12345-...}`). End users would
rather type a name. So `finders.ts` provides:

- `findMixByIdOrName`, `findChannelByIdOrName`, `findInputByIdOrName`,
  `findOutputByIdOrName` — case-insensitive lookup by ID first, then by name.
  Return `null` when nothing matches.
- `requireMix`, `requireOutput`, `requireInput`, `requireChannel` — same as
  above but `exitWithError(...)` when the entity is missing.

The returned shapes are the local DTOs from `src/types/index.ts` — *not* the
raw SDK shapes. This decouples the CLI from upstream library changes and keeps
the rest of the code working with a minimal, stable surface.

### 5.5 Doing the work: back in `output.ts`

```ts
export async function setOutputVolume(client, outputId, volumePercent) {
  const output = await requireOutput(client, outputId);
  await client.setOutputVolume(output.deviceId, output.outputId, volumePercent / 100);
  console.log(`Successfully set output '${output.outputName}' volume to ${volumePercent}%`);
}
```

Note the unit conversion: the CLI accepts `0–100` (percent), but the SDK
expects `0.0–1.0`. The `utils/format.ts::formatPercent` does the inverse for
display.

### 5.6 Output

All output is plain `console.log` — no logger abstraction, no JSON output mode,
no `--quiet` flag. Errors go to `console.error` and the process exits with
code `1`. That's the entire I/O contract.

---

## 6. Domain model (Wave Link concepts)

Useful to know if you're new to Wave Link:

- **Channels** — the *sources* of audio: System, Music, Browser, Game,
  per-application channels, etc. Each channel has a master level/mute plus a
  per-mix level/mute (so the same channel can be loud on your stream mix and
  quiet in your headphones).
- **Mixes** — virtual *busses*: typically `Stream Mix`, `Monitor Mix`, and (in
  3.0) optionally `Personal Mix` and others. Each mix mixes all channels into
  a single stereo signal.
- **Output devices** — *physical* outputs (headphones, speakers). Each output
  is assigned to exactly zero or one mix at a time.
- **Input devices** — hardware microphones / line-ins, each with gain and a
  mute state.

The CLI exposes one command group per concept (`channel`, `mix`, `output`,
`input`), plus a top-level `info` command for the connected Wave Link app
metadata.

The "set X as the only output for mix Y" operation (`mix set-output` /
`output assign`) is interesting: there's no single SDK call for it, so
`setSingleOutputForMix` in `output.ts` iterates all outputs and issues
`switchOutputMix` / `removeOutputFromMix` calls as needed. It then reports
exactly what changed.

---

## 7. Developer workflow

All scripts live in `package.json` and assume Bun is on `PATH` (Node also works
for `start`/`format`/`lint`/`typecheck` if you swap `bun run` for `npx`).

| Command              | What it does                                                     |
| -------------------- | ---------------------------------------------------------------- |
| `bun run start`      | Run the CLI from source (`bun run src/index.ts`)                 |
| `bun run build`      | Bundle `src/index.ts` → `dist/index.js` via `scripts/build.js`   |
| `bun run format`     | Prettier write                                                    |
| `bun run format:check` | Prettier check (CI-style)                                       |
| `bun run lint`       | ESLint over `src/**/*.ts`                                         |
| `bun run lint:fix`   | ESLint with `--fix`                                               |
| `bun run typecheck`  | `tsc --noEmit` — pure type check, no output                       |
| `bun run check`      | `format:check && lint && typecheck` — the "is it green?" command  |
| `prepublishOnly`     | npm lifecycle: runs `build && check` before publishing            |

There are **no tests** in the repo today. Given the CLI is a thin shim over an
external WebSocket-based SDK that needs a running Wave Link app, that's a
defensible choice — most useful tests would need Wave Link installed or a
fairly involved mock of `WaveLinkClient`.

### Suggested mental model for working on it

1. Want to add or change a command? Edit the appropriate file in `src/commands/`.
2. Need a new way to look up an entity? Add to `src/services/finders.ts`.
3. Need a new validator / formatter? Add to `src/utils/`.
4. Need to expose a new SDK capability? Likely just call the new method on
   `WaveLinkClient` from a command — the SDK already wraps the protocol.

---

## 8. Things a .NET developer might find surprising

- **No DI container, no interfaces around the client.** `withClient` is just a
  function. For a CLI this size, perfectly fine.
- **`process.exit(1)` is idiomatic** for fatal errors here, instead of throwing
  to the top-level. Note `exitWithError` returns `never`, which is TS's
  equivalent of `[DoesNotReturn]`.
- **`.js` extensions on TS imports** — required by Node's ESM resolver.
- **Top-level `await`** in `src/index.ts` — legal in ESM modules; no
  `async Task<int> Main` ceremony.
- **Bun is both runtime and bundler** for this repo. You can substitute Node
  for runtime, but the `build` script explicitly calls `bun build`. Reproducing
  the build without Bun would require swapping in esbuild or tsup.
- **Strings everywhere for IDs.** There's no branded-type / value-object
  pattern — `string` is `string`. The `finders` layer is how the codebase keeps
  ID handling consistent.

---

## 9. TL;DR

`wavelink-cli` is a ~10-file TypeScript CLI built on Commander.js that thinly
wraps the `@raphiiko/wavelink-ts` SDK to control Elgato's Wave Link 3.0 mixer
(which is a real version — public beta opened in October 2025, and the CLI
specifically targets "3.0 Beta Update 4"). Code is split into
`commands/` (CLI wiring + presentation), `services/` (connection lifecycle and
id-or-name lookups), and small `utils/` + `types/` modules. Bun is used as the
runtime and bundler; ESLint, Prettier, and `tsc --noEmit` cover quality
checks. There are no tests, no DI, and no clever abstractions — and given the
project's scope, that's appropriate.
