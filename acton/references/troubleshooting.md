# Troubleshooting

Use a focused diagnostic for the observed failure. Check the installed version before applying 1.2.1-specific guidance.

## Acton is missing or the wrong version runs

- Check `command -v acton` and `acton --version`; a Git checkout update does not replace the installed executable.
- If the CLI is needed and missing, use the public installer from [SKILL.md](../SKILL.md), then open a fresh shell or reload the updated profile.
- On Windows, install/run inside WSL. Git hooks and editor tooling must use the same WSL environment; a WSL terminal alone does not move the editor's Git process into WSL.

## Docs and CLI disagree

- Compare `acton <command> --help` and `acton help <command>` with documentation for that version.
- Follow the installed CLI for execution; update only if the task needs newer behavior and the project supports it.
- Current replacements include `init --create-dapp` for `--create-app` and Simulator for the old lightweight Localnet implementation. Since 1.2, `acton localnet` is the real Docker network, not a removed command.
- Treat `litenode`, `[mappings]`, `--broadcast`, `--api-key`, `up --canary`, and `--tonconnect-port` as stale unless an older installed version supports them.
- Do not add `--net` to `verify`: its ticket/payment CLI uses testnet even though the verifier service supports broader workflows.

## Project root or manifest confusion

- Use `acton doctor` to inspect resolved paths and the manifest.
- `--project-root /abs/project` chooses project-root defaults; `--manifest-path /abs/project/Acton.toml` selects a manifest without changing root resolution. They cannot be combined.
- Config-relative output paths use the project root. Ordinary CLI paths typically remain relative to cwd; simulator snapshot files and mutation rules are examples of explicit project-relative exceptions.
- Check the selected project's `.env`, wallet overlays, import mappings, and generated output locations rather than assuming cwd selected everything.

## Build or import resolution failures

- Confirm each contract's `src` exists. It may be `.tolk` or precompiled `.boc`; a BoC interface is supplied separately by `types`.
- Confirm dependency IDs exist. `depends` accepts names or structured entries with `name`, `kind`, and optional generated function/path overrides. Do not configure dependencies on a precompiled BoC itself.
- Inspect `[import-mappings]` and dependency helpers. `acton init` creates defaults only for a missing manifest; it leaves an existing manifest untouched. Repair missing mappings explicitly rather than expecting init to rewrite them.
- Use `acton build --clear-cache` when there is evidence of a stale cache; otherwise read the actual compiler error first. In 1.2.1 caches include compiler version/commit, settings, and requested artifacts.
- For single-file `compile --json` automation on 1.2.1, handle both `errors[]` source diagnostics and `error` fatal/input/configuration failures. Source positions are one-based lines and UTF-8 byte columns; missing ranges may be null.

## Wrapper generation problems

- Pass a configured contract ID, not a source path. Read the wrapper compilation error; a prior build is unnecessary.
- Check `[contracts.<id>].types` for a BoC contract. A direct request requires it; `--all` skips BoCs without it.
- Confirm exported ABI contains the relevant `storage`, `incomingMessages`, `incomingExternal`, and get-method declarations. Internal structs do not automatically become public serializers.
- Since 1.2, wrapper names/files use the PascalCase contract ID. After upgrades, check imports and case-insensitive batch name collisions.
- `--test-output`/`--test-output-dir` require `--test`; neither can combine with `--ts`. `--all` cannot use a contract name or single-file destinations.
- TypeScript generation needs Node.js/npm/npx for `@ton/tolk-abi-to-typescript@0.5.0`. Check its actual subprocess error rather than editing generated output.
- See [wrappers and dApps](wrappers-and-dapps.md) for per-contract settings, interface compatibility, and signed-cell migration.

## Test behavior differs from scripts or environments

- Re-run the failing case with `acton test --filter "<specific-test>" --backtrace full`.
- Tests execute locally, including forked tests. Scripts submit to an endpoint only with `--net`; a selected Studio environment does not change either command's mode.
- Omitted script `--fork-net` defaults to `--net` when broadcasting; an explicit fork/network mismatch is rejected.
- Local/latest-fork execution uses current time. Historical forks start at the selected block's time; set `testing.setNow(...)` when a fixed time matters.
- In 1.2, `crypto.getFastRandomBytes(bytes)` uses VM random state and no longer accepts a seed. Set `testing.setRandomSeed(seed)` or `random.setSeed(seed)` for repeatability; do not substitute a fast RNG for secure key generation.
- After network submission, external-message `isAccepted()` errors if acceptance is unknown. Use `acceptanceKnown`, `waitForFirstTransaction()`, or `waitForTrace()` instead of reporting success from submission alone.
- Regenerate stale wrappers when ABI changes. In 1.2.1 an `exitCode` predicate in `toHaveFailedTx` still requires failure, not just a matching code.

## Coverage, profiling, fuzzing, or reporter confusion

- Coverage formats are `lcov` and `text`; use `--coverage-file`, `--coverage-minimum-percent`, and optional wrapper/test inclusion. These differ from source-level gas profiles.
- Gas baselines use `--snapshot`, `--baseline-snapshot`, and `--fail-on-diff`. Hot-path analysis uses `--gas-profile` with `cpuprofile` or `collapsed`, optionally including tests and UI flamegraphs.
- Reporters are `console`, `dot`, `teamcity`, and `junit`; combine with commas. JUnit output uses `--junit-path` and `--junit-merge`.
- Reuse `--fuzz-seed` for fuzz inputs, but also pin time/state when the scenario depends on them.
- `--no-capture` streams stdout/stderr while retaining captured reports; it cannot run with mutation mode.

## Mutation failures, timeouts, or interrupted runs

- Resume with the printed `--mutation-session-id` and the same filters. `--mutation-id` selects mutants from a prior report; it is not a new independent rule set.
- Use `--mutation-workers` to control concurrency when resource pressure is the observed cause.
- In 1.2.1 the default timeout is 60 seconds for each mutant's compilation plus tests. Configure `--mutation-timeout` / `[test.mutation].timeout` for a legitimately longer workload.
- Timed-out mutants are outside the mutation score, remaining mutants still run, and final status is failure. A satisfactory percentage alone does not mean the run passed.

## Test UI or Studio problems

- `test --ui` starts a server and opens a browser; `--ui-port` changes its port. If tests fail before UI startup, diagnose without UI first.
- `studio --no-open` starts a server without opening a browser; use `--port` for an occupied port. Only loopback `--host` addresses are accepted.
- Check that CLI test reporting targets the same project as Studio. `--no-studio-reporting` disables reporting; reporting failure does not change test results.
- Use `--save-test-trace [DIR]` for offline traces when serving a UI is outside the task's scope. Respect session restrictions on services/browser use.

## Simulator, Localnet, or client endpoint mismatch

- Identify whether the process is Simulator, standalone Docker Localnet, or Studio-managed. Their state directories, ports, snapshots, and time controls differ.
- Simulator defaults stay in `[localnet]`; inspect `simulator status` with the same port/auth settings. Manual empty blocks require empty-block mining to be enabled.
- For Docker Localnet, inspect `localnet status <NAME> --json` and `localnet logs <NAME>`. Saved endpoint records may exist while the runtime is stopped. Check Docker/Compose readiness only if the failing task actually requires them.
- Read the actual v2/v3 URLs from the selected environment; `--net localnet` can point at either runtime. Changing a Studio tab does not redirect an already configured client.
- Fund the wallet in that chain, and align the dApp's RPC settings and signing environment. TON Connect does not automatically configure the RPC client.
- Use stop/start to preserve data. Restore/delete replaces or removes managed state; diagnose before using those operations. Read [environments](environments.md) for lifecycle and snapshots.

## Wallet, faucet, or signing issues

- Inspect `wallet list` and only request `--balance` when a network read is useful. Local wallet names override global ones.
- Confirm the mnemonic source: `mnemonic-env`, `mnemonic-file`, `mnemonic-keyring`, or `mnemonic`. Do not print the source's contents while diagnosing.
- Explicit `--secure=true` fails when keyring storage is unavailable; omitted secure selection may store plaintext. Use an appropriate CI mnemonic source rather than committing secrets.
- In 1.2.1, select `ton`, `bip39`, or `rotation` explicitly when importing non-TON phrases. Word count alone does not select BIP39; TON is the default.
- Check version, workchain, explicit `wallet-id`, and expected addresses. Custom networks default to global ID `-3`; V5 wallets for custom mainnet endpoints need `global-id = -239` unless an explicit ID intentionally overrides network defaults.
- `wallet sign` reads hex/base64 external-body BoC or stdin, prefers hex when ambiguous, and does not broadcast. In 1.2.1 JSON no longer includes `input`.
- `wallet airdrop --net testnet` uses a PoW faucet; `--net localnet` uses that configured local chain. `--faucet-url` applies only to testnet; `--no-wait-airdrop` skips the testnet balance wait.
- `export-mnemonic` requires an interactive terminal and confirmation. Removal updates config/keyring, not external mnemonic files or environment variables.

## TON Center API keys and rate limits

- Use `TONCENTER_TESTNET_API_KEY`, `TONCENTER_MAINNET_API_KEY`, or `<NORMALIZED_NAME>_API_KEY` for `custom:<name>`. Names are uppercased and non-alphanumeric characters become `_`.
- Acton loads `.env` during project work; inspect which project is selected before changing credentials.
- Simulator authentication uses `ACTON_LOCALNET_AUTH_TOKEN`; Localnet control-service tokens have a separate administrative role. Do not replace one with a public-provider key.
- Missing/rate-limited keys can affect forked tests, scripts, verification, wallet balances, RPC, disassembly, and retrace. Check the relevant endpoint response rather than retrying every command or changing networks.

## Verification mismatch or interrupted payment

- `verify --dry-run` prepares without payment/upload; normal verification may require a testnet payment and publishes source bytes.
- Check the local code hash and any supplied address before a new paid attempt. In 1.2.1 the default remote compiler version matches the bundled compiler; use `--compiler-version` when the deployed build requires another version.
- Already verified code succeeds without another payment and can return before the optional address comparison. Verification success alone is not proof of the state at an address.
- Reuse a finalized matching payment with `--payment-tx-hash` rather than paying again after an interruption. It conflicts with wallet/tonconnect/dry-run; inspect the recorded payment and verifier status before retrying a state-changing attempt.
- There is no `verify --net`; do not infer CLI network options from the verifier service's mainnet support.

## RPC, disasm, or retrace problems

- Confirm the requested network and block. `rpc info`/`rpc call` can use `--abi` with compiler JSON or a Tolk interface instead of automatic matching.
- Getters require ABI-compatible arguments and return types. `rpc call --raw` prints the TON Center stack; `rpc info --raw` skips domain inspection. `--with-comments` conflicts with raw getter output.
- Generate source maps for readable disassembly: `compile contract.tolk --source-map contract.json --boc contract.boc`, then `disasm contract.boc --source-map contract.json --show-offsets`.
- Source-level retrace needs `--contract <ID>` with `--debug`; precompiled code alone does not supply source-level metadata.

## func2tolk, LSP, or completions problems

- FunC conversion requires the npm-based converter. Check Node.js/npm/npx and use `func2tolk --version <VERSION>` when a converter version is the issue.
- LSP can select stdlib through `--stdlib-path`, project `.acton/tolk-stdlib`, other discovered directories, or its bundled fallback. `init --stdlib-only` refreshes the project copy; do not assume that it is the only supported source.
- LSP stdio is the default. TCP serves one client; `--log-level`, `--no-log`, and `--profile` control diagnosis without changing language behavior.
- Prefer `COMPLETE=<shell> acton` where supported for dynamic project names; regenerate static completion scripts after upgrading.

## Missing deploy command confusion

Use `acton script scripts/deploy.tolk` for local emulation. Add `--net <NET>` only when submission to that selected network is intended and authorized. Deployment is script-driven by design.
