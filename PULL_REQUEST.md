# Pull Request: Comprehensive E2E Test Suite and On-Chain Admin Rate Limiting

## Overview

This PR integrates two major features:
1. A comprehensive end-to-end test suite that simulates the complete cross-border payment flow, including KYC submission, quote generation, and final settlement (validating SEP-31 alongside existing SEP-10, SEP-12, SEP-24, and SEP-38).
2. On-chain rate limiting for sensitive administrative actions, enforcing mandatory cooldown periods between admin operations.

## Design Decision
For rate limiting, we implemented **Option A — Per-action cooldown map**. 
A global admin cooldown was considered but rejected because it would artificially block concurrent distinct operations (e.g., an admin might need to pause an asset *and* update an oracle feed simultaneously). By isolating the cooldowns per action type, we maintain high operational flexibility while preventing spam.

## What Changed

### 1. E2E Test Infrastructure
**Files:**
- `backend/src/test/e2e.test.ts` (comprehensive test suite)
- `backend/src/test/sep31-e2e.test.ts` (SEP-31 focused tests)
- `backend/demo-e2e.js` (demonstration script)
- `backend/run-e2e.js` (test runner script)

- Added comprehensive test coverage for SEP-31 cross-border payments.
- Extended existing E2E tests to include SEP-12 KYC flows.
- Created focused test suite for SEP-31 payment lifecycle.
- Added mocking for external services (KYC providers, price feeds, callbacks).
- Created demonstration and runner scripts for test execution.

### 2. Database Schema Updates
**File:** `backend/prisma/schema.prisma`
- Added `Quote` model for handling price quotes in SEP-38.
- Includes fields for asset exchange rates, expiration, and metadata.

### 3. API Route Configuration
**File:** `backend/src/index.ts`
- Added SEP-31 route mounting (`/sep31`).
- Added SEP-12 route mounting (`/sep12`) under rate limiting.
- Added auth route mounting (`/auth`).

### 4. Authentication Service Fixes
**File:** `backend/src/services/auth.service.ts`
- Fixed TypeScript compilation error in `verifySep10ChallengeTransaction` function.
- Corrected function declaration syntax.

### 5. Configuration Cleanup
**File:** `backend/src/config/tracing.ts`
- Removed unused tracing configuration to resolve compilation issues.

### 6. Test Dependencies and Scripts
**File:** `backend/package.json`
- Added `nock` for HTTP request mocking.
- Added test scripts: `test:e2e` and `test:sep31`.
- Updated npm scripts for better test execution.

### 7. Documentation Updates
- Added comprehensive testing section.
- Added rate-limiting architecture docs.

### 8. Rate Limiting Core
- Introduced `ActionType` enum (`AddAsset`, `UpdateOracle`, `SetFee`, `RemoveAsset`, `UpdateAdmin`).
- Implemented `RateLimiter::check_and_update` to validate elapsed time and advance the execution timestamp.
- Implemented `RateLimiter::set_cooldown` with a `MIN_COOLDOWN` guard to prevent misconfiguration (zero-cooldown).
- Used `checked_add` and `checked_sub` for all timestamp arithmetic to prevent integer overflow exploits.

### 9. Storage Key Integration
- Added `ActionCooldown(ActionType)` and `ActionCooldownDuration(ActionType)` to the `DataKey` enum.

### 10. Admin Function Wrapping
- Injected `RateLimiter::check_and_update(&env, ActionType::...)` into all sensitive admin state transition functions.

## Testing

### Prerequisites
```bash
docker-compose up -d
cd backend && npx prisma generate
```

### Running E2E Tests
```bash
cd backend && npm run test:e2e
cd backend && npm run test:sep31
```
