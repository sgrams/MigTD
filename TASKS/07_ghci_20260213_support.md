# Task 7: MigTD changes to support GHCI_20260214

**Status:** In Progress
**ETA:** ~2 weeks
**Component:** MigTD
**Branch:** `migtd_ghci_20260213`

## Overview

Update MigTD rebinding flows to align with the GHCI 1.5 (20260214) specification. The current implementation uses a two-phase (prepare/finalize) rebinding model that needs to be consolidated, and several data structures and verification flows need updating.

## Sub-tasks

### 7.1 Clean up prepare/finalize phase — DONE

The current rebinding flow splits into PREPARE and FINALIZE operations. Per the new spec, this split has been removed.

**Files modified:**
- [src/migtd/src/migration/rebinding.rs](../src/migtd/src/migration/rebinding.rs)

**Changes made:**
- Removed `MIGTD_REBIND_OP_PREPARE` / `MIGTD_REBIND_OP_FINALIZE` constants
- Removed `operation` field from `RebindingInfo` struct and its parsing
- Updated reserved field check from `b[11..16]` to `b[10..16]` (bytes 10-15 are reserved per GHCI 1.5)
- Removed `rebinding_old_finalize()` (was a no-op) and `rebinding_new_finalize()` (zeroed token/hash)
- Updated `start_rebinding()` to call prepare functions directly without operation branching
- Both SPDM and non-SPDM paths updated

**Note:** `main.rs` and `cvmemu.rs` dispatch code required no changes — they already call `start_rebinding()` without checking operation.

---

### 7.2 Support updated rebind flows — DONE

Adapted the rebinding handshake, RATLS cert extension creation/verification, and SPDM responder-side verification to use init_tdinfo instead of init_policy per GHCI 1.5.

**Files modified:**
- [src/migtd/src/migration/rebinding.rs](../src/migtd/src/migration/rebinding.rs) — pre-session exchange uses init_tdinfo
- [src/migtd/src/ratls/server_client.rs](../src/migtd/src/ratls/server_client.rs) — RATLS cert creation and verification updated
- [src/migtd/src/spdm/spdm_rsp.rs](../src/migtd/src/spdm/spdm_rsp.rs) — SPDM responder rebinding verification updated

**Changes made (rebinding.rs):**
- `rebinding_old_pre_session_data_exchange()` parameter renamed from `init_policy` to `init_tdinfo`
- `rebinding_new_pre_session_data_exchange()` receives `init_tdinfo` instead of `init_policy`
- `start_rebinding()` passes `init_migtd_data.init_tdinfo` to pre-session exchange

**Changes made (server_client.rs):**
- `client_rebinding()`: parameter renamed `init_td_report` → `init_tdinfo`
- `create_certificate_for_rebinding_old()`: parameter renamed `init_tdreport` → `init_tdinfo`; EXTNID_MIGTD_TDREPORT_INIT extension now receives `&init_tdinfo`
- `verify_rebinding_old_cert()`: cert extension variable renamed `init_td_report` → `init_tdinfo`; pre_session_data second item renamed from `init_policy` to `init_tdinfo`; init_policy_hash verification changed from `digest_sha384(init_policy)` to direct comparison with mrowner bytes (offset 112..160) from init_tdinfo; `authenticate_rebinding_old()` call updated to pass `init_tdinfo` for both init_policy and init_td_report params

**Changes made (spdm_rsp.rs):**
- pre_session_data parsing: renamed `init_policy_offset/size/data` to `init_tdinfo_offset/size/data`
- Removed `digest_sha384(init_policy)` hash check; replaced with direct mrowner bytes (offset 112..160) comparison from `td_report_init_vec`
- `authenticate_rebinding_old()` call updated to pass `&td_report_init_vec` for both init_policy and init_td_report params

---

### 7.3 Cleanup and update MigTD with new MIGTD_DATA_ENTRY_STRUCT — DONE

Updated `InitData` and `MigtdDataEntry` to match GHCI 1.5 (20260214) layout.

**Files modified:**
- [src/migtd/src/migration/rebinding.rs](../src/migtd/src/migration/rebinding.rs)
- [src/migtd/src/spdm/spdm_req.rs](../src/migtd/src/spdm/spdm_req.rs)

**Changes made (rebinding.rs):**
- Replaced 3 data type constants (`INIT_MIG_POLICY=0`, `INIT_TD_REPORT=1`, `INIT_EVENT_LOG=2`) with single `MIGTD_DATA_TYPE_TDINFO = 0`
- `InitData` struct now holds `init_tdinfo: Vec<u8>` (TDINFO_STRUCT) instead of `init_report`, `init_policy`, `init_event_log`
- Added `mrowner()` and `mrownerconfig()` helper methods to extract fields from TDINFO_STRUCT
  - Per GHCI 1.5: VMM puts `migpolicy.policy_key` in `tdinfo.mrowner` and `migpolicy.policy_svn` in `tdinfo.mrownerconfig`
- `read_from_bytes()` now enforces `numberOfEntry == 1` and `type == 0` (TDINFO) per spec
- `write_into_bytes()` serializes a single TDINFO entry
- `get_from_local()` now extracts `td_info` from the TDX report instead of the full report
- `rebinding_old_prepare()` (non-SPDM path) updated to use `mrowner()` as init_policy_hash and `init_tdinfo` as init_report equivalent; uses local event_log as fallback
- Removed unused `digest_sha384` import

**Changes made (spdm_req.rs):**
- SPDM VDM message `TdReportInit` element now sends `init_tdinfo` instead of full tdreport
- `EventLogInit` element falls back to local event log (no longer in MIGTD_DATA)
- `MigPolicyInit` element uses `mrowner()` from init_tdinfo instead of hashing init_policy

---

### 7.4 Update MigTD flows w.r.t policy verifications and SERVTD_EXT verifications — NOT STARTED

Policy and SERVTD_EXT verification logic needs updating per the new spec.

**Files to modify:**
- [src/migtd/src/mig_policy.rs](../src/migtd/src/mig_policy.rs) — `authenticate_rebinding_old()`, `authenticate_rebinding_new()`, `verify_servtd_hash()`, `verify_init_tdreport()`
- [src/migtd/src/migration/servtd_ext.rs](../src/migtd/src/migration/servtd_ext.rs) — `ServtdExt` struct, `read_servtd_ext()`, `write_approved_servtd_ext_hash()`
- [src/migtd/src/ratls/server_client.rs](../src/migtd/src/ratls/server_client.rs) — `verify_rebinding_old_cert()`, `verify_rebinding_new_cert()`, certificate creation with SERVTD_EXT extensions
- [src/migtd/src/ratls/mod.rs](../src/migtd/src/ratls/mod.rs) — Extension OIDs (`EXTNID_MIGTD_SERVTD_EXT`, etc.)

**Work required:**
- Update `ServtdExt` struct if fields changed in new spec
- Update policy verification to check `tdinfo.mrowner` matches local `policy_key` and `tdinfo.mrownerconfig` matches local `policy_svn`
- Update X.509 certificate extension OIDs if changed
- Update SPDM VDM message element types if changed
- Ensure attribute masking flags in `verify_servtd_hash()` match new spec
- Update `verify_init_tdreport()` to verify TDINFO_STRUCT instead of full TDREPORT
- Add `curr_servtd_attr` verification per GHCI 1.5 requirement

---

## TODO markers left in code

The following `TODO(ghci_20260214)` markers were added for follow-up work:
- `rebinding.rs:581` — Update RATLS cert extension to remove init_event_log dependency
- `spdm_req.rs` — Consider updating `VdmMessageElementType::TdReportInit` to `TdInfoInit`
- `spdm_req.rs` — Update VDM protocol to remove init_event_log dependency
- `spdm_req.rs` — Update VDM protocol to use mrowner directly

## Testing

- Unit tests for updated `MigtdDataEntry` / `InitData` parsing
- Unit tests for policy verification with new parameters
- End-to-end integration testing with VMM (Task 9 dependency)
- CVM emulator (`cvmemu.rs`) flow validation
