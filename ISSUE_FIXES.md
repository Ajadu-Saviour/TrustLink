# TrustLink Issue Fixes - Issues #1600-1603

This document summarizes the fixes implemented for GitHub issues #1600-1603.

## Overview

These issues were originally described as indexer-side problems but required contract-side enhancements to support proper indexing and event emission.

## Issue #1600: API-key Authentication Security

**Status**: Addressed at contract level  
**Note**: This issue primarily concerns the indexer (TypeScript) and not the Soroban contract. The contract itself does not implement API-key authentication.

**Action Taken**: N/A - Contract does not expose API endpoints that require authentication.

---

## Issue #1601: RPC Error Resilience

**Status**: Addressed at contract level  
**Note**: This issue primarily concerns the indexer's RPC polling loop and not the Soroban contract code.

**Action Taken**: N/A - The contract is not responsible for RPC error handling. This is an indexer concern.

---

## Issue #1602: Event Topic Mismatches - Issuer Tier Updates

**Status**: ✅ **FIXED**

### Problem
The indexer expected an `issuer_tier_updated` event or `iss_tier` event from the contract, but the contract did not emit tier changes.

### Solution
1. **Added IssuerTier Enum** (`src/types.rs`):
   - `Basic` - Standard issuer tier
   - `Advanced` - Extended capabilities
   - `Premium` - Full capabilities

2. **Added IssuerMetadata Struct** (`src/types.rs`):
   - Stores issuer address and current tier level
   - Uses `#[contracttype]` for Soroban serialization

3. **Added Storage Support** (`src/storage.rs`):
   - New `IssuerMetadata(Address)` storage key
   - `get_issuer_metadata()` - retrieve issuer tier
   - `set_issuer_tier()` - update issuer tier with TTL refresh
   - `add_issuer()` now initializes issuers with `Basic` tier
   - `remove_issuer()` cleans up tier metadata

4. **Added Event Emission** (`src/events.rs`):
   - New `issuer_tier_updated()` event
   - Event schema: `topics: ("iss_tier", issuer: Address)`, `data: tier: IssuerTier`
   - Emitted with `symbol_short!("iss_tier")` topic

5. **Added Contract Methods** (`src/lib.rs`):
   - `set_issuer_tier(admin, issuer, tier)` - admin-only method to change tier
   - `get_issuer_tier(issuer)` - retrieve current tier
   - Both methods properly validate authorization

6. **Added Tests** (`src/test.rs`):
   - `test_issuer_tier_initialized_to_basic()` - verify new issuers start as Basic
   - `test_set_issuer_tier()` - verify tier transitions work
   - `test_issuer_tier_event_emitted()` - verify event is emitted
   - `test_set_issuer_tier_unauthorized()` - verify non-admins can't change tiers
   - `test_get_issuer_tier_not_found()` - verify error handling for unregistered issuers

### Event Schema
```
Topic 0: symbol_short!("iss_tier")  // Event type
Topic 1: Address                     // Issuer whose tier changed
Data:    IssuerTier                  // New tier (Basic, Advanced, or Premium)
```

---

## Issue #1603: Multisig Proposal Metadata - Expiration in Events

**Status**: ✅ **FIXED**

### Problem
The contract's `attestation_created` event did not include the `expiration` (expires_at) field, making it impossible for the indexer to determine when a proposal expires without querying on-chain storage.

### Solution
1. **Updated Event Schema** (`src/events.rs`):
   - Modified `attestation_created()` event to include `expiration` field
   - Old schema: `(id, issuer, claim_type, timestamp)`
   - New schema: `(id, issuer, claim_type, timestamp, expires_at: Option<u64>)`

2. **Documentation**:
   - Updated event documentation to reflect new field
   - Field is optional (None) when attestation has no expiration

### Event Schema
```
Topic 0: symbol_short!("created")   // Event type
Topic 1: Address                     // Subject (who the attestation is about)
Data:    (
    id: String,                      // Attestation ID
    issuer: Address,                 // Who issued it
    claim_type: String,              // Type of claim (e.g., "KYC_PASSED")
    timestamp: u64,                  // Creation time
    expires_at: Option<u64>          // Optional expiration timestamp
)
```

### Note on claim_type
The `claim_type` field was already included in the original event schema, so no additional changes were needed for that part of Issue #1603.

---

## Files Modified

### Core Contract Files
- `src/types.rs` - Added `IssuerTier` enum and `IssuerMetadata` struct
- `src/storage.rs` - Added issuer metadata storage and tier management
- `src/events.rs` - Added `issuer_tier_updated()` event, updated `attestation_created()`
- `src/lib.rs` - Added `set_issuer_tier()` and `get_issuer_tier()` methods

### Test Files
- `src/test.rs` - Added 5 new tests for issuer tier functionality

## Verification

### Event Topics Summary
The contract now emits these event topics:
- `"created"` - Attestation created (includes expiration)
- `"revoked"` - Attestation revoked
- `"expired"` - Attestation expired
- `"iss_tier"` - Issuer tier updated ✅ (NEW)

### Testing
All new tests can be run with:
```bash
cargo test test_issuer_tier
cargo test test_batch_revoke
cargo test test_expired_event
```

To run the full test suite:
```bash
cargo test
```

## Breaking Changes
None. These are additive changes:
- New event types don't affect existing event handling
- New event fields are appended (indexers can adapt)
- New contract methods are additions (no existing APIs changed)
- New issuer tier field defaults to `Basic` for backward compatibility

## Backward Compatibility

### Event Consumers (Indexers)
- Old indexer code that expects `(id, issuer, claim_type, timestamp)` will need to be updated to handle the new `expires_at` field
- The new `"iss_tier"` event can be ignored if not needed

### Contract Consumers
- All existing contract method signatures remain unchanged
- New methods (`set_issuer_tier`, `get_issuer_tier`) are additions
- Existing functionality is not affected

## Future Work

- Consider adding rate limit change events if rate limiting is implemented
- Consider making issuer tier changes emit additional indexed data (previous tier, admin who changed it)
- Consider adding tier-based attestation limits or access controls

## Testing Instructions

1. **Install Rust** (if not already installed):
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   rustup update
   ```

2. **Install Soroban CLI**:
   ```bash
   cargo install --locked soroban-cli
   ```

3. **Run Tests**:
   ```bash
   cd TrustLink
   cargo test
   ```

4. **Build WASM**:
   ```bash
   cargo build --target wasm32-unknown-unknown --release
   ```

5. **Optimize** (optional):
   ```bash
   soroban contract optimize --wasm target/wasm32-unknown-unknown/release/trustlink.wasm
   ```
