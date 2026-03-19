# Task 10: Fix provisional log support unit tests

**Status:** DONE
**Component:** MigTD

## Overview

The provisional logging feature (dual-buffer system that captures logs before `EnableLogArea` and copies them to shared memory) is fully implemented. The remaining work is fixing failing unit tests.

## Implementation Summary

**File:** [src/migtd/src/migration/logging.rs](../src/migtd/src/migration/logging.rs)

The logging system uses two buffer pools:
- `PROVISIONAL_LOGAREAPTR` — private 4KB pages for early boot logs
- `LOGAREAPTR` — shared memory pages exposed to VMM

**Flow:**
1. `create_logarea()` ([lines 159-225](../src/migtd/src/migration/logging.rs#L159-L225)) — allocates both buffer sets, enables provisional mode
2. `entrylog()` ([lines 300-531](../src/migtd/src/migration/logging.rs#L300-L531)) — routes writes to provisional or shared buffer based on state
3. `enable_logarea()` ([lines 243-298](../src/migtd/src/migration/logging.rs#L243-L298)) — copies provisional→shared, frees provisional, sets initialized
4. `free_provisional_logarea()` ([lines 227-241](../src/migtd/src/migration/logging.rs#L227-L241)) — deallocates provisional buffers

## Key Structs

| Struct | Lines | Purpose |
|--------|-------|---------|
| `LoggingInformation` | [57-62](../src/migtd/src/migration/logging.rs#L57-L62) | Global state: num_vcpus, flags, entry ID counter, max log level |
| `LogAreaBufferHeader` | [70-77](../src/migtd/src/migration/logging.rs#L70-L77) | Per-buffer header: signature, vcpu index, start/end offsets |
| `LogEntryHeader` | [87-93](../src/migtd/src/migration/logging.rs#L87-L93) | Per-entry header: entry ID, request ID, log level, message length |

## Unit Tests

| # | Test | Lines | What it tests |
|---|------|-------|---------------|
| 1 | `test_create_logarea` | [856-893](../src/migtd/src/migration/logging.rs#L856-L893) | Buffer allocation, LoggingInformation initialization |
| 2 | `test_enable_logarea` | [895-923](../src/migtd/src/migration/logging.rs#L895-L923) | Async enable_logarea(), error when logarea not created |
| 3 | `test_provisional_entrylog` | [926-990](../src/migtd/src/migration/logging.rs#L926-L990) | Logging to provisional buffers, entry ID increments, header validation |
| 4 | `test_entrylog` | [992-1060](../src/migtd/src/migration/logging.rs#L992-L1060) | Regular logging after enable, shared buffer writes |
| 5 | `test_provisional_and_entrylog_combined` | [1062-1151](../src/migtd/src/migration/logging.rs#L1062-L1151) | Full flow: create→provisional log→enable→log again, verifies copy |
| 6 | `test_entrylog_message_max_buffersize` | [1153-1199](../src/migtd/src/migration/logging.rs#L1153-L1199) | Max message fitting in buffer |
| 7 | `test_entrylog_message_greaterthan_buffersize` | [1201-1236](../src/migtd/src/migration/logging.rs#L1201-L1236) | Oversized message rejection |
| 8 | `test_entrylog_message_with_headerlen_left_at_bottom` | [1238-1328](../src/migtd/src/migration/logging.rs#L1238-L1328) | Circular buffer wrap-around (header-sized space left) |
| 9 | `test_entrylog_message_with_lessthan_headerlen_at_bottom` | [1330-1420](../src/migtd/src/migration/logging.rs#L1330-L1420) | Circular buffer wrap (less than header size remaining) |
| 10 | `test_entrylog_message_with_startoffset_at_invalid_message` | [1422-1556](../src/migtd/src/migration/logging.rs#L1422-L1556) | Invalid/corrupted entry handling and buffer reset |
| 11 | `test_log_info` | [1458-1556](../src/migtd/src/migration/logging.rs#L1458-L1556) | log::info! macro integration, migration_request_id extraction |

## Work Required

- ~~Identify which specific tests are failing (run `cargo test` with appropriate features)~~
- ~~Fix test failures~~
- ~~Ensure all 11 tests pass~~

Fixed mismatched `cfg` gates in `data.rs` that prevented compilation with
`vmcall-raw,policy_v2` (without `main`). All 11 logging tests pass.

## How to Run

```sh
# Run logging tests (adjust features as needed)
cargo test --package migtd -p migtd --lib migration::logging --features "vmcall-raw,policy_v2"
```

## Resolution

All 11 logging unit tests pass — both with serial (`--test-threads=1`) and parallel execution.

**Fix applied:** The `RebindingInfo` import and `StartRebinding` variant in `data.rs` were
gated on `vmcall-raw + policy_v2`, but the `rebinding` module requires `main + policy_v2 +
vmcall-raw`. Added the missing `feature = "main"` gate to both cfg attributes so
compilation succeeds when `main` is not enabled.
