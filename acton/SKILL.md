---
name: acton
description: "Develop and inspect TON projects with Acton CLI. Use for Acton commands, manifests, builds, wrappers, tests, scripts, wallets, verification, and local development environments."
---

# Acton TON CLI Workflow

This guidance targets Acton 1.2.1, reviewed against its source and command manuals on 2026-10-07. Features introduced in 1.2.1 are marked below; older installations may not support them.

## Source of truth

- Record `acton --version` and inspect `acton <command> --help` for accepted syntax; `acton help <command>` provides the detailed manual.
- Prefer the installed binary for local execution. Hosted docs and trunk source may describe a newer version; identify the version difference before choosing flags or updating the tool.
- Official documentation:
  - `https://ton-blockchain.github.io/acton/docs/welcome/`
  - `https://ton-blockchain.github.io/acton/docs/commands/overview`
  - `https://ton-blockchain.github.io/acton/llms-full.txt`
- Use `https://github.com/ton-blockchain/acton-contracts` for project and contract examples, not CLI flag definitions. For implementation questions, inspect source matching the relevant Acton version.
- Read only the references relevant to the task:
  - [Command map](references/command-map.md): command selection and useful options.
  - [Environments](references/environments.md): Simulator, Docker Localnet, Studio, client endpoints, and snapshots.
  - [Wrappers and dApps](references/wrappers-and-dapps.md): ABI exposure, precompiled contracts, output settings, and frontend integration.
  - [Troubleshooting](references/troubleshooting.md): diagnosis of command, build, test, wallet, and network failures.

## Install or update Acton

When the task requires Acton and it is missing:

```bash
curl -LsSf https://github.com/ton-blockchain/acton/releases/latest/download/acton-installer.sh | sh
```

Open a fresh shell or reload the updated shell profile, then verify `acton --version`.

- `acton up` installs the latest stable release; `--list` lists versions, a positional version selects one, and `--trunk` selects a trunk build.
- Updating a source checkout does not update the installed executable. Check `command -v acton` when multiple installations are possible.
- Versioned macOS and Linux GNU releases are supported; on Windows, use WSL and run Git and Acton in the same distribution.
- In CI, use the project's pinned version with `ton-blockchain/setup-acton@master` or `ghcr.io/ton-blockchain/acton:<version>`.

## First checks in any project

1. Inspect the project layout, `Acton.toml`, existing scripts, and validation requirements. Use `acton doctor` for resolved paths, stdlib, overlays, and network diagnostics when needed.
2. Select project context explicitly when outside the project:
   - `acton --project-root <PATH> ...`
   - `acton --manifest-path <PATH>/Acton.toml ...`
3. Inspect only relevant manifest sections: `[package]`, `[toolchain]`, `[contracts]`, `[build]`, `[wrappers.*]`, `[test]`, `[fmt]`, `[lint]`, `[networks]`, `[localnet]`, `[scripts]`, and `[import-mappings]`. Wallet and library overlays may also come from separate local/global TOML files.

`--project-root` and `--manifest-path` are mutually exclusive. Config-relative paths use the resolved project root; ordinary CLI input/output paths usually use the current directory. Some options, such as mutation rule files, are explicitly project-relative: inspect their help rather than applying one path rule to every flag. `--manifest-path` selects the manifest without changing project-root resolution.

## Project bootstrap

- `acton new <path> --template empty|counter|jetton|nft|w5-extension` creates a project. Use `.` for the current directory.
- All built-in templates support `--app` for a React/Vite TypeScript dApp. Preserve an existing project's layout when adding Acton.
- `--name`, `--description`, `--license`, and `--agents` configure the scaffold. `--overwrite` replaces colliding files; inspect collisions before using it.
- `--hooks` creates and installs the default pre-push hook. Use `--hooks=pre-commit` to select pre-commit; an explicit hook value requires `=`.
- `acton init` creates a missing manifest, discovers contract sources, patches `.gitignore`, refreshes stdlib, and attempts wallet/library overlay symlinks. An existing `Acton.toml` is left untouched; repair its mappings explicitly.
- `acton init --create-dapp [path]` creates only the app scaffold, defaulting to `app`. It fails if the destination exists.
- `acton init --stdlib-only` re-extracts stdlib even outside an initialized project, without touching the manifest or overlays.

## Build, compile, and wrappers

- `acton build [contract-name]` builds all configured contracts or one plus transitive dependencies. Build settings can export ABI, BoC, Fift, and source bundles.
- `acton compile <file.tolk>` compiles one explicit source, without traversing the dependency graph. Use `--allow-no-entrypoint` for helper files; `--source-map`, `--abi`, `--boc`, and `--fift` export artifacts.
- In 1.2.1, `compile --json` reports source diagnostics in `errors[]`, with optional ranges, function context, and related locations. Input/configuration/fatal failures use `error`; consumers must handle both shapes.
- `acton wrapper <contract-name>` generates Tolk wrappers; `--ts` generates TypeScript through `npx @ton/tolk-abi-to-typescript@0.5.0`. Node.js/npm/npx are required for this mode.
- Use `--test` to generate a test stub. `--test-output` and `--test-output-dir` require `--test`; TypeScript mode conflicts with all test-stub options.
- `acton wrapper --all` processes configured contracts. A prior build is not required: wrapper generation compiles source or a configured ABI interface directly.
- Regenerate after ABI changes or upgrades that affect generated names. Read [wrappers and dApps](references/wrappers-and-dapps.md) before changing serialization, using `.boc` contracts, or integrating generated code into a frontend.

## Tests and quality gates

- `acton test [paths...]` runs emulator tests. `--filter`, `--include`, `--exclude`, and `--fail-fast` narrow execution; `--fuzz-seed` makes fuzz inputs reproducible.
- Local tests/scripts start from current wall-clock time. A fork pinned with `--fork-block-number` uses that block's time; latest forks use current time. Set `testing.setNow(...)` when the test requires a fixed clock.
- `--fork-net <network>` resolves remote accounts while execution stays local. It does not turn tests into transactions on a running Simulator or Localnet. `--no-fork-cache` bypasses the persistent cache for pinned forks.
- Use `--debug`, `--debug-port`, `--backtrace full`, or `--verbose` for diagnosis. `--no-capture` streams captured test output and conflicts with mutation mode.
- Reporters include `console`, `dot`, `teamcity`, and `junit`; combine them with commas. Use `--junit-path` and `--junit-merge` for CI output.
- Coverage: `--coverage`, `--coverage-format lcov|text`, `--coverage-file`, `--coverage-minimum-percent`, and optional inclusion of wrappers/tests.
- Gas regressions: create `--snapshot <file>`, then compare with `--baseline-snapshot <file>` and optionally `--fail-on-diff`.
- Source-level gas profiles: `--gas-profile <file>`, `--gas-profile-format cpuprofile|collapsed`, and `--gas-profile-include-tests`. These identify hot paths rather than enforce a snapshot baseline.
- Mutation: `--mutate --mutate-contract <name>`, optionally scoped by diff, rule levels, or IDs. Preserve the printed session ID and filters when resuming.
- In 1.2.1, `--mutation-timeout <seconds>` / `[test.mutation].timeout` defaults to 60 seconds per mutant, including compilation. Timed-out mutants are excluded from the score; remaining mutants continue and the final exit status is 1.
- In 1.2.1, `toHaveFailedTx` accepts an `exitCode` predicate; the matcher still requires a failed transaction.
- `--ui` serves test results and opens a browser; `--save-test-trace [dir]` saves offline bundles. `--no-studio-reporting` disables reporting to a running Studio instance without changing test execution.
- Defaults come from `[test]`, `[test.coverage]`, `[test.fuzz]`, and `[test.mutation]`; CLI flags override them for that invocation.

Follow the repository's validation matrix. Common CI commands are `acton build`, `acton test --reporter console,junit`, `acton check --output-format github`, and `acton fmt --check`; select the checks relevant to the change.

## Linting, formatting, and hooks

- `acton check [target]` checks the project, a contract ID, or a `.tolk` path. Configure `[lint]`, `[lint.rules]`, and per-contract overrides.
- `--fix` rewrites only safe linter fixes in plain mode. `--output-file` requires a non-plain format; output formats include `json`, `sarif`, `github`, and `gitlab`.
- `--enable-only <codes>` selects rules; `--explain <rule>` describes one. Inline `// check-disable-next-line <rule-name>` uses names, not codes, and cannot suppress compiler/parser diagnostics.
- `acton fmt [paths...]` formats sources; `--check` validates formatting. `--stdin`, `--stdin-filepath`, and `--range` support editor integration; ranges use zero-based positions and UTF-8 byte columns.
- `acton hooks new|install|status|uninstall` manages `.githooks`. The default hook runs `check` and `fmt --check` on the working tree, not an isolated copy of staged or pushed revisions. Preserve existing hook configuration.

## Scripts, deployment, and network reads

- `acton script <path> [args...]` executes a standalone Tolk script. Deployment is script-driven; there is no `acton deploy`.
- Without `--net`, scripts emulate locally. `--fork-net` reads remote state without broadcasting.
- With `--net testnet|mainnet|localnet|custom:<name>`, scripts submit transactions to that network. Omitted `--fork-net` defaults to it for reads; explicit fork and broadcast networks must match.
- Network selection and `--fork-block-number` control the emulation context; they do not prove a transaction was accepted. After broadcasting, inspect `waitForFirstTransaction()` or `waitForTrace()`. `ExternalSendResult.isAccepted()` cannot establish acceptance from submission alone; check `acceptanceKnown`.
- `--tonconnect` uses native QR/deep-link wallet approval for mainnet/testnet broadcasting. It requires `--net`; no browser bridge or `--tonconnect-port` is needed.
- `--explorer` accepts `actonscan` (default), `tonscan`, `toncx`, `dton`, or `tonviewer`.
- `acton run <script-name> [args...]` executes a manifest `[scripts]` entry. Script arguments use `main()` ABI; use `--` before arguments resembling Acton flags.
- Validate changes locally and use testnet before a new mainnet deployment. Match validation to the change and the user's authorized scope; running a local example does not authorize a public-network transaction.
- Acton loads `.env` during project work. Built-in keys are `TONCENTER_TESTNET_API_KEY`, `TONCENTER_MAINNET_API_KEY`, and `<NORMALIZED_NAME>_API_KEY` for custom networks (uppercase, non-alphanumeric characters replaced by `_`).

## Development environments

Available since 1.2:

- Simulator: deterministic local chain execution, public-state forks, virtual time, manual mining, snapshots, and configurable API conditions; no validators or consensus.
- Localnet: real validators and TON Center services in Docker; use for elections, synchronization, and full-node/indexer behavior.
- Studio: browser workspace for tests, traces, contracts, wallets, and environment management. `--no-open` suppresses browser opening, but still starts the server.

Read [environments](references/environments.md) to select endpoints and lifecycle commands. Starting a server, opening a browser, or creating a Docker network must fit the requested task and session permissions.

## Wallets, verification, and inspection

- `acton wallet new|import|list|export-mnemonic|sign|remove|airdrop` manages local/global wallet overlays. Local names override global ones; use keyring storage when available or `mnemonic-env` for CI.
- In 1.2.1, TG Wallet rev00 is supported, with `mnemonic-scheme = "ton"|"bip39"|"rotation"`. Import with `--mnemonic-scheme` or interactive selection; TON remains the default independently of word count. BIP39 accepts 12 or 24 words.
- In 1.2.1, explicit `wallet-id` applies on every network; omitting it keeps version/network defaults. Workchain values must fit i8. For V5 on a custom mainnet endpoint, configure `networks.<name>.global-id = -239` (custom default: `-3`).
- `wallet sign --body <BoC>` signs an external body supplied as hex/base64 or stdin; it does not broadcast. In 1.2.1 its JSON output no longer includes the redundant `input` encoding field; signed bodies remain hex.
- `acton verify [contract-name] [--address <addr>]` compiles and publishes Tolk sources through the ticket-based verifier with a testnet payment. The CLI has no `--net` flag. Already verified code needs no new payment.
- Use `verify --dry-run` to prepare without payment/upload. `--payment-tx-hash` reuses a finalized testnet payment bound to the same code hash; it conflicts with `--wallet`, `--tonconnect`, and `--dry-run`. In 1.2.1 the default compiler version is the bundled Tolk version.
- `acton rpc info`, `rpc call`, `rpc block`, `rpc block-number`, and `rpc trace` inspect network state. ABI can come from the project, catalog, verifier, or explicit JSON/Tolk input. `acton doc abi <contract-or-code-hash>` prints ABI JSON.
- `acton library publish|fetch|info|topup` manages on-chain code libraries and local/global metadata. `publish`/`topup` can spend funds; `fetch` and `info` are inspection paths. Regular accounts cannot directly perform change-library publication on public networks.
- Low-level tools include `disasm`, `retrace`, `doc tvm`, `func2tolk`, and `ls`. Dynamic completions use `COMPLETE=<shell> acton`; `acton completions <shell>` emits a static script.

## Safety and correctness rules

- Before broadcasting, paying for verification, or publishing/topping up a library, establish the wallet, destination, network, and existing user authorization. Stop if those assumptions cannot be verified.
- Never put mnemonics, keyring secrets, or private API/control tokens in shared output or committed files.
- Keep emulation, submission, execution evidence, and source verification distinct when reporting results.
- Avoid stale spellings such as `init --create-app`, `litenode`, `[mappings]`, `--broadcast`, `--api-key`, or `up --canary` unless the installed version supports them.
- Use TON docs for blockchain concepts outside Acton's CLI/tooling scope. Acton 1.2 uses GRAM/nanogram terminology; old TON aliases remain supported. Inspect the relevant stdlib docs when migrating APIs such as random-byte generation.
