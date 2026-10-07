# Tolk Language and Migration

Use this reference for version-sensitive code and upgrades to Tolk 1.5.0. The language version, bundled stdlib, compiler build, and selected toolchain are separate facts; record the actual build's version rather than inferring it from the installed CLI's name.

## Sources of truth

Official source baseline: TON `ed629c416f7a03cd3838697fcee9f8cd0097700b` (Tolk 1.5 merge). Inspect the matching versions of:

- `tolk/tolk-version.h`: compiler version.
- `tolk/type-system.cpp`, `overload-resolution.cpp`, `pack-unpack-serializers.cpp`: aliases, methods, and serializer selection.
- `tolk/pipe-loop-break-continue.cpp`, `loop-control-analysis.cpp`: loop restrictions.
- `tolk/inline-return-analysis.cpp`, `pipe-detect-inline-in-place.cpp`: inlining eligibility.
- `tolk/ast-from-tokens.cpp`, `tolk-tester/tests/preserve-user-calls.tolk`: purity annotations and preserved calls.
- `crypto/smartcont/tolk-stdlib/`: serialization, messages, typed collections, strings, reflection, and low-level APIs.

These paths are in `https://github.com/ton-blockchain/ton`; use the version/revision relevant to the project. Read `https://docs.ton.org/tolk/changelog` for the release overview, then verify non-obvious behavior in source/tests.

## Fixed-size binary data and currency helpers

Built-in `bytesN` was removed. Replace a legacy `bytes64` field with `bits512`, converting bytes to bits rather than retaining the same numeric suffix. `bitsN` is slice-backed and serializes N bits with zero refs; it does not validate arbitrary casts immediately. Packing checks its shape unless explicitly skipped.

A project may declare its own alias named `bytes64`; that is a user type, not the removed primitive. Inspect its definition before renaming it.

Use `grams("0.05")` for nanogram constants; the string must be constant. `ton(...)` is a deprecated alias, not a renamed serialization type. Likewise prefer `reserveGramsOnBalance` over `reserveToncoinsOnBalance`. `coins` remains the variable-length coin encoding; a helper rename does not change it.

## Alias methods and custom serializers

In 1.5, alias receivers are directional. A method on `cell` is available on `type ProofCell = cell`, but a method on `ProofCell` is not automatically callable on an ordinary `cell`. The same distinction applies through alias chains and generic containers.

Aliases retain the underlying runtime representation and ordinary assignment compatibility. They are not opaque constructors or runtime validators. A cast can select an alias-owned method/serializer without proving that the value satisfies its domain rules.

Serializer dispatch follows the receiver type. A plain struct uses its ordinary encoding; an alias can define a distinct encoding:

```tolk
struct Amount {
    value: uint16
}

type TaggedAmount = Amount

fun TaggedAmount.packToBuilder(self, mutate b: builder) {
    b.storeUint(1, 1);
    b.storeUint(self.value, 16);
}

fun TaggedAmount.unpackFromSlice(mutate s: slice): TaggedAmount {
    assert (s.loadUint(1) == 1) throw 9;
    return { value: s.loadUint(16) };
}
```

`Amount { value: 7 }.toCell()` encodes 16 bits, while `(value as TaggedAmount).toCell()` uses the alias serializer. Validate corresponding cell vectors when migrating, including alias smart casts inside `match` and generic overloads.

Do not form `int | IntAlias` when `type IntAlias = int`: equal runtime variants do not become separate tagged alternatives. A union of prefixed structs is a different case with explicit serialized discriminants. `match`/`is` cannot invent an alias downcast from an underlying-type variant.

## Purity and discarded results

In 1.5, explicit reachable expressions preserve their behavior when their result is unused. For example, a slice load or dictionary value decode can still throw; do not classify it as removed validation merely because the value is discarded. Compiler-generated unused work and harmless peephole patterns can still be optimized away.

`@pure` on a regular function is rejected. It remains supported on asm/builtin definitions and is not a promise that the call cannot throw. Pure asm calls can be reordered; exception-order-sensitive code should not assume source order from that annotation.

When a finding depends on optimization or gas, inspect the emitted code and a focused runtime case using the exact compiler. Avoid importing old FunC `impure`/dead-call assumptions into Tolk 1.5.

## Loops

`break` and `continue` work in `while`, `do-while`, and `repeat`, targeting the nearest loop, with compiler-enforced restrictions:

- Neither is allowed within `try/catch` or in loop-condition expressions.
- A loop using `break` cannot also contain a `return` exiting the function, including from a nested loop.
- Structural `continue` requires routing the remaining statements through a single fallthrough path. Complex nested partial branches or a `match` with several surviving branches may be rejected.

Prefer simple top-level guards when skipping items. Advance a map iterator before `continue`, otherwise the loop may revisit the same item forever:

```tolk
fun sumPositive(values: map<uint32, int32>): int {
    var entry = values.findFirst();
    var total = 0;
    while (entry.isFound) {
        val amount = entry.loadValue();
        entry = values.iterateNext(entry);
        if (amount <= 0) { continue; }
        total += amount;
        if (total > 1000) { break; }
    }
    return total;
}
```

## Inlining and early returns

The compiler can inline eligible functions with multiple returns in 1.5. Keep automatic selection unless a measured reason warrants `@inline`, `@inline_ref`, or `@noinline`.

Explicit `@inline` is a requirement, not a hint that silently falls back. Recursive functions, returns inside loops/try-catch, and branching that leaves multiple fallthrough routes can be rejected. A simple guard followed by a final return is supported:

```tolk
@inline
fun positiveOrZero(value: int): int {
    if (value <= 0) { return 0; }
    return value;
}
```

Inlining does not change a getter's ABI or make a helper a public entrypoint.

## Migration validation

- Pin compiler and stdlib together; a `tolk 1.5` source directive is not a package manager or automatic upgrade.
- Remove built-in bytesN uses and regular-function `@pure`; check alias-owned methods, serializers, unions, and smart casts.
- Check rejected `@inline` and loop shapes using the diagnostic's related location, not just its headline.
- Rebuild affected artifacts and compare ABI, serialization/signature vectors, getters, code hashes, and derived deployment addresses relevant to the change. Regenerate client bindings when needed.
- Do not redeploy a contract merely to refresh a skill or a frontend helper. Establish a contract code change and the user's authorization first.
