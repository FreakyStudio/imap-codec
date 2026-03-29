# Himalaya & CRM: using patched `imap-codec` (Mail.ru `body-fld-param` NIL quirk)

This document tells an agent or developer how to wire **[`imap-codec`](https://github.com/duesee/imap-codec)** with the **`quirk_body_fld_param_nil_value`** fix so **Himalaya**, **email-lib / imap-client**, or a **custom CRM** can parse Mail.ru (and similar) **BODYSTRUCTURE** lines that contain `("boundary" NIL)` instead of a proper string.

Upstream context: [imap-codec#700](https://github.com/duesee/imap-codec/issues/700).

---

## 1. What is being patched?

| Item | Value |
|------|--------|
| **Fork** | `https://github.com/FreakyStudio/imap-codec.git` |
| **Branch** | `fix/body-fld-param-nil-minimal` |
| **Crate** | `imap-codec` (version `2.0.0-alpha.7`, aligned with `imap-next` on crates.io) |
| **Feature** | `quirk_body_fld_param_nil_value` (included in the crate’s default **`quirk`** bundle) |

**Behavior:** where RFC 3501 expects a `string` as the second element of a `body-fld-param` pair, some servers send **`NIL`**. The quirk accepts that and maps it to an **empty** `IString` (e.g. boundary shows as `Quoted("")` in debug output).

**Not covered by this branch:** other Mail.ru quirks (e.g. empty `""` in envelope addresses) are **out of scope** here; if FETCH still fails, capture the line and consider an additional quirk or patch.

---

## 2. Dependency chain (why one patch is enough)

Typical stack:

```text
himalaya / your CRM
  → email-lib (optional)
  → imap-client (pimalaya `io-imap`)
    → imap-next
      → imap-codec   ← patch this crate
```

`imap-next` depends on `imap-codec` roughly as:

```toml
imap-codec = { version = "2.0.0-alpha.7", features = ["quirk_crlf_relaxed"] }
```

Cargo **does not** set `default-features = false` on `imap-codec`, so **`imap-codec`’s default `quirk` feature stays on**, which includes **`quirk_body_fld_param_nil_value`** on this fork. You normally **do not** need to fork `imap-next` only to enable this quirk.

---

## 3. Workspace `[patch.crates-io]` (Himalaya or any binary / CRM root)

Add a **`[patch.crates-io]`** table to the **workspace root** `Cargo.toml` (the same manifest that contains `[package]` or `[workspace]` for the app you are building).

**Minimal patch** (keep any **existing** `[patch.crates-io]` entries; merge tables):

```toml
[patch.crates-io]
imap-codec = { git = "https://github.com/FreakyStudio/imap-codec.git", branch = "fix/body-fld-param-nil-minimal" }
```

Then:

```bash
cargo update -p imap-codec
cargo build
```

**Himalaya upstream** already patches other crates; its root `Cargo.toml` may look like:

```toml
[patch.crates-io]
pimalaya-tui = { git = "https://github.com/pimalaya/tui" }
imap-client = { git = "https://github.com/pimalaya/io-imap" }
```

**Append** the `imap-codec` line to that same table (do not create a second `[patch.crates-io]`).

---

## 4. Pinning / reproducibility

- After patching, commit **`Cargo.lock`** so CI and other machines resolve the same **git revision** of `imap-codec`.
- To pin an exact commit instead of a branch:

```toml
imap-codec = { git = "https://github.com/FreakyStudio/imap-codec.git", rev = "f24c447" }
```

(Replace `rev` with the current commit from the branch.)

---

## 5. Local path patch (monorepo / vendor)

If the CRM workspace **clones** this repo next to the app:

```toml
[patch.crates-io]
imap-codec = { path = "../imap-codec/imap-codec" }
```

Use the path to the **inner** crate directory that contains `imap-codec`’s `Cargo.toml`, not only the repo root (the repo is a **workspace** with `imap-codec` and `imap-types` members).

---

## 6. Verifying the patch is active

From the patched workspace root:

```bash
cargo tree -p imap-codec -i imap-codec
```

You should see the dependency coming from **git** (or **path**), not only `registry+…`.

**Optional:** run `imap-codec`’s own tests on the fork:

```bash
cd path/to/imap-codec/imap-codec
cargo test test_body_fld_param_nil
```

**Optional:** end-to-end parse check using the upstream example (from `imap-codec` crate directory):

```bash
cargo run -p imap-codec --example client
```

Paste a greeting line, then a Mail.ru-shaped `FETCH` with `("boundary" NIL)`; with the quirk bundle active it should parse as **`Data(Fetch { … })`**, not **`DecodingFailure`**.

---

## 7. Building Himalaya from source with the patch

1. Clone [pimalaya/himalaya](https://github.com/pimalaya/himalaya).
2. Edit **root** `Cargo.toml`: add `imap-codec` under `[patch.crates-io]` as in §3 (merge with existing patches).
3. `cargo build --release` (or `cargo install --path .`).

**Rust version:** match `rust-toolchain.toml` / MSRV in Himalaya and `imap-codec` (workspace MSRV is part of the fork’s root `Cargo.toml`).

---

## 8. Integrating into a custom Desktop Rust CRM

1. Add the same **`[patch.crates-io]`** at the **workspace root** of the CRM (one place for the whole dependency graph).
2. Depend on IMAP as you already plan (e.g. **`imap-client`**, **`email-lib`**, or **`imap-codec`** directly). Do not duplicate conflicting versions of `imap-codec`.
3. If something in the tree uses **`imap-codec` with `default-features = false`**, you must enable quirks explicitly, e.g.:

   ```toml
   imap-codec = { version = "2.0.0-alpha.7", default-features = false, features = ["quirk"] }
   ```

   or at minimum `quirk_body_fld_param_nil_value` (plus any other quirks you rely on).

4. **Mail.ru account config** (host, TLS, OAuth/password) is orthogonal; this patch only fixes **protocol parsing** for the NIL-boundary BODYSTRUCTURE case.

---

## 9. Troubleshooting

| Symptom | What to check |
|--------|----------------|
| `failed to select a version` / patch ignored | Patch is on the **workspace root** that owns the build; run `cargo update -p imap-codec`. |
| Still `DecodingFailure` on FETCH | Confirm `cargo tree` shows **patched** `imap-codec`; failure may be a **different** syntax issue (capture bytes). |
| Duplicate patch definitions | Only one `[patch.crates-io]` table; merge all patches. |
| Version conflict | Fork branch must stay compatible with **`imap-codec = 2.0.0-alpha.7`** (or bump fork + align `imap-next` if you control both). |

---

## 10. Reference links

- Fork branch: [FreakyStudio/imap-codec @ `fix/body-fld-param-nil-minimal`](https://github.com/FreakyStudio/imap-codec/tree/fix/body-fld-param-nil-minimal)
- Upstream issue: [duesee/imap-codec#700](https://github.com/duesee/imap-codec/issues/700)
- Himalaya: [pimalaya/himalaya](https://github.com/pimalaya/himalaya)
- IMAP client library: [pimalaya/io-imap](https://github.com/pimalaya/io-imap) (`imap-client` on crates.io)
