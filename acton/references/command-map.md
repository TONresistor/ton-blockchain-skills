# Command Map

Use this file for fast command selection before opening full docs or `acton help`. Examples target 1.2.1; verify syntax with the installed binary, especially features marked 1.2.1. Brackets and `|` indicate alternatives, not literal shell syntax.

`NET` means `testnet`, `mainnet`, `localnet`, or `custom:<name>` where supported. `--net localnet` selects configured endpoints and can target a Simulator or Docker Localnet; it does not start either. Common global flags are `--color auto|always|never` and mutually exclusive `--project-root PATH` / `--manifest-path PATH`.

## Setup and project context

- `curl -LsSf https://github.com/ton-blockchain/acton/releases/latest/download/acton-installer.sh | sh`
- Use when Acton is missing and the task requires the CLI. On Windows, run inside WSL.

- `acton --version`
- Use to record the executable version separately from any source checkout.

- `acton doctor`
- Use for resolved project-root, manifest, stdlib, overlays, env-var, and network diagnostics.

- `acton --project-root <PATH> ...`
- Use when config-relative defaults must resolve from a specific project root.

- `acton --manifest-path <PATH>/Acton.toml ...`
- Use to select a manifest without changing project-root resolution.

- `acton up [VERSION] [--list|--trunk|--stable] [--force]`
- Use for version management; a positional version conflicts with list/trunk/stable. Regenerate affected wrappers after upgrading.

## Bootstrap and config

- `acton new <PATH> [--template empty|counter|jetton|nft|w5-extension] [--app] [--hooks[=pre-push|pre-commit]] [--agents] [--overwrite]`
- Use for fresh projects. All built-in templates support dApps. Overwrite replaces colliding paths; hooks default to pre-push when enabled.

- `acton init`
- Use to create a missing manifest, discover contracts, patch `.gitignore`, refresh `.acton`, and link global overlays. Existing manifests are not rewritten.

- `acton init --create-dapp [PATH]`
- Use for the app scaffold only (default: `app`); target must not exist.

- `acton init --stdlib-only`
- Use to re-extract bundled stdlib without reading or changing the manifest.

## Build and wrapper generation

- `acton build [CONTRACT_NAME] [--clear-cache] [--graph PATH] [--out-dir DIR] [--gen-dir DIR] [--output-abi DIR] [--output-boc DIR] [--output-fift DIR] [--output-sources DIR] [--info]`
- Use for project builds, dependency helpers, and artifact exports. Source bundles are for `.tolk` contracts, not precompiled BoCs.

- `acton compile <PATH> [--json|--base64-only] [--boc FILE] [--fift FILE] [--source-map FILE] [--abi FILE] [--allow-no-entrypoint] [--clear-cache]`
- Use for single-file compilation. In 1.2.1, JSON failures can contain `errors[]` diagnostics or a fatal `error` string.

- `acton wrapper <CONTRACT_NAME> [-o PATH|--output-dir DIR]`
- Use for Tolk wrappers directly from source or a precompiled contract's `types` interface.

- `acton wrapper <CONTRACT_NAME> --test [--test-output PATH|--test-output-dir DIR]`
- Use for a wrapper and test stub. Destination flags require `--test`.

- `acton wrapper <CONTRACT_NAME> --ts [-o PATH|--output-dir DIR]`
- Use for TypeScript wrappers via `@ton/tolk-abi-to-typescript@0.5.0`; conflicts with test-stub generation.

- `acton wrapper --all [--ts|--test] [--output-dir DIR]`
- Use for batch generation. Do not pass a contract name or single-file output. BoCs without `types` are skipped. See [wrappers and dApps](wrappers-and-dapps.md).

## Tests and quality

- `acton test [PATHS...] [--filter REGEX] [--include GLOB] [--exclude GLOB] [--fail-fast] [--fuzz-seed SEED] [--reporter console|dot|teamcity|junit]`
- Use for emulator tests; comma-separated reporters and `--junit-path DIR` / `--junit-merge` support CI.

- `acton test --coverage --coverage-format lcov|text [--coverage-file PATH] [--coverage-minimum-percent PERCENT] [--coverage-include-wrappers] [--coverage-include-tests]`
- Use for coverage export and gates.

- `acton test --snapshot build/gas-baseline.json`
- Use to create a gas/fee baseline.

- `acton test --baseline-snapshot build/gas-baseline.json [--fail-on-diff]`
- Use to compare or enforce gas regressions.

- `acton test --gas-profile build/gas.cpuprofile [--gas-profile-format cpuprofile|collapsed] [--gas-profile-include-tests]`
- Use to locate source-level gas hot paths; combine with `--ui` to inspect flamegraphs.

- `acton test --mutate --mutate-contract <CONTRACT_NAME> [--mutation-diff worktree|ref|branch] [--mutation-diff-ref REF] [--mutation-levels critical,major,minor]`
- Use for mutation testing scoped to a contract, diff, or rule levels. Other selectors include `--mutation-disable-rules`, `--mutation-rules-file`, and `--mutation-minimum-percent`.

- `acton test --mutate --mutate-contract <CONTRACT_NAME> --mutation-session-id <ID> [--mutation-id ID[,ID...]] [--mutation-workers N]`
- Use the same session ID and filters to resume; use IDs to rerun mutants from a report. In 1.2.1, `--mutation-timeout SECONDS` limits each mutant (default: 60); timeouts cause final failure independently of score.

- `acton test --debug --debug-port 12345 [--backtrace full]`
- Use for source-level debugging. `--verbose` shows executor logs; `--no-capture` streams output and conflicts with mutation mode.

- `acton test --ui [--ui-port 12344]`
- Use for browser inspection of results, traces, coverage, profiles, and mutation reports. It starts a server and opens a browser.

- `acton test --save-test-trace [DIR]`
- Use for offline trace bundles; default directory is `build/traces`.

- `acton test --fork-net <NET> [--fork-block-number N] [--no-fork-cache]`
- Use remote chain state in local execution. Pin the block and set explicit test time for reproducibility; this does not broadcast to the environment.

- `acton test --no-studio-reporting`
- Use to disable delivery of test reports to a running Studio instance.

- `acton check [TARGET] [--fix] [--output-format plain|json|sarif|github|gitlab] [--output-file PATH] [--enable-only CODE[,CODE...]] [--explain RULE]`
- Use for linting and annotations. Fix requires plain output; output-file requires a non-plain format.

- `acton fmt [PATHS...] [--check]`
- Use for source formatting or CI validation.

- `acton fmt --stdin [--stdin-filepath PATH] [--range startLine:startChar-endLine:endChar]`
- Use for editor input and range formatting; positions are zero-based with UTF-8 byte columns.

- `acton hooks new [--hook pre-push|pre-commit] [--template empty|default]`
- Use to scaffold working-tree checks. `acton hooks install|status|uninstall` manages the local Git override.

## Environments (since 1.2)

- `acton simulator start [--port PORT] [--fork-net NET] [--fork-block-number N] [--accounts NAME[,NAME...]] [--db-path PATH]`
- Use for a lightweight local chain. `[localnet]` retains its simulator defaults. See [environments](environments.md) for API conditions, auth, mining, and persistence options.

- `acton simulator status [--port PORT] [--json]`
- Use to inspect the selected simulator; commands for funding/time/mining/snapshots are detailed in [environments](environments.md).

- `acton localnet start <NAME> [--detach] [--port-base PORT] [--block-time-ms MS] [--election-time-seconds N] [--accounts NAME[,NAME...]] [--accounts-file PATH]`
- Use for a real Docker validator network. `create` saves its definition without starting containers; creation flags apply to new networks.

- `acton localnet list` / `acton localnet status <NAME> [--json]`
- Use for definitions and runtime endpoints. `--state-dir PATH` selects the catalog; stopped endpoints are not proof of a running service.

- `acton localnet stop <NAME>` / `acton localnet delete <NAME>`
- Stop preserves blockchain and snapshots; delete removes managed state. Inspect node/snapshot/operation workflows in [environments](environments.md).

- `acton studio [--host LOOPBACK_IP] [--port 3015] [--no-open]`
- Use for the project browser workspace. `--no-open` still starts a server.

## Scripts, deployment, and blockchain interaction

- `acton script <PATH> [ARGS...]`
- Use for local execution/emulation; there is no `acton deploy` command.

- `acton script <PATH> --net <NET> [--explorer actonscan|tonscan|toncx|dton|tonviewer]`
- Use for transaction submission to configured endpoints; public networks can spend real funds.

- `acton script <PATH> --fork-net <NET> [--fork-block-number N] [--no-fork-cache]`
- Use remote state without broadcasting. With `--net`, fork reads default to the same network; an explicit mismatch is rejected.

- `acton script <PATH> --net testnet|mainnet --tonconnect`
- Use native wallet approval rather than stored signing keys. It requires a supported public broadcast network.

- `acton run <SCRIPT_NAME> [ARGS...]`
- Use manifest `[scripts]` shortcuts; `--` separates forwarded arguments when needed.

## Wallets, verification, libraries, and RPC

- `acton wallet new [--name NAME] [--version VERSION] [--local|--global] [--secure=true] [--airdrop] [--faucet-url URL] [--no-wait-airdrop] [--json]`
- Use to create a wallet. Explicit secure storage fails if unavailable; omitted secure mode can fall back to plaintext.

- `acton wallet import [--name NAME] [--version VERSION] [--local|--global] [--secure=true] [--mnemonic-scheme ton|bip39|rotation] [--json]`
- Use to import an existing mnemonic through prompts or positional words; prefer interactive input to avoid retaining secrets in shell history. Scheme selection and `tg-wallet` are new in 1.2.1.

- `acton wallet list [--balance] [--json]` / `acton wallet export-mnemonic [NAME]`
- Use for inventory or interactive mnemonic export; local wallet names take precedence over global names.

- `acton wallet sign [NAME] [--body HEX_OR_BASE64] [--json]`
- Use for external-body signing (stdin is supported; `--message` is an alias). It does not submit the result. In 1.2.1 JSON omits `input`.

- `acton wallet remove [NAME] [--yes] [--json]` / `acton wallet airdrop [NAME] [--net testnet|localnet] [--faucet-url URL] [--no-wait-airdrop] [--json]`
- Use for config removal or faucet funding. Removal requires confirmation or `--yes` in non-interactive mode; faucet-url applies only to testnet.

- `acton verify [CONTRACT_NAME] [--address ADDRESS] [--wallet NAME|--tonconnect] [--compiler-version VER] [--dry-run]`
- Use for ticket-based Tolk source publication with testnet payment. No `--net` flag; dry-run does not pay or upload.

- `acton verify [CONTRACT_NAME] --payment-tx-hash <HASH> [--address ADDRESS]`
- Use to reuse a finalized payment for that code hash. Conflicts with wallet/tonconnect/dry-run; already verified code needs no new payment.

- `acton library publish [CONTRACT_NAME] [--code HEX_OR_BASE64] [--net NET] [--duration DURATION|--amount GRAM] [--wallet NAME|--tonconnect] [--local|--global] [--yes]`
- Use to publish code and optionally track metadata. TON Connect supports mainnet/testnet; amount overrides duration estimation.

- `acton library fetch <HASH> [--net NET] [--disasm] [-o PATH] [--json]` / `acton library info [NAME]`
- Use for code or tracked-library inspection. Disasm prints text even with json requested.

- `acton library topup [NAME] [--duration DURATION|--amount GRAM] [--wallet NAME|--tonconnect] [--yes]`
- Use to extend storage funding on the library's tracked network; this command does not take `--net`.

- `acton rpc info <ADDRESS> [--net NET] [--abi PATH] [--block-number N] [--json] [--raw]`
- Use for account/storage inspection; raw skips domain inspectors. Explicit ABI JSON/Tolk overrides automatic matching.

- `acton rpc call <ADDRESS> <METHOD> [ARGS...] [--net NET] [--abi PATH] [--block-number N] [--json] [--raw|--with-comments]`
- Use for typed getter calls; raw prints the TON Center stack. The network defaults to testnet.

- `acton rpc block|block-number [--net NET]` / `acton rpc trace <HASH> [--net NET] [--summary|--tree|--verbose] [--show-bodies]`
- Use for chain liveness/block information or decoded transaction trees.

## Inspection and developer tooling

- `acton disasm [BOC_FILE] [-s HEX_OR_BASE64] [-o PATH] [--json] [--address ADDRESS] [--net NET] [--source-map FILE] [--show-offsets] [--show-hashes] [--follow-libraries]`
- Use for BoC or live-code disassembly. File, `--string` (`-s`), and address are mutually exclusive inputs.

- `acton retrace <TX_HASH> [--net NET] [--verbose] [--logs-dir DIR] [--contract CONTRACT] [--debug] [--debug-port PORT]`
- Use to replay an on-chain trace locally. Source-level debug requires a project contract.

- `acton doc abi <CONTRACT_OR_CODE_HASH>`
- Use for local/bundled ABI or verifier lookup by code hash.

- `acton doc tvm <QUERY...> [--find] [--description] [--json]`
- Use for TVM instruction lookup; description requires find.

- `acton ls [--stdio|--port PORT] [--log-file PATH] [--log-level LEVEL] [--no-log] [--stdlib-path PATH] [--profile]`
- Use for native language-server tooling; stdio is the default, TCP serves one client, and profile enables `ton/profile` diagnostics.

- `acton func2tolk <PATH> [--output PATH] [--warnings-as-comments] [--no-camel-case] [--version VERSION]`
- Use for npm-based FunC conversion; it does not establish behavioral equivalence.

- `COMPLETE=<shell> acton` / `acton completions bash|elvish|fish|powershell|zsh|nushell`
- Prefer dynamic completions where supported for project contract/script names; static completions write to stdout and should be regenerated after upgrades.
