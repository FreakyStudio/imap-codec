# imap-codec — minimal NIL `body-fld-param` quirk

- [x] Add `quirk_body_fld_param_nil_value` (`imap-codec/Cargo.toml`, `imap-codec/src/body.rs`)
- [x] Register quirk in `justfile` `cargo hack` group-features
- [x] Unit test `test_body_fld_param_nil_boundary_quirk` (wire shape from maintainer repro, #700)

**Branch:** `fix/body-fld-param-nil-minimal`  
**Implementation:** `imap-codec/src/body.rs`, `imap-codec/Cargo.toml`, `justfile`

**Next:** Push to fork and open a small PR (short description + link to issue #700 comment); optional: entry in modern-email/defects (upstream).
