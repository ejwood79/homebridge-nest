# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Source of truth

The authoritative guidance for working here lives in [`AGENTS.md`](AGENTS.md) — read it first. It is written for coding agents and covers the architecture in depth. This file is a quick reference; keep the two in sync and prefer updating `AGENTS.md` when adding detail.

## What this is

`@ejwood79/homebridge-nest` — a Homebridge platform plugin (community fork of `chrisjshull/homebridge-nest`) exposing Nest Thermostats (+ Temperature Sensors), Nest Protect, Nest x Yale Lock, and Home/Away state to HomeKit via Nest's unofficial APIs. CommonJS, no build step, no automated tests. Targets Node ≥7.0.0 and Homebridge 1.x or 2.x.

## Commands

```bash
npm run lint      # only checked-in script (eslint flat config); also runs on preversion/prepublishOnly
npm link          # symlink for local Homebridge testing
npm install -g .  # install a copy globally for testing
npm version <x>   # bump version; preversion hook runs lint
```

There is no test suite. Verify changes by running the plugin under a local Homebridge against a real Nest account and reading the logs — add `"options": ["Debug.Verbose"]` to config for protocol-level traces.

Lint style (`eslint.config.js`, `eslint:recommended` + overrides): 4-space indent, single quotes, semicolons required, unix linebreaks, `caughtErrors: 'none'` (bare `catch (e) {}` allowed). Match it when editing.

## Architecture essentials (see AGENTS.md for full detail)

- **Factory bootstrap.** `index.js` is the Homebridge entry point; it injects HAP types into `lib/nest-device-accessory.js` (the base), which every device module (`lib/nest-thermostat-accessory.js`, etc.) mixes into via a factory call `require('./lib/nest-*-accessory')()`. Device files cannot be `require`d in isolation — they depend on types injected through the base. New device types must follow this factory shape and be wired into `index.js`'s require block and the `loadDevices(...)` calls in `didFinishLaunching`.
- **Two data paths, one model.** `lib/nest-connection.js` (~1900 lines) drives both `subscribe()` (legacy JSON REST long-poll) and `observe()` (newer HTTP/2 protobuf stream, devices tagged `using_protobuf: true`). Both converge through `apiResponseToObjectTree(...)` into a uniform device/structure tree, dispatched by `deviceGroup` (`topaz` = Protect, `kryptonite` = Temp Sensor, etc.). Features often need handling in both branches, which diverge in property naming.
- **Timing constants** live in the first ~80 lines of `lib/nest-connection.js` (retry/backoff/debounce/merge windows). Edit the named constants rather than inlining values; pending HomeKit writes are merged with incoming Nest state so user changes aren't clobbered before Nest acknowledges them.
- **Feature toggles** are gated via `NestPlatform.optionSet(key, serialNumber, deviceId)` in `index.js`, matching either a bare `"Domain.Feature.Action"` string or a per-device `.<serial>`/`.<deviceId>` suffix. New toggles follow the same naming and must be documented in `README.md` and `config.schema.json`.
- **Protobuf** sources are under `lib/protobuf/` (loaded at runtime by `protobufjs`). `protobuf.util.Long = null` is set deliberately for an int64 bug — do not remove it.
- **Auth shapes** accepted in `config.json` and validated in `setupConnection()`: `googleAuth: { issueToken, cookies }` (current), `access_token` (legacy Nest), and `refreshToken` (broken since Oct 2022 but still parsed). Keep all working unless explicitly removing one.
