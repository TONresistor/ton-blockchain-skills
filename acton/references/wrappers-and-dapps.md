# Wrappers and dApps

Read this reference for generated bindings, precompiled code, ABI changes, or frontend integration. It supplements the command map without prescribing a new application architecture.

Primary guides:

- `https://ton-blockchain.github.io/acton/docs/building/wrappers`
- `https://ton-blockchain.github.io/acton/docs/building/precompiled-boc`
- `https://ton-blockchain.github.io/acton/docs/commands/wrapper`
- `https://ton-blockchain.github.io/acton/docs/dapps`
- `https://ton-blockchain.github.io/acton/docs/acton-toml`

## Contract identity and output settings

Use the contract ID from `[contracts.<id>]`, not its display name or source path. Since 1.2, the PascalCase ID controls generated type names and default filenames. For example, `first_wallet` produces `FirstWallet.gen.tolk` or `FirstWallet.gen.ts`, independent of `display-name`.

Names must start with an ASCII letter and contain only ASCII letters/digits after conversion. Batch generation checks case-insensitive collisions before writing. Regenerating after an upgrade may require updating imports/type names.

Output precedence is CLI options, then per-contract settings, then project defaults. Per-contract fields inherit individually:

```toml
[wrappers.tolk]
output-dir = "wrappers"
generate-test = false
test-output-dir = "tests"

[wrappers.typescript]
output-dir = "wrappers-ts"

[contracts.counter.wrappers.typescript]
output-dir = "app/src/wrappers"
```

When no configured Tolk wrapper directory is present, the `@wrappers` mapping is considered before the default directory. Explicit CLI destinations and manifest destinations have different path-resolution rules; inspect help when running outside the project.

`wrapper --all` conflicts with a contract name, `--output`, and `--test-output`. Directory outputs work with batches. `--test-output` and `--test-output-dir` require `--test`; neither combines with `--ts`.

## ABI exposure

Wrapper generation compiles `.tolk` source directly; a previous `acton build` is unnecessary. The compiler ABI defines the exposed interface:

- `storage` enables typed storage initialization/access.
- `incomingMessages` enables typed internal-message helpers.
- `incomingExternal` enables typed external-message helpers and generic external sends.
- Declared get methods provide getter bindings.

Tolk wrappers can initialize union-alias storage. `fromStorage(storage, toShard, workchain)` defaults to BASECHAIN and preserves selected shard/workchain settings on deployment.

An internal struct is not automatically a public TypeScript serializer. Inspect the emitted ABI and generated exports before replacing manual encoding. Without declared storage/messages, generic untyped helpers may remain available, but that does not establish the schema.

## Precompiled BoC contracts

```toml
[contracts.precompiled]
src = "contracts/Precompiled.boc"
types = "contracts/Precompiled.types.tolk"
```

`src` supplies the deployable code; optional `types` supplies ABI for tooling. Build can use a BoC without an interface, while wrappers need one. `wrapper --all` skips BoCs without `types`; selecting such a contract directly fails with an interface requirement.

The `types` file compiles in interface-only mode; its compiled code is ignored. In 1.2.1, bodyless `get fun` declarations are supported. For earlier versions, use valid stub getter bodies if required. Validate storage layout, opcodes, field types, and getter signatures against the actual BoC: a plausible interface alone does not prove compatibility.

A source contract may depend on a BoC contract. Do not put `depends` on the BoC itself: already compiled code cannot consume newly generated dependency helpers. Dependencies can be names or objects with `name`, `kind = "embed_code"|"library_ref"`, and optional generated `function`/`path` settings.

Build exports distinguish code from sources: `--output-boc` can export compiled or precompiled code; `--output-sources` produces source/debug/ABI registration bundles only for source contracts. Precompiled code plus ABI does not provide source-level debugging.

## TypeScript and React integration

`acton wrapper --all --ts` calls `npx @ton/tolk-abi-to-typescript@0.5.0`. Use the repository's generation script if it adds required checks/corrections. Do not hand-edit generated bindings; change their input or the maintained generation step.

`new --app` and `init --create-dapp` provide React/Vite, TON Connect, and React Query integration. Existing apps can consume bindings without adopting the scaffold. Keep reads/payload construction in the project's SDK/helper layer; verify actual generated APIs before introducing adapters.

For signed cells, a serializer change must preserve the exact cell hash and signature input. Compare existing Tolk/TypeScript vectors, including refs and optional fields, before replacing an encoder. Generating a wrapper does not validate app authentication or signature compatibility.

Regenerate after ABI changes and check generated-file freshness in the project's CI where appropriate. A frontend-only helper change does not justify contract deployment unless the contract code actually changed. Keep private API credentials in a backend; a browser credential is visible to its users.

## RPC and execution evidence

`rpc info` and `rpc call` accept explicit compiler ABI JSON or Tolk interface files through `--abi`; explicit input wins over project/catalog/verifier matching. Use `--block-number` for historical reads, `--json` for machine output, `--raw` for untyped inspection, or `--with-comments` for getter field comments.

`acton doc abi <contract-or-code-hash>` searches local contracts then the bundled catalog, or queries the verifier for a code hash. Inspect the ABI selected before interpreting storage/getter results.

After a broadcast, a successful submit is not proof that the contract accepted or completed execution. For external messages, inspect `waitForFirstTransaction()` / `waitForTrace()`; `isAccepted()` is for known local acceptance and errors when network acceptance is unknown. Confirm transaction outcomes and resulting state before reporting deployment or interaction success.
