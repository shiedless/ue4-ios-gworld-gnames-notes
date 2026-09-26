<h1 align="center">ue4-ios-gworld-gnames-notes</h1>

<p align="center">finding GWorld and GNames in a UE4 game on iOS — the two globals everything hangs off, and the anti-tamper traps around them</p>

<p align="center">
  <img src="https://img.shields.io/badge/engine-Unreal%20Engine%204-C7192E?style=for-the-badge" alt="engine">
  <img src="https://img.shields.io/badge/platform-iOS%20arm64-000000?style=for-the-badge" alt="platform">
  <img src="https://img.shields.io/badge/targets-GWorld%20%C2%B7%20GNames-1f6feb?style=for-the-badge" alt="targets">
</p>

---

`GWorld` and `GNames` are the two globals everything else hangs off. `GWorld` gets
you into the live scene (actors, players, your pawn); `GNames` turns FName indices
into strings. Get these two and the rest of the SDK falls out.

> This is the arm64/iOS angle specifically — where you don't have the Windows tooling
> everyone's tutorials assume.

---

## contents

- [what they are](#what-they-are)
- [GNames, the easy one first](#gnames-the-easy-one-first)
- [GWorld, and why it's harder now](#gworld-and-why-its-harder-now)
- [the anti-tamper pointer trap](#the-anti-tamper-pointer-trap)
- [the "wrong binary in IDA" trap](#the-wrong-binary-in-ida-trap-this-ones-cheap-to-avoid)
- [verifying you actually got them](#verifying-you-actually-got-them)
- [order I do it in](#order-i-do-it-in)

---

## what they are

`GWorld` is a global `UWorld*`. From it: `PersistentLevel` → the actor array → every
actor in the scene. It's the root of a live read.

`GNames` (the name pool, `FNamePool` on 4.23+) is the interned-string table. You need
it to resolve any FName to text — so class filtering, function matching, all of it
depends on `GNames` being right.

```mermaid
flowchart LR
    gw["GWorld<br/>UWorld*"] --> lvl["PersistentLevel"]
    lvl --> arr["actor array"]
    arr --> act["every actor<br/>players · your pawn"]
    gn["GNames<br/>FNamePool"] --> str["FName index -> string<br/>class / function matching"]

    style gw fill:#C7192E,color:#fff
    style gn fill:#1f6feb,color:#fff
```

Both are just pointers sitting in the binary's data. The job is finding the right
address, and on modern builds, not getting fooled by anti-tamper indirection.

---

## GNames, the easy one first

The name pool almost always resolves cleanly because so much code touches it. Two
ways:

**From the dump.** Whatever tool generated your SDK dump already located the name
pool to print names, so lift its address and layout from the dump's config/output. If
your dump has correct class and function names, its `GNames` is correct by
construction. Start here — it's free.

**From `FName::ToString` / `GetNames`.** Find a function that converts an FName to a
string (xref a plaintext name-ish log string, or find `objc`/error paths that print
names). It will index a global array, block-allocated on 4.23+:

```
entry = Pool.Blocks[index >> 16] + stride * (index & 0xFFFF)
```

The `Pool` base in that expression is `GNames`. Read it off the instruction that
loads it (an `ADRP`/`ADD` or a `LDR` from a data address). Confirm by resolving a
known index and checking you get a sane string. (Layout details are in my fname
notes.)

---

## GWorld, and why it's harder now

On older/naive builds `GWorld` is a fixed global you can read directly — one
`ADRP`+`LDR` and you're in. On a lot of current shipped games it isn't that simple:
the pointer is behind a chain or an anti-tamper accessor, and the value at the
"obvious" address is a decoy or is only valid after decode. **Don't assume the first
candidate is real.**

Ways to land it, roughly in order of how much I trust them:

### 1. Walk the engine chain instead of a raw global

Even when `GWorld` itself is hidden, `GEngine` (or the game instance) usually reaches
the world through a stable member chain:

```mermaid
flowchart TD
    ge["GEngine"] --> gvc["GameViewportClient"]
    ge --> gi["GameInstance"]
    gvc --> w1["World"]
    gi --> w2["World"]
    gi --> lp["LocalPlayers[0]"]
    lp --> pc["PlayerController"]
    pc --> w3["... -> World"]

    style ge fill:#C7192E,color:#fff
    style w1 fill:#1f6feb,color:#fff
    style w2 fill:#1f6feb,color:#fff
    style w3 fill:#1f6feb,color:#fff
```

Find `GEngine` (same technique — an accessor or a well-referenced global), then hop
the members. This survives builds where the bare `GWorld` global is obfuscated,
because the game itself has to traverse this chain to function.

### 2. From a function that clearly uses the world

Anything that spawns, line traces, or iterates actors dereferences the world early.
Xref such a function (a gameplay statics call, a trace) and watch where it loads its
`UWorld*` from. That load site points you at whatever holds the world — global or
chained.

### 3. Cross-check against a live pointer

If you can run in-process or attach, you're not guessing at all — read your
controller/pawn and walk *up*: pawn → player state / controller → the world it lives
in. Then match that runtime address back to the static location that produced it.

---

## the anti-tamper pointer trap

> [!WARNING]
> The one that eats days: on protected builds `GWorld` (and sometimes `GNames`) has
> **no single fixed address** — it's reached through an obfuscated pointer chain or an
> accessor that decodes/validates on each call. The value you read at the address the
> dump lists can be wrong, or right only sometimes.

If your world reads work for a second and then everything goes null, or every derived
actor is garbage, suspect this before you suspect your offsets.

**Handling it:** find the accessor the game itself calls to get the world, and
replicate exactly what it does (the same loads, the same decode), rather than reading
a raw global. Whatever the game does to get a valid `UWorld*` is the correct recipe by
definition.

---

## the "wrong binary in IDA" trap (this one's cheap to avoid)

> [!IMPORTANT]
> If you opened the wrong architecture slice, or rebased, or your image base is off,
> `GWorld` and `GNames` resolve to addresses that are **silently wrong** and everything
> downstream is noise.

I've burned hours on "my offsets are all wrong" that was just this. Confirm your image
base (iOS commonly `0x100000000`) and that you thinned to arm64 **before** you blame
the pointers.

---

## verifying you actually got them

Don't trust a value until it round-trips:

```mermaid
flowchart TD
    gn["GNames"] -->|resolve FName of<br/>PlayerController / GEngine| ok1{"prints expected<br/>class name?"}
    ok1 -->|yes| good1["GNames + layout right"]
    gw["GWorld"] -->|PersistentLevel<br/>-> actor count| ok2{"sane count?<br/>(dozens–hundreds,<br/>not 0, not 4 billion)"}
    ok2 -->|yes| pos["read one actor position"]
    pos -->|finite & plausible| good2["you're in"]
    ok2 -->|no| trap["one pointer off<br/>OR anti-tamper trap"]

    style good1 fill:#2ea043,color:#fff
    style good2 fill:#2ea043,color:#fff
    style trap fill:#C7192E,color:#fff
```

- **`GNames`:** resolve the FName of an object you can name (the local
  `PlayerController`, `GEngine`). If it prints the expected class, `GNames` and its
  layout are right.
- **`GWorld`:** read `PersistentLevel`, then the actor count. A sane count (dozens to
  a few hundred, not 0 and not 4 billion) means the world pointer and the level offset
  are right. Then read one actor's position — if it's finite and in a plausible range,
  you're in.

If the actor count reads as some huge number or zero every frame, you're either one
pointer off in the chain or you fell into the anti-tamper trap above.

---

## order I do it in

1. **`GNames` from the dump first** — free, and you need it to sanity-check everything
   else.
2. **`GEngine`**, then hop to the world through the chain rather than chasing a bare
   `GWorld` global.
3. **Verify** by walking to an actor and reading a position.
4. Only if the chain approach fails do I go looking for a raw `GWorld` global — and if
   I find one on a protected build, I treat it as **suspect until it round-trips**.

---

<p align="center">— shiedless</p>
