# Development Environments

Available since Acton 1.2. Use this reference when a task needs a running chain shared by contracts, scripts, wallets, and a dApp. Ordinary `acton test` runs remain local emulator tests; selecting an environment does not redirect them into that chain.

## Choose the execution surface

| Surface | Use for | Limits |
| --- | --- | --- |
| Emulator tests/scripts | Focused contract behavior, assertions, reproducible scenarios, and forked reads | Not a persistent network shared with application clients |
| Simulator | Fast dApp development, state forks, virtual time, manual mining, snapshots, API delays/rate limits | No validators, consensus, elections, or full production indexer stack |
| Docker Localnet | Real validators, elections, node synchronization, full-node APIs, indexed data | Docker/Compose, more resources, real block production; no simulator virtual-time control |
| Studio | Project browser workspace for tests, Explorer, wallets, traces, and managed environments | Starts a local server; managed environments have their own runtime lifecycle |

Primary guides:

- `https://ton-blockchain.github.io/acton/docs/environments/overview`
- `https://ton-blockchain.github.io/acton/docs/environments/how-to/connect`
- `https://ton-blockchain.github.io/acton/docs/environments/simulator/overview`
- `https://ton-blockchain.github.io/acton/docs/environments/localnet/overview`
- `https://ton-blockchain.github.io/acton/docs/studio`

## Simulator

`acton simulator start` runs in the foreground. Startup defaults still live in `[localnet]` in `Acton.toml`, despite the command rename. The default HTTP port is 5411.

Choose only the options needed by the scenario:

- `--fork-net <NET>` and `--fork-block-number <SEQNO>` load remote account state and chain configuration. An explicit block also sets the initial virtual time; otherwise the clock starts from current system time.
- `--accounts <NAME[,NAME...]>` initializes project wallets with 100 GRAM each.
- `--db-path <PATH>` persists state in SQLite. A CLI-relative path uses the current directory; a manifest-relative path uses the project root.
- `--no-mining` disables automatic block production. `--block-time-ms <MS>` sets its target interval, default 500; `--mine-empty-blocks` allows blocks without pending messages.
- `--rate-limit <RPS>` simulates limits on `/api/*`; `--response-delay-ms <MS>` delays TON Center v2/v3 and Emulate responses. Control/UI routes and streaming latency are not covered by that delay setting.
- `--require-auth` requires a token on HTTP API/control/emulate/streaming endpoints. Client commands accept `--auth-token` or `ACTON_LOCALNET_AUTH_TOKEN`; keep the token private.
- `--liteapi` enables the optional LiteAPI (HTTP port + 1 by default); `--liteapi-port` requires `--liteapi`. This surface suits clients that do not require full proof verification.
- `--snapshots-dir <PATH>` selects persistent JSON snapshots; defaults depend on the database or project directory.

Useful control commands (pass the same `--port` and auth settings when needed):

```bash
acton simulator status --json
acton simulator airdrop <ADDRESS> --amount 100
acton simulator mine 1
acton simulator increase-time 3600
acton simulator set-time <UNIX_TIMESTAMP>
acton simulator set-next-block-timestamp <UNIX_TIMESTAMP>
acton simulator snapshot create before-change
acton simulator snapshot list
acton simulator snapshot restore <SNAPSHOT_ID>
```

Time cannot move below the latest mined block. Empty manual blocks are skipped unless empty-block mining is enabled. A snapshot captures accounts, history, metadata, pending messages, and virtual time; restoration also updates SQLite when configured. Import saves a snapshot without applying it; restore is a separate state change. Export/import use project-relative paths and returned IDs, not display names:

```bash
acton simulator snapshot export <SNAPSHOT_ID> --out snapshots/baseline.json
acton simulator snapshot import snapshots/baseline.json --name baseline
```

Use `snapshot delete <ID>` to remove a snapshot. Stop the foreground process with Ctrl-C when finished; database/snapshot persistence is separate from the process lifecycle.

For account/configuration edits or changing API conditions at runtime, use the documented Simulator control API or Studio rather than inventing CLI subcommands:
`https://ton-blockchain.github.io/acton/docs/environments/simulator/control-api`.

## Docker Localnet

Check Docker Engine and Compose v2 availability before starting a network. Names are scoped to `--state-dir`, defaulting to `<project>/.acton-localnet`. Readiness and endpoints come from the actual network record:

```bash
acton localnet start dev --detach
acton localnet status dev --json
acton localnet logs dev --tail 100
acton localnet stop dev
```

- `start` creates or resumes a network; `create` saves a stopped definition. `--port-base`, `--block-time-ms`, `--election-time-seconds`, `--accounts`, and `--accounts-file` are creation inputs, not edits to an existing network.
- `--detach` leaves a newly launched service/network running after readiness. `--json` changes output, not lifecycle or asynchronous behavior.
- `list` and `status` can inspect saved state without starting the network. Stored endpoint URLs do not imply the services are live.
- `stop` preserves blockchain and snapshots. `delete` removes containers and managed blockchain/snapshot data; do not use deletion to fix an endpoint mismatch.
- Additional nodes use `acton localnet node dev add worker [--validator]`. Validation membership changes through elections; enabling it does not prove the node is already elected. Removing a validator normally requires leaving both elected sets; `--force` can disrupt consensus.
- `acton localnet snapshot dev create <NAME>` makes a cold snapshot and may stop/restart nodes. `snapshot dev restore <ID> --yes` replaces chain state/topology and rebuilds indexed data. These differ from Simulator's in-process JSON snapshots.
- `acton localnet operation <ID> --network dev [--wait]` inspects or waits for an accepted operation. Use the result/status rather than treating acceptance of an operation as completion.
- `acton localnet shutdown dev` shuts down its network/control service. Keep `--state-dir` consistent between lifecycle calls.

For administrative account/config changes and detailed node/snapshot actions, inspect `acton help localnet` and the environment guides. Some administrative workflows use the control API; its token is distinct from public-chain API keys.

## Connect scripts and applications

`--net localnet` is a selector, not a guarantee of Docker validators. It can target the Simulator or a Docker network through `[networks.localnet]`. Read `localnet status` or the environment's connection examples and configure the actual URLs:

```toml
[networks.localnet]
api.v2 = "http://127.0.0.1:5411/api/v2"
api.v3 = "http://127.0.0.1:5411/api/v3"
```

These example URLs are for the default Simulator only. A Docker network using port-base 19000 has v2/v3 on 19002/19003; use the printed endpoints rather than hard-coding these ports. Named connections can use `[networks.dev]` and `--net custom:dev`.

After checking endpoints and funding the intended wallet, `acton script scripts/deploy.tolk --net localnet` submits to that local chain. Configure the dApp's RPC client and TON Connect signing for the same environment. Wallet connection alone does not configure RPC reads/submission.

## Studio

`acton studio --no-open` starts the project server without opening a browser, default `127.0.0.1:3015`; `--host` accepts loopback addresses only. `--port` selects another port. Respect session restrictions on starting services and UI.

Studio manages environments and stores workspace history/data under `.studio/`. A separately CLI-started environment is not automatically added to its environment list. Use Studio's generated project/RPC connection settings when observing requests through its proxy; proxy URLs survive environment renames/restarts. Selecting another environment in the browser does not redirect a running client.

CLI tests report to a running Studio for the same project. `--no-studio-reporting` or `[test].studio-reporting = false` disables this; reporting failures do not change test success. Stopping the UI process and stopping/deleting a managed chain are distinct lifecycle operations.
