# Agent Instructions

Guidance for coding agents (Claude Code, Cursor, Codex, Gemini CLI, WARP, etc.) working in this repository. Human contributors should also find this useful as an architecture primer.

## Project

`@ejwood79/homebridge-nest` is a community fork of `chrisjshull/homebridge-nest`. It's a Homebridge platform plugin that exposes Nest Thermostats (+ Temperature Sensors), Nest Protect, Nest x Yale Lock, and Home/Away state to HomeKit. It talks to Nest's unofficial APIs directly (no Nest developer account); Google-account users authenticate via the `issueToken` + `cookies` method described in `README.md`.

CommonJS, no build step, no test suite. Targets Node ≥7.0.0 and Homebridge 1.x or 2.x.

## Commands

```bash
npm run lint                 # only checked-in script — eslint flat config
npm link                     # symlink for local Homebridge testing
npm install -g .             # install built copy globally for testing
```

There is no automated test suite. Verifying changes means running the plugin under a local Homebridge against a real Nest account and watching the logs (enable `"options": ["Debug.Verbose"]` for protocol-level traces).

ESLint rules (flat config, `eslint.config.js`): 4-space indent, single quotes, semicolons required, unix linebreaks, `caughtErrors: 'none'` so `catch (e) {}` is allowed. Match this style when editing.

## Architecture

### Plugin bootstrap and the factory pattern (non-obvious)

`index.js` exports a function that Homebridge calls with its `homebridge` object. That function packs Homebridge's HAP types into `exportedTypes` and **passes them to `lib/nest-device-accessory.js` first**, which mixes its prototype into a Homebridge `Accessory`. Each device-specific module (`nest-thermostat-accessory.js`, etc.) is a factory: `require('./lib/nest-thermostat-accessory')()` — it must be called *after* the base module has been initialized.

Concretely, you cannot `require` a device accessory file in isolation; it depends on global-ish HAP types injected via the base module. New device types must follow the same factory shape and be wired into both `index.js`'s `require` block and the `loadDevices(...)` calls inside `didFinishLaunching`.

### Two parallel data paths share one accessory model

`lib/nest-connection.js` (~1900 lines) drives **both** APIs:

- **`subscribe()`** — legacy JSON REST long-poll, used for older account types and devices.
- **`observe()`** — newer HTTP/2 protobuf stream, used for newer Nest hardware. Devices delivered through this path are tagged `using_protobuf: true`.

Both paths converge through `apiResponseToObjectTree(...)` which produces a uniform `{ devices: { thermostat: {...}, topaz: {...}, kryptonite: {...}, ... }, structures: {...} }` tree. Accessories are dispatched by `deviceGroup` (e.g. `topaz` = Protect, `kryptonite` = Temp Sensor). When adding a feature, check whether you need to handle it in the JSON branch, the protobuf branch, or both — they often diverge in property naming.

### Tunable timing lives at the top of `nest-connection.js`

The first ~80 lines of `lib/nest-connection.js` are named constants for every retry/backoff/debounce window (`API_PUSH_DEBOUNCE_SECONDS`, `API_MERGE_PENDING_MAX_SECONDS`, `API_MODE_CHANGE_DELAY_SECONDS`, etc.). Pending HomeKit writes are merged with incoming Nest state for `API_MERGE_PENDING_MAX_SECONDS` so user changes don't get clobbered before Nest acknowledges them. Edit the constants instead of hard-coding values inline; the names are referenced throughout the file.

### Feature options are checked everywhere via `optionSet`

`NestPlatform.optionSet(key, serialNumber, deviceId)` (in `index.js`) checks `config.options` for either a bare string (`"Thermostat.Disable"`) or a per-device variant (`"Thermostat.Disable.<serial>"` or `.<deviceId>"`). Accessories call this through `this.platform.optionSet(...)` to gate UI features. New feature toggles should follow the same `Domain.Feature.Action` naming and be documented in both `README.md` and `config.schema.json`.

### Protobuf definitions

`lib/protobuf/` contains `.proto` sources (`root.proto`, `ObserveTraits.protobuf`, plus the `nest/`, `nestlabs/`, and `weave/` trees) loaded at runtime by `protobufjs`. `protobuf.util.Long = null` is set explicitly because of an int64 translation bug — don't remove it. Adding support for a new protobuf trait usually means dropping a `.proto` file into the right subdirectory and decoding it inside `nest-connection.js`'s observe pipeline.

### Auth shapes

Three auth shapes are accepted in `config.json`, validated in `setupConnection()`:

1. `googleAuth: { issueToken, cookies }` — current Google flow.
2. `access_token` — legacy Nest-account token from `home.nest.com/session`.
3. `refreshToken` — historical Google flow, **broken since October 2022**; still parsed but documented as non-functional.

`email`/`password` is also still parsed for legacy reasons. Keep all three shapes working unless explicitly removing one.

## Release artifacts

The repo currently contains a checked-in tarball (`homebridge-nest-4.6.10.tgz`) — that's the published release for reference, not something to edit. Bump `package.json` version through `npm version` and let `preversion` (which runs `lint`) run.
