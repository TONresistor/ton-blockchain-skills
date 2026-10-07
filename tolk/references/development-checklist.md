# Tolk Development Checklist

Use the relevant parts before finalizing implementation, review, migration, or debugging. Follow the project's validation matrix; this is a checklist of possible checks, not a requirement to run every suite for every edit.

## Docs and version lookup

- Start with `https://docs.ton.org/llms.txt`, then fetch only the relevant Tolk/standard pages.
- Record the actual compiler version and stdlib selected by the build tool. An Acton/Blueprint CLI version is not the Tolk version; a source directive does not install a compiler.
- For 1.5 behavior, inspect [language and migration](language-and-migration.md) and the matching official compiler/tests. Hosted examples may still mention removed `bytesN` or regular-function `@pure`.
- Useful topics: `tolk/features/contract-abi`, `auto-serialization`, `lazy-loading`, `message-handling`, `message-sending`, `contract-getters`, and `tolk/types/maps` under `https://docs.ton.org/`.
- Standard contracts require the applicable public opcodes, layouts, getter names, and stack shapes; an idiomatic-looking example is not proof of standard compatibility.

## Design checklist

- Roles, authorization, signature domains, seqno/expiration, and replay rules are explicit where relevant.
- Storage field order, widths, refs, defaults, and uninitialized variants preserve the intended cell layout.
- Incoming messages have schema-defined prefixes and fields; unknown/empty-message handling is deliberate.
- The `contract` directive describes the public ABI, including storage/messages and any types explicitly exported for clients.
- Outgoing messages specify destinations, value source, bounce modes, send flags, and reserve assumptions.
- Asynchronous success, bounced recovery, retries, and settlement are distinguished from submitting an action.
- Child deployment uses shared StateInit/address construction and intentional shard/workchain selection.
- Getter names, method IDs, field order, and tensor/tuple shape match consumers and standards.

## Implementation checklist

- Follow existing module boundaries; introduce separate schemas/helpers only when they improve the actual project.
- Use named storage load/save helpers and typed messages/maps/cell refs where appropriate.
- Inspect lazy parsing boundaries: partial decoding does not validate skipped fields or automatically assert end.
- Use `UnpackOptions.assertEndAfterReading` correctly; eager whole-value decoding and middle-of-slice loads have different semantics.
- For legacy bounces, recovery fields fit the returned prefix; rich bounces use `RichBounceBody`. Correlate sender/query/pending state before changing balances.
- External message rejection checks precede acceptance when invalid requests should not charge contract gas. Persist/commit replay state according to the intended action-failure behavior.
- Keep manual serialization, low-level dictionaries, exotic cells, and asm within justified boundaries.
- For 1.5 migration, check bytes-to-bits widths, alias-owned serializers/methods, `match` smart casts, purity annotations, and loop/inlining restrictions.

## Tooling

Prefer repository commands and package scripts. Do not introduce a framework or reinstall a compiler merely to answer a source-level question.

For an Acton project, inspect `Acton.toml`, `acton --version`, and the relevant help. Typical commands, selected according to the change:

```bash
acton build
acton test --filter "<relevant-case>"
acton wrapper <CONTRACT_ID> --test
acton compile contracts/Main.tolk --source-map build/Main.map.json --boc build/Main.boc --abi build/Main.abi.json
acton disasm build/Main.boc --source-map build/Main.map.json
```

`disasm` takes a BoC path, inline code, or address; it does not resolve a contract ID. Wrapper generation compiles its own source/interface, so a prior build is not required. In 1.2.1, compile diagnostics may use `errors[]` as well as `error`; record the ABI's `compiler_version` when checking Tolk 1.5.

For Blueprint projects, inspect `blueprint.config.*`, wrapper compilation configuration, package scripts, and the installed compiler package. Use the actual test runner; do not assume Acton flags work with Blueprint or that a package named `tolk-js` has the latest language release.

- `npx blueprint build` is a usual project build entrypoint; confirm its help/version.
- Use existing `npm`, `yarn`, `pnpm`, or `bun` scripts for tests/coverage/gas when configured, rather than inventing test flags.
- Source-only migration examples can be compiled in an isolated directory with the exact compiler. Do not modify the user's global toolchain to do so.

Scripts without `--net` emulate locally in Acton. Network submissions, source-verification payments, and deployments require the user's authorization for that target. Local validation does not grant it.

## Focused testing checklist

Cover the behavior touched by the change, selecting relevant scenarios:

- Deployment, initial storage, uninitialized variants, and deterministic addresses.
- Accepted/unknown/empty messages, malformed or truncated payloads, unexpected refs, authorization, and expected exit codes.
- Getters, exact field/stack order, and client decoding.
- Outgoing destination, opcode, value, inline/ref placement, bounce mode, send/reserve flags, and action failure.
- Bounce repair, sender/pending-query correlation, duplicate recovery, and out-of-order asynchronous responses.
- Signature payload hashes, destination/domain binding, seqno/replay state, and expiration boundaries.
- Cell bit/ref limits, variable payloads, remainder tails, and custom serializers.
- Map insertion/deletion/lookup/iteration boundaries; iterator progress before `continue`.
- Compiler upgrades: alias serializer vectors, discarded validating calls, supported early returns/loop guards, ABI, code hashes, and deployment-address changes.

Use fresh emulator state or an explicit scenario baseline. Pin time/random seed/fork block when determinism matters; local/latest-fork execution may start from wall-clock time. Report emulation measurements as local evidence, not a production gas guarantee.

## Debugging checklist

- Read the compiler's primary and related source locations. For rejected `@inline`/`continue`, simplify the identified branch shape before changing the public API.
- Compare message opcode and actual cell tree, including refs/optional markers, before assuming business logic is wrong.
- Check eager versus lazy decoding, retained validation calls, and alias serializer selection against the exact compiler.
- Inspect emitted ABI/getter stack order and the actual wrapper exports.
- Verify envelope body placement and bounce-parser alignment; a payload beginning with `0xffffffff` is not by itself a bounced message.
- Check fee/reserve/send-mode effects and actual transaction/action outcomes.
- Use disassembly or a focused runtime reproducer when an optimization, exception, or gas claim cannot be established from source alone.

## Run-end report

State what changed or was established, the relevant compiler/toolchain version, checks and results, and any remaining unverified boundary. Keep local compilation, emulator behavior, network execution, and source verification separate.
