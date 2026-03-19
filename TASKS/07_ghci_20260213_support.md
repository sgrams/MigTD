# Task 7: MigTD changes to support GHCI_20260213

**Status:** Not Started
**ETA:** ~2 weeks
**Component:** MigTD

## Overview

Update MigTD rebinding flows to align with the GHCI_20260213 specification. The current implementation uses a two-phase (prepare/finalize) rebinding model that needs to be consolidated, and several data structures and verification flows need updating.

## Sub-tasks

### 7.1 Clean up prepare/finalize phase

The current rebinding flow splits into PREPARE and FINALIZE operations. Per the new spec, this split should be removed.

**Files to modify:**
- [src/migtd/src/migration/rebinding.rs](../src/migtd/src/migration/rebinding.rs)

**Current state:**
- Constants at [lines 51-52](../src/migtd/src/migration/rebinding.rs#L51-L52):
  ```rust
  const MIGTD_REBIND_OP_PREPARE: u8 = 0;
  const MIGTD_REBIND_OP_FINALIZE: u8 = 1;
  ```
- `start_rebinding()` ([lines 327-421](../src/migtd/src/migration/rebinding.rs#L327-L421)) branches on `info.operation` to select PREPARE or FINALIZE for each side (old/new TD)
- Four separate functions exist:
  - `rebinding_old_prepare()` — TLS client / SPDM requester for source TD
  - `rebinding_new_prepare()` — TLS server / SPDM responder for destination TD
  - `rebinding_old_finalize()` — no-op (returns Ok)
  - `rebinding_new_finalize()` — zeroes session token and servtd_ext hash

**Work required:**
- Remove `MIGTD_REBIND_OP_PREPARE` / `MIGTD_REBIND_OP_FINALIZE` constants
- Remove `operation` field handling from `RebindingInfo` parsing ([lines 106-133](../src/migtd/src/migration/rebinding.rs#L106-L133))
- Merge prepare + finalize logic into a single flow per side (old/new)
- Inline finalize cleanup (zeroing token/hash) into the end of the prepare path
- Update `start_rebinding()` to remove the operation branch

**Also update:**
- [src/migtd/src/migration/session.rs](../src/migtd/src/migration/session.rs) — request parsing that populates `operation` field
- [src/migtd/src/bin/migtd/main.rs](../src/migtd/src/bin/migtd/main.rs) — `StartRebinding` dispatch
- [src/migtd/src/bin/migtd/cvmemu.rs](../src/migtd/src/bin/migtd/cvmemu.rs) — emulator rebinding calls

---

### 7.2 Support updated rebind flows

Adapt the rebinding handshake to the new spec flow (single-phase).

**Files to modify:**
- [src/migtd/src/migration/rebinding.rs](../src/migtd/src/migration/rebinding.rs) — core flow
- [src/migtd/src/ratls/server_client.rs](../src/migtd/src/ratls/server_client.rs) — TLS rebinding functions (`server_rebinding()`, `client_rebinding()`)
- [src/migtd/src/spdm/spdm_req.rs](../src/migtd/src/spdm/spdm_req.rs) — SPDM rebinding requester/responder

**Work required:**
- Update the TLS/SPDM handshake flows to match new spec sequencing
- Verify certificate extension creation/verification still aligns with updated flow
- Update VDM message element ordering if spec changed it

---

### 7.3 Cleanup and update MigTD with new MIGTD_DATA_ENTRY_STRUCT

The `MigtdDataEntry` struct and `InitData` parsing need to match the GHCI_20260213 layout.

**Files to modify:**
- [src/migtd/src/migration/rebinding.rs](../src/migtd/src/migration/rebinding.rs)

**Current state:**
- `MigtdDataEntry` ([lines 244-251](../src/migtd/src/migration/rebinding.rs#L244-L251)): `type` (u32) + `length` (u32) + `value` (&[u8])
- `InitData` ([lines 137-154](../src/migtd/src/migration/rebinding.rs#L137-L154)): `init_report`, `init_policy`, `init_event_log` (all `Vec<u8>`)
- Data type constants ([lines 48-50](../src/migtd/src/migration/rebinding.rs#L48-L50)):
  ```rust
  MIGTD_DATA_TYPE_INIT_MIG_POLICY = 0
  MIGTD_DATA_TYPE_INIT_TD_REPORT  = 1
  MIGTD_DATA_TYPE_INIT_EVENT_LOG  = 2
  ```
- `InitData::read_from_bytes()` ([lines 156-190](../src/migtd/src/migration/rebinding.rs#L156-L190)) validates `MIGTD_DATA_SIGNATURE` ("MIGTDATA"), version, and parses TLV entries
- `InitData::write_into_bytes()` ([lines 192-216](../src/migtd/src/migration/rebinding.rs#L192-L216)) serializes with header + 3 entries

**Work required:**
- Update `MigtdDataEntry` fields to match new GHCI_20260213 struct definition
- Update data type constants if new types were added or values changed
- Update `read_from_bytes()` / `write_into_bytes()` to handle new layout
- Update `RebindingInfo` ([lines 93-104](../src/migtd/src/migration/rebinding.rs#L93-L104)) if the VMM request format changed

---

### 7.4 Update MigTD flows w.r.t policy verifications and SERVTD_EXT verifications

Policy and SERVTD_EXT verification logic needs updating per the new spec.

**Files to modify:**
- [src/migtd/src/mig_policy.rs](../src/migtd/src/mig_policy.rs) — `authenticate_rebinding_old()`, `authenticate_rebinding_new()`, `verify_servtd_hash()`, `verify_init_tdreport()`
- [src/migtd/src/migration/servtd_ext.rs](../src/migtd/src/migration/servtd_ext.rs) — `ServtdExt` struct, `read_servtd_ext()`, `write_approved_servtd_ext_hash()`
- [src/migtd/src/ratls/server_client.rs](../src/migtd/src/ratls/server_client.rs) — `verify_rebinding_old_cert()`, `verify_rebinding_new_cert()`, certificate creation with SERVTD_EXT extensions
- [src/migtd/src/ratls/mod.rs](../src/migtd/src/ratls/mod.rs) — Extension OIDs (`EXTNID_MIGTD_SERVTD_EXT`, etc.)

**Work required:**
- Update `ServtdExt` struct if fields changed in new spec
- Update policy verification parameters and checks
- Update X.509 certificate extension OIDs if changed
- Update SPDM VDM message element types if changed
- Ensure attribute masking flags in `verify_servtd_hash()` match new spec

---

## Testing

- Unit tests for updated `MigtdDataEntry` / `InitData` parsing
- Unit tests for policy verification with new parameters
- End-to-end integration testing with VMM (Task 9 dependency)
- CVM emulator (`cvmemu.rs`) flow validation
