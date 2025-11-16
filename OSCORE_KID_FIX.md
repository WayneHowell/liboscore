# liboscore RFC 8613 KID Inclusion Fix

## Problem Summary

liboscore was not including the sender's Key ID (KID) in OSCORE-protected requests, violating RFC 8613 Section 5.4 which states: "The sender SHALL include the 'kid' parameter in requests."

### Observed Behavior
- OSCORE option in requests: `[01, 01]` (2 bytes: flags=0x01, PIV=0x01)
- k-flag (bit 3) not set in flags byte
- KID byte missing from option value

### Expected Behavior (RFC 8613)
- OSCORE option in requests: `[09, 01, 01]` (3 bytes: flags=0x09 with k-flag, PIV=0x01, KID=0x01)
- k-flag (bit 3) set in flags byte (0x08)
- KID byte present after PIV

## Root Cause Analysis

The issue was in `oscore_prepare_request()` in `liboscore/rust/liboscore/c-src/protection.c`:

1. **Flag Ordering Bug**: The `OSCORE_MSG_PROTECTED_FLAG_REQUEST` flag was set AFTER calling `_prepare_encrypt()`
2. **Flag Overwriting**: `_prepare_encrypt()` initializes the flags field, potentially overwriting previously set flags
3. **Timing Issue**: The OSCORE option is written during user closure execution (when `flush_autooptions_outer_until()` is called), which happens INSIDE `_prepare_encrypt()`
4. **Result**: By the time the OSCORE option was being constructed, the REQUEST flag had not yet been set, so the k-flag logic didn't trigger

### Code Flow
```
oscore_prepare_request() 
  → _prepare_encrypt()
      → user closure executes
          → flush_autooptions_outer_until() called (writes OSCORE option)
              → checks REQUEST flag to determine if KID should be included
              → FLAG NOT SET YET!
  → REQUEST flag set (TOO LATE)
```

## The Fix

### Modified Files

#### 1. `liboscore/rust/liboscore/c-src/protection.c`

**Location**: `oscore_prepare_request()` function (around line 570)

**Change**: Set `OSCORE_MSG_PROTECTED_FLAG_REQUEST` BEFORE calling `_prepare_encrypt()`

```c
// Set the REQUEST flag BEFORE _prepare_encrypt so that flush_autooptions_outer_until()
// can see it when building the OSCORE option (required for including KID per RFC 8613 Section 5.4)
unprotected->flags |= OSCORE_MSG_PROTECTED_FLAG_REQUEST;

enum oscore_prepare_result result = _prepare_encrypt(protected, unprotected, secctx);

return result;
```

**Previous Code** (INCORRECT):
```c
enum oscore_prepare_result result = _prepare_encrypt(protected, unprotected, secctx);

// Setting flag AFTER _prepare_encrypt - too late!
unprotected->flags |= OSCORE_MSG_PROTECTED_FLAG_REQUEST;

return result;
```

#### 2. `liboscore/rust/liboscore/c-src/protection.c` - Flag Preservation

**Location**: `_prepare_encrypt()` function (around line 503)

**Existing Code** (CORRECT - preserved flags):
```c
// Initialize everything except the previously initialized partial_iv and request_id
unprotected->backend = protected;

// Set base flags, preserving REQUEST flag if it was already set
uint8_t request_flag = unprotected->flags & OSCORE_MSG_PROTECTED_FLAG_REQUEST;
unprotected->flags = OSCORE_MSG_PROTECTED_FLAG_WRITABLE | OSCORE_MSG_PROTECTED_FLAG_PENDING_OSCORE | request_flag;
```

This code correctly preserves any REQUEST flag that was set before calling `_prepare_encrypt()`.

## Verification

### Test Program
Created `test_oscore_kid.rs` to verify the fix:

```rust
// Uses RFC 8613 C.1.1 test vectors
// Master Secret: 0x0102030405060708090a0b0c0d0e0f10
// Sender ID: [0x01]
// Creates OSCORE-protected request and inspects OSCORE option
```

**Before Fix**:
```
OSCORE option value (2 bytes): [01, 01]
k-flag (KID present): false
✗ FAILURE: KID is NOT present
```

**After Fix**:
```
OSCORE option value (3 bytes): [09, 01, 01]
k-flag (KID present): true
KID value: [01]
✓ SUCCESS: KID is present in OSCORE option (k-flag is set)
```

## Impact

### Benefits
1. **RFC 8613 Compliance**: Requests now properly include sender KID as required
2. **Interoperability**: OSCORE servers can now identify the sender's security context
3. **No Workarounds Needed**: Application code no longer needs to manually modify OSCORE options

### Backwards Compatibility
- **Breaking Change**: Yes, OSCORE option structure changes from 2 bytes to 3 bytes for requests
- **Wire Format**: Increases request size by 1 byte (KID)
- **Servers**: Must be prepared to parse KID from requests (already required by RFC 8613)

## Testing

Run the verification test:
```bash
cargo run --bin test_oscore_kid
```

Expected output:
```
✓ SUCCESS: KID is present in OSCORE option (k-flag is set)
✓ The liboscore KID issue has been RESOLVED!
```

## References

- **RFC 8613**: Object Security for Constrained RESTful Environments (OSCORE)
  - Section 5.4: "The sender SHALL include the 'kid' parameter in requests"
- **liboscore**: https://github.com/coap-security/liboscore
- **Test Vectors**: RFC 8613 Appendix C.1.1

## Related Changes

### Application Code Updates
Removed workaround from `fence_oscore_transmitter_poc_2_1/src/oscore_helpers.rs`:
- Previously manually set k-flag and appended KID byte
- Now relies on corrected liboscore behavior

## Fix Date
2024

## Contributors
Fix developed for fence_oscore_transmitter_poc_2_1 project integration.
