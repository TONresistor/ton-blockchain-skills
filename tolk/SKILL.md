---
name: tolk
description: "Write, review, debug, and test Tolk smart contracts for TON. Use for .tolk code, typed storage/messages/getters, serialization, message flows, and language or compiler migrations."
---

# Tolk

Target: Tolk 1.5.0, reviewed on 2026-10-07 against the official compiler, stdlib, and regression tests at TON revision `ed629c416f7a03cd3838697fcee9f8cd0097700b`. Check the project's actual compiler before applying 1.5-specific syntax or behavior.

## Operating Rules

- Preserve the repository's framework, layout, compiler pin, and validation matrix. Updating a source checkout does not update the compiler used by Acton, Blueprint, or another build tool.
- Use official TON Docs for concepts and standards: start from `https://docs.ton.org/llms.txt` and load relevant Tolk or contract pages. For version-sensitive semantics, compare the exact compiler/stdlib source and tests at `https://github.com/ton-blockchain/ton/tree/master/tolk` and `https://github.com/ton-blockchain/ton/tree/master/tolk-tester/tests`.
- Hosted docs may lag a compiler release. In particular, Tolk 1.5 removes built-in `bytesN` and deprecates `ton(...)`; prefer `bitsN` and `grams(...)`.
- Keep public binary layouts explicit: opcodes, field order, fixed widths, refs, optional markers, getter stack shapes, send modes, and bounce behavior. A compiler upgrade can change code hashes and deployment addresses even when the source schema is unchanged.
- Prefer typed structs, cells, maps, union dispatch, and `createMessage`. Keep raw cells/slices/dicts and assembler in justified boundaries.
- Read only relevant references:
  - [Idiomatic patterns](references/idiomatic-patterns.md) for contract structure, ABI, serialization, and message handling.
  - [Language and migration](references/language-and-migration.md) for 1.5 alias, purity, loop, inlining, and compiler changes.
  - [Development checklist](references/development-checklist.md) for tooling, focused validation, and debugging.

## Workflow

1. Identify the contract surface.
   - Establish roles, inbound/outbound messages, getters, storage, deployment, fee assumptions, and asynchronous outcomes.
   - For standard interfaces, check the applicable standard before choosing opcodes or return shapes.

2. Design schemas and public ABI.
   - Follow existing file organization; focused `storage.tolk`, `messages.tolk`, and `errors.tolk` are useful when the project needs them.
   - Use a `contract Name { ... }` directive to expose storage and message families to ABI tooling. Declared getters are exported; helpers do not become ABI serializers automatically.
   - Model uninitialized/deployed storage variants and payload inline/ref layout explicitly.

3. Implement typed entrypoints.
   - `fun onInternalMessage(in: InMessage)` handles non-bounced internal messages.
   - Use `lazy AllowedMessage.fromSlice(in.body)` and `match` for opcode families; decide deliberately whether to ignore empty top-ups and reject unknown messages.
   - `fun onBouncedMessage(in: InMessageBounced)` handles bounce recovery. Choose the parser for the outgoing bounce mode and authenticate/correlate recovery with pending state.
   - `fun onExternalMessage(inMsg: slice)` handles external requests. Validate shape, signature, expiration, and replay state before `acceptExternalMessage()` when rejection should not charge contract gas. Import `@stdlib/gas-payments` for that primitive.
   - `InMessage`/`InMessageBounced` are compiler-managed inputs: access fields directly; do not pass/copy the whole input or capture it in a lambda.

4. Implement storage and outgoing actions.
   - Use named load/save helpers around `contract.getData()` and `contract.setData(...)`.
   - Use lazy reads for partial access, but validate fields explicitly when correctness requires full decoding. Lazy parsing is not a guarantee of complete payload validation.
   - Compose typed messages with `createMessage` and choose value, bounce mode, reserve policy, and send flags for the actual flow.
   - Share `StateInit`/`AutoDeployAddress` construction between address calculation and child deployment.
   - Treat sends as asynchronous actions: submission, execution, bounces, and application settlement are distinct outcomes.

5. Validate the behavior touched by the change.
   - Use the project's existing build/test commands, with focused cases for authorization, layout, getters, sends, bounces, fees, or replay invariants that changed.
   - For compiler or serializer migrations, compare relevant cell hashes/ABI/addresses and behavioral vectors before updating consumers or deploying.

## Preferred Patterns

- Opcode-bearing messages use `struct (0x...) Name { ... }`; preserve exact standard opcode widths and values.
- Known message families use union types and `match`. Generic library unions need not be message opcode unions.
- Use `Cell<T>` for typed references, `cell` for opaque ones, and `RemainingBitsAndRefs` for a true trailing slice.
- Serialized numeric fields use `intN`, `uintN`, `coins`, or other schema-defined encodings. `bitsN` holds exactly N bits with no refs; it is not an integer.
- Use `address` for required internal addresses and `address?` for internal-or-none. Use `any_address` only when broader encodings are intended; an unchecked `as address` cast does not validate an untrusted address.
- `map<K,V>.get()` returns a result with `isFound`, not a nullable value. Check it before `loadValue()`; keys must be fixed-width and values serializable.
- Use arrays, tensors, typed tuples, strings, and optional values according to their actual runtime and serialization representation. Do not confuse tuple-backed containers with dictionaries or cell refs.
- Getter reply structs improve multi-value names; preserve standard getter names, method IDs, stack order, and nested tuple shape.
- In 1.5, alias-specific methods/serializers do not leak to their underlying type. An alias or cast provides no runtime domain validation; perform it explicitly.
- Prefer compiler auto-inlining. `@inline`, `break`, and `continue` have structural limits; consult [language and migration](references/language-and-migration.md) when control flow is rejected.
- Do not use `@pure` on regular functions in 1.5. Explicit calls whose result is unused still preserve validation/throw behavior; `@pure` is not a no-throw guarantee.

## Primary Docs

- `https://docs.ton.org/tolk/overview`
- `https://docs.ton.org/tolk/changelog`
- `https://docs.ton.org/tolk/idioms-conventions`
- `https://docs.ton.org/tolk/features/message-handling`
- `https://docs.ton.org/tolk/features/contract-abi`
- `https://docs.ton.org/tolk/features/auto-serialization`
- `https://docs.ton.org/tolk/features/lazy-loading`
- `https://docs.ton.org/tolk/features/message-sending`
- `https://docs.ton.org/tolk/features/contract-storage`
- `https://docs.ton.org/tolk/features/contract-getters`
- `https://docs.ton.org/tolk/types/maps`
- `https://docs.ton.org/tolk/examples`

## Completion Gate

Report the result with the actual compiler/toolchain version, relevant validation, and any remaining uncertainty:

- Binary layouts and standard interfaces are preserved or intentionally migrated.
- Authorization, replay, fee, bounce, and deployment behavior relevant to the task are explicit.
- Raw serialization/assembler boundaries are justified; typed representations cover the rest where appropriate.
- Required checks passed, or the exact blocker and untested boundary are stated.
- A local build/emulation does not imply deployment or authorize a network transaction.
