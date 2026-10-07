# Idiomatic Tolk Contract Patterns

Use this reference for implementing or reviewing Tolk 1.5 contract code. Examples are independent patterns; adapt their types and state transitions to the actual contract. For version-sensitive changes, read [language and migration](language-and-migration.md).

## Table of Contents

- Project layout
- Contract directive and ABI
- Storage
- Message schemas
- Internal messages
- Bounced messages
- External messages
- Outgoing messages and deployment
- Getters
- Maps
- Serialization boundaries
- Standard contract families

## Project Layout

Follow the existing project layout. For a new multi-file contract, a small set of focused files is usually sufficient:

- `contracts/errors.tolk`: error constants or enums.
- `contracts/messages.tolk`: incoming and outgoing message structs, opcodes, payload aliases, and union types.
- `contracts/storage.tolk`: persistent storage structs and `load/save` helpers.
- `contracts/*-contract.tolk`: entrypoints, getters, validation, state transitions, and sends.
- `tests/*`: integration tests and helper wrappers according to the project's framework.
- `scripts/*`: deployment or operational flows if the project already uses scripts.

Keep shared schemas easy to import. Current Tolk also permits importing another contract’s types while its entrypoints/getters stay local to its defining file; do not assume that importing a declared contract necessarily duplicates reserved entrypoints.

## Contract Directive and ABI

Declare the public shape for tooling in the contract entrypoint file:

```tolk
contract Counter {
    storage: Storage
    incomingMessages: AllowedMessage
}
```

`incomingExternal`, `outgoingMessages`, `emittedEvents`, `thrownErrors`, and `storageAtDeployment` describe other public surfaces when relevant. Declared get methods are exported. Use `forceAbiExport` for additional types needed by tooling, then inspect actual ABI and generated bindings. A directive describes types; it does not implement message dispatch, authorization, storage initialization, or contract invariants. A source file can have only one contract directive.

## Storage

Use persistent storage as a struct and wrap raw data access:

```tolk
struct Storage {
    ownerAddress: address
    seqno: uint32
    balance: coins
}

fun Storage.load() {
    return Storage.fromCell(contract.getData())
}

fun Storage.save(self) {
    contract.setData(self.toCell())
}
```

Use `lazy Storage.load()` when only a subset of fields is needed. Lazy reads load fields on demand and do not prove that every field or trailing bit was validated:

```tolk
get fun currentOwner(): address {
    val storage = lazy Storage.load();
    return storage.ownerAddress;
}
```

For storage with multiple valid shapes, model the shapes explicitly. Start from a loader struct or slice, inspect remaining bits/refs, then parse the exact shape:

```tolk
struct ItemStorage {
    itemIndex: uint64
    collectionAddress: address
    ownerAddress: address
    content: cell
}

struct ItemStorageNotInitialized {
    itemIndex: uint64
    collectionAddress: address
}

struct ItemStorageLoader {
    itemIndex: uint64
    collectionAddress: address
    private rest: RemainingBitsAndRefs
}

fun ItemStorage.startLoading() {
    return ItemStorageLoader.fromCell(contract.getData())
}

fun ItemStorageLoader.isInitialized(self) {
    return !self.rest.isEmpty()
}

fun ItemStorageLoader.endLoading(mutate self): ItemStorage {
    return {
        itemIndex: self.itemIndex,
        collectionAddress: self.collectionAddress,
        ownerAddress: self.rest.loadAny(),
        content: self.rest.loadAny(),
    }
}
```

## Message Schemas

Represent message bodies as structs. Use 32-bit prefixes for opcode-bearing messages:

```tolk
struct (0x12345678) Increment {
    queryId: uint64
    amount: coins
}

struct (0x23456789) Reset {
    queryId: uint64
    value: coins
}

type AllowedMessage = Increment | Reset
```

Use the narrowest correct types:

- `address` for required internal addresses.
- `address?` for optional internal addresses.
- `any_address` or `any_address?` when the contract intentionally accepts broader address encodings.
- `Cell<T>` when the payload is a typed ref.
- `cell` when a ref is intentionally opaque.
- `RemainingBitsAndRefs` when the schema includes "the rest of the slice".
- Fixed-width `intN`/`uintN` or `coins` inside serialized structs, not unbounded `int`.

Use custom serializers for domain encodings that do not fit built-in types. In 1.5, an alias-owned serializer is selected for that alias, not for every value of the underlying type:

```tolk
type MetadataTail = slice

fun MetadataTail.unpackFromSlice(mutate s: slice): MetadataTail {
    val rest = s;
    s = createEmptySlice();
    return rest
}

fun MetadataTail.packToBuilder(self, mutate b: builder) {
    b.storeSlice(self)
}
```

## Internal Messages

Use `InMessage`, lazy union parsing, and `match`:

```tolk
const ERR_NOT_OWNER = 401

fun onInternalMessage(in: InMessage) {
    val msg = lazy AllowedMessage.fromSlice(in.body);

    match (msg) {
        Increment => {
            var storage = lazy Storage.load();
            assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_OWNER;
            storage.balance += msg.amount;
            storage.save();
        }
        Reset => {
            var storage = lazy Storage.load();
            assert (in.senderAddress == storage.ownerAddress) throw ERR_NOT_OWNER;
            storage.balance = msg.value;
            storage.save();
        }
        else => {
            assert (in.body.isEmpty()) throw 0xFFFF
        }
    }
}
```

Choose the unknown-message policy deliberately. Empty internal messages are commonly balance top-ups; non-empty unknown messages are usually rejected.

## Bounced Messages

Use `onBouncedMessage` for bounce-specific recovery. A body prefix alone does not turn an ordinary message into a bounce. Match the parser to the original outgoing bounce mode.

For `BounceMode.Only256BitsOfBody`, only a prefix of the original body is available, with no original refs. Put recovery identifiers/amounts early enough to fit; do not try to deserialize an entire original message containing addresses or refs.

```tolk
struct (0x34567890) TransferOut {
    queryId: uint64
    amount: coins
}

fun onBouncedMessage(in: InMessageBounced) {
    in.bouncedBody.skipBouncedPrefix();
    val msg = lazy TransferOut.fromSlice(in.bouncedBody);
    val queryId = msg.queryId;
    val amount = msg.amount;
    // Correlate queryId and in.senderAddress with pending state.
    // Apply the matching state repair once, not a blind credit from amount.
}
```

This example only parses the recoverable fields; the comments stand for application-specific state repair. For rich bounces, parse the rich wrapper instead of skipping the legacy prefix:

```tolk
fun originalRichBody(bouncedBody: slice): cell {
    val rich = lazy RichBounceBody.fromSlice(bouncedBody);
    return rich.originalBody;
}
```

`RichBounceBody` has prefix `0xfffffffe`, original body/info, phase, exit code, and optional compute details. `RichBounceOnlyRootCell` omits the original body's refs. Full `RichBounce` retains richer recovery data but costs more. `NoBounce` gives no bounce-based recovery. If several bounce families are possible, dispatch explicitly rather than applying one parser to all of them.

## External Messages

Validate the signed schema, destination/domain, expiration, signature, and replay state before accepting external gas when invalid requests should be rejected without charging contract gas. The signature must cover the exact intended payload, including refs.

```tolk
import "@stdlib/gas-payments"

struct ExternalStorage {
    publicKey: uint256
    domain: uint32
    seqno: uint32
}

struct SignedPayload {
    destination: address
    domain: uint32
    validUntil: uint32
    seqno: uint32
}

struct SignedRequest {
    signature: bits512
    payload: SignedPayload
}

fun onExternalMessage(inMsg: slice) {
    val request = SignedRequest.fromSlice(inMsg);
    var storage = ExternalStorage.fromCell(contract.getData());
    assert (request.payload.destination == contract.getAddress()) throw 100;
    assert (request.payload.domain == storage.domain) throw 101;
    assert (request.payload.validUntil > blockchain.now()) throw 102;
    assert (request.payload.seqno == storage.seqno) throw 103;

    var signedBody = inMsg;
    signedBody.skipBits(512);
    assert (isSignatureValid(signedBody.hash(), request.signature as slice, storage.publicKey)) throw 104;
    acceptExternalMessage();

    storage.seqno += 1;
    contract.setData(storage.toCell());
}
```

Here the signer hashes the payload cell, excluding the leading signature. This is one example schema, not the wire format of a standard wallet. Select domain values according to the protocol; never refactor a deployed signed format without hash/signature vectors.

For flows with outbound actions, specify when replay state is persisted or committed and what should survive action failure. Do not add `commit()` mechanically: wallet versions and application recovery rules can differ.

## Outgoing Messages And Deployment

Prefer `createMessage({ ... })` and pass typed bodies directly:

```tolk
val reply = createMessage({
    bounce: BounceMode.NoBounce,
    dest: in.senderAddress,
    value: 0,
    body: Reset {
        queryId: msg.queryId,
        value: 0,
    }
});
reply.send(SEND_MODE_CARRY_ALL_REMAINING_MESSAGE_VALUE);
```

Passing a typed body lets `createMessage` choose its envelope inline/ref representation. Passing a cell explicitly selects a referenced whole body. This differs from an inner payload field declared `Cell<T>`; preserve the receiver’s schema:

```tolk
// Typed whole body; automatic envelope placement.
body: SomeMessage { queryId }

// Explicit referenced whole body, when needed.
body: SomeMessage { queryId }.toCell()
```

Extract deploy address/state construction into a helper:

```tolk
fun calcDeployedChild(ownerAddress: address, childCode: cell): AutoDeployAddress {
    val initialStorage: ChildStorage = {
        ownerAddress,
        value: 0,
    };

    return {
        stateInit: {
            code: childCode,
            data: initialStorage.toCell(),
        }
    }
}
```

For shard-sensitive deployments, use `toShard` with a documented reason:

```tolk
dest: {
    stateInit: { code, data },
    toShard: {
        closeTo: ownerAddress,
        fixedPrefixLength: 8,
    }
}
```

Use `UnsafeBodyNoRef` only when direct inline-body encoding is required and the full message is known to fit. Choose send/reserve modes deliberately: fees, remaining inbound value, and remaining contract balance are different sources. Neither a bounce flag nor a successful compute phase proves all asynchronous actions succeeded.

## Getters

Prefer explicit return types. Use standard getter names when a standard defines them; otherwise use camelCase.

```tolk
struct ContractDataReply {
    ownerAddress: address
    balance: coins
    code: cell
}

get fun contractData(): ContractDataReply {
    val storage = lazy Storage.load();
    return {
        ownerAddress: storage.ownerAddress,
        balance: storage.balance,
        code: contract.getCode(),
    }
}
```

Getter reply structs make wrappers and off-chain clients easier to audit. Verify the emitted TVM stack shape: `(A,B)` tensors and `[A,B]` tuples are not interchangeable. Naming fields does not authorize changing a standard getter’s method ID or stack order.

## Maps

Use typed maps instead of low-level dictionary operations in core logic. Keys must have fixed-width serializable encodings; values must be serializable, with refs modeled explicitly:

```tolk
type ReplayMap = map<uint64, bool>
const ERR_REPLAY = 402

struct Storage {
    processed: ReplayMap = []
}

fun Storage.markProcessed(mutate self, queryId: uint64) {
    assert (!self.processed.exists(queryId)) throw ERR_REPLAY;
    self.processed.set(queryId, true);
}
```

Map lookup returns a result object:

```tolk
val r = storage.records.get(key);
if (r.isFound) {
    val record = r.loadValue();
}
```

Do not check map lookup with `== null`. Use `isFound`, `mustGet`, `exists`, `isEmpty`, and iteration helpers such as `findFirst` and `iterateNext`.

When a low-level dictionary enters from a boundary API, convert once:

```tolk
val records = createMapFromLowLevelDict<uint256, Cell<Record>>(rawDict);
```

## Serialization Boundaries

Use auto-serialization first. Reach for manual builders/slices only for:

- custom TL-B shapes not expressible with built-in types;
- validation of a raw external payload;
- precise inline/ref control;
- opaque pass-through data;
- low-level blockchain configuration or proof structures.

Common checks:

- A cell has at most 1023 bits and 4 refs. Model a split with `Cell<T>`; the compiler does not split arbitrary storage structs into refs automatically.
- Optional encodings are schema-specific: generic nullable fields usually add a marker, while `address?` uses the internal-or-none address encoding. An absent trailing field is not automatically a nullable field.
- Eager `fromSlice`/`fromCell` assert end by default with `assertEndAfterReading`. `slice.loadAny` intentionally reads from the middle and ignores that option; `lazy` also ignores it while reading on demand.
- When full lazy validation is necessary, use `forceLoadLazyObject().assertEnd()` on the decoded branch. Referenced `Cell<T>` values still need their own decoding when deep validation is required. Validate a used field explicitly if a lazy skip would otherwise avoid checking it. Do not impose eager validation on every partial read.
- Use `bitsN` for fixed-size binary slices. Built-in `bytesN` is removed in 1.5; casts do not themselves validate N bits/no refs.
- `PackOptions.skipBitsNValidation` is a trust/performance choice, not a replacement for input validation.
- Preserve field order, prefix, inline/ref placement, and hashes when adding custom serializers or alias casts. `RemainingBitsAndRefs` must represent an intentional tail, not arbitrary ignored garbage.

## Standard Contract Families

When implementing standard interfaces, read the standard first and preserve its public shape:

- Jettons: `https://docs.ton.org/standard/tokens/jettons/api`
- NFTs: `https://docs.ton.org/standard/tokens/nft/api`
- Wallet V5: `https://docs.ton.org/standard/wallets/v5-api`
- Vesting: `https://docs.ton.org/standard/vesting`

Typical family-specific patterns:

- Jettons: separate minter/wallet storage, wallet address calculation helper, typed transfer/burn/notification/excess messages, bounce restoration for supply or balances, forward payload as a remainder type.
- NFTs: collection/item split, explicit uninitialized item storage, royalty/getter reply structs, batch deploy map iteration with action-list limits.
- Wallets: external signature validation, expiration, replay state, action validation, and commit/persist ordering.
- Vesting/multisig/policy contracts: typed allowlists as maps, clear authorization methods, explicit time and seqno checks.
