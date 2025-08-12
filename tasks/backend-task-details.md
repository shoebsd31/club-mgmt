# Backend Task Breakdown & Acceptance Criteria (ASP.NET Core + PostgreSQL)

**Stack assumptions**
- ASP.NET Core 8 (Minimal APIs/Controllers), C# 12
- EF Core 8 with Npgsql provider (or Dapper for read-heavy endpoints as noted)
- JWT authentication; policy-based authorization
- FluentValidation for DTO validation
- ProblemDetails (RFC 7807) for error responses
- xUnit + FluentAssertions + NSubstitute/Moq for unit tests
- Testcontainers for repository/integration-style unit tests against Postgres (optional but recommended)

---

## 1) Auth & Identity

### POST `/auth/login/google`
**Purpose:** Exchange Google ID token for app JWT + refresh token; link or create user & identity.

**Tasks**
- Validate request: `idToken` required.
- Verify Google token signature and audience using Google library or JWKS (abstracted `IGoogleTokenValidator`).
- Lookup or create `UserIdentities(provider=google, provider_subject)` and corresponding `Users`.
- Issue `AuthTokens` (JWT + refresh); persist session in `user_sessions` with hashed refresh token.
- Upsert `last_login_at` for user and identity.
- Return `AuthResponse` with tokens and `User` DTO.

**Security**
- No auth required; strict token verification and issuer/audience checks.

**Errors**
- 401 if token invalid.
- 409 if identity conflict (rare; handle by linking).

**Acceptance Criteria (Unit Tests)**
- Returns 200 with tokens and user when Google token valid.
- Creates new user+identity when none exists.
- Reuses existing user+identity when exists; updates `last_login_at`.
- Persists `user_sessions` with refresh hash and expiry.
- Returns 401 for invalid token.
- Emits ProblemDetails on failure.

---

### POST `/auth/login/apple`
_Same as Google flow; replace validator with `IAppleTokenValidator`._

**Acceptance Criteria**
- Mirrors Google tests for Apple tokens.

---

### POST `/auth/login/otp/request`
**Purpose:** Send OTP to phone; create hashed OTP entry.

**Tasks**
- Validate `phone` (E.164).
- Rate limit per phone and IP (e.g., 5 req / 15 min).
- Generate OTP (6 digits), hash with pepper.
- Upsert user by phone if not exists.
- Persist `otp_codes` with expiry (e.g., 5 minutes).
- Send via SMS provider (abstract `ISmsSender`).

**Errors**
- 429 when rate limit exceeded.

**Acceptance Criteria**
- Stores hashed OTP with correct expiry and purpose.
- Invokes `ISmsSender` exactly once.
- Returns 200 even if phone not recognized (don’t leak existence).
- Enforces rate limiting; returns 429 when exceeded.

---

### POST `/auth/login/otp/verify`
**Purpose:** Verify OTP; issue tokens; delete/consume OTP.

**Tasks**
- Validate `phone`, `code` present; compare `code_hash` securely.
- Enforce attempt counter; lock after N attempts.
- Mark `consumed_at` on success; create session + tokens.
- Upsert user if not exists; mark `phone_verified=true`.

**Errors**
- 401 on wrong/expired OTP.
- 409 if already consumed.

**Acceptance Criteria**
- Succeeds with correct OTP; returns tokens.
- Fails with 401 for wrong/expired.
- Increments `attempt_count`; locks after threshold.
- Marks OTP consumed and creates `user_sessions` row.

---

### POST `/auth/refresh`
**Purpose:** Exchange refresh token for new access/refresh tokens.

**Tasks**
- Validate refresh token; lookup session by refresh hash.
- Check not revoked and not expired.
- Rotate tokens (invalidate old session or update with new hash).
- Return new `AuthTokens`.

**Acceptance Criteria**
- Returns new tokens for valid session.
- Rejects invalid/expired/revoked with 401.
- Rotates refresh token; previous token becomes invalid.

---

### POST `/auth/logout`
**Purpose:** Revoke current session.

**Tasks**
- Require `bearerAuth`; read session from context.
- Set `revoked_at` (soft revoke).

**Acceptance Criteria**
- Returns 204; session marked revoked in store.
- Revoked session cannot be refreshed afterward.

---

## 2) Users & Memberships

### GET `/users/me`
**Purpose:** Return current user profile.

**Tasks**
- Require `bearerAuth`.
- Map domain `User` → DTO; include `createdAt`, `lastLoginAt`.

**Acceptance Criteria**
- 200 with correct user details.
- 401 when missing/invalid token.

---

### GET `/memberships/me`
**Purpose:** List memberships for current user.

**Tasks**
- Join `members` by `users.id`, then list `memberships` with club info.
- Optionally cache short TTL for performance.

**Acceptance Criteria**
- 200 with array of user’s memberships.
- Includes `clubId`, `type`, `status`, `startDate`.

---

## 3) Clubs, Venues, Courts

### GET `/clubs`
**Purpose:** Paged list of clubs.

**Tasks**
- Support `page`, `pageSize` with defaults.
- Order by `created_at DESC`.
- Return `PagedClubs` with `PageMeta`.

**Acceptance Criteria**
- 200 returns `data` and `meta` correctly.
- Page bounds respected; pageSize capped (<=200).

---

### POST `/clubs`
**Purpose:** Create a new club (admin/organizer).

**Tasks**
- Require `bearerAuth` & `ClubCreatePolicy`.
- Validate input; insert into `clubs`.
- Return 201 with created entity.

**Acceptance Criteria**
- 201 and persisted club with `id`.
- Reject unauthenticated/unauthorized with 401/403.
- Validates `name` non-empty.

---

### GET `/clubs/{clubId}`
**Purpose:** Get club details.

**Acceptance Criteria**
- 200 with club when exists.
- 404 when not found.

---

### PATCH `/clubs/{clubId}`
**Purpose:** Update club info.

**Acceptance Criteria**
- 200 with updated fields.
- 404 if not found; 403 if unauthorized.
- Only provided fields updated (partial).

---

### DELETE `/clubs/{clubId}`
**Purpose:** Delete a club (soft or hard).

**Acceptance Criteria**
- 204 on success; cascades/constraints honored.
- 404 when not found; 403 unauthorized.

---

### GET `/clubs/{clubId}/venues`
**Purpose:** List venues in a club.

**Acceptance Criteria**
- 200 with array; only venues for that club.

---

### POST `/clubs/{clubId}/venues`
**Purpose:** Create venue in a club.

**Acceptance Criteria**
- 201 with created venue.
- 404 if `clubId` invalid; validation for `name` required.

---

### GET `/venues/{venueId}/courts`
**Purpose:** List courts for a venue.

**Acceptance Criteria**
- 200 with courts; filters by `venueId` only.

---

### POST `/venues/{venueId}/courts`
**Purpose:** Create court under venue.

**Acceptance Criteria**
- 201 with created court; `code` unique per venue.
- 404 if venue missing; validates `code` not empty.

---

## 4) Sessions & RSVP

### GET `/sessions`
**Purpose:** Query sessions by `clubId` and optional date range.

**Acceptance Criteria**
- 200 with sessions in range; default window if none supplied.
- Validates `from <= to`.

---

### POST `/sessions`
**Purpose:** Create playing session (organizer).

**Acceptance Criteria**
- 201 with created session; requires `clubId`, `venueId`, `startAt`.
- 403 if user not organizer/admin in club.

---

### GET `/sessions/{sessionId}`
**Acceptance Criteria**
- 200 when session exists; 404 otherwise.

---

### PATCH `/sessions/{sessionId}`
**Acceptance Criteria**
- 200 with updated session; only allowed fields changed.
- 403 if not organizer/admin; 404 if missing.

---

### DELETE `/sessions/{sessionId}`
**Acceptance Criteria**
- 204 on success; 403 if not authorized; 404 if missing.

---

### GET `/sessions/{sessionId}/rsvps`
**Acceptance Criteria**
- 200 with list of RSVPs; includes `status`, `guestCount`, `rsvpAt`.

---

### POST `/sessions/{sessionId}/rsvp`
**Purpose:** Create/update current user RSVP.

**Acceptance Criteria**
- 200 with RSVP updated to `yes|no|maybe`.
- Validates member belongs to the club of the session.
- Debounces duplicate submissions (idempotent by `(sessionId,userId)`).

---

## 5) Wallet & Club Transactions

### GET `/wallet`
**Purpose:** Return current user’s wallet summary.

**Acceptance Criteria**
- 200 with `balance` and `updatedAt`.
- Creates wallet on-the-fly if missing.

---

### GET `/wallet/transactions`
**Purpose:** Paged list of wallet transactions.

**Acceptance Criteria**
- 200 with `data` and `meta`.
- Sorted by `createdAt DESC`.

---

### GET `/clubs/{clubId}/transactions`
**Purpose:** List club-level transactions (finance).

**Acceptance Criteria**
- 200 with transactions; 403 if user lacks finance permission.

---

### POST `/clubs/{clubId}/transactions`
**Purpose:** Add club transaction (expense/income).

**Acceptance Criteria**
- 201 created; numeric `amount` > 0; `direction` valid.
- 403 if not authorized; 404 if club missing.

---

## 6) Tournaments, Divisions, Registrations, Brackets, Matches

### GET `/tournaments`
**Purpose:** List tournaments, filter by `clubId` and `status`.

**Acceptance Criteria**
- 200 with filtered list; supports pagination if needed.

---

### POST `/tournaments`
**Purpose:** Create tournament (organizer).

**Acceptance Criteria**
- 201 with tournament; validates `name`, `format`, `startDate`.
- 403 if not organizer/admin.

---

### GET `/tournaments/{tournamentId}`
**Acceptance Criteria**
- 200 when found; 404 otherwise.

---

### PATCH `/tournaments/{tournamentId}`
**Acceptance Criteria**
- 200 with updated values; 403 unauthorized; 404 not found.

---

### GET `/tournaments/{tournamentId}/divisions`
**Acceptance Criteria**
- 200 with divisions for tournament.

---

### POST `/tournaments/{tournamentId}/divisions`
**Acceptance Criteria**
- 201 created; `name` and `type` required; `gender` valid enum.
- 403 unauthorized; 404 when tournament missing.

---

### POST `/divisions/{divisionId}/registrations`
**Purpose:** Register a team/player in a division.

**Acceptance Criteria**
- 201 with registration; unique `(divisionId, teamId)` enforced.
- 409 when division full; 404 if division missing.

---

### POST `/registrations/{registrationId}/checkout`
**Purpose:** Create Stripe Checkout Session; return redirect `url` and `checkoutSessionId`.

**Acceptance Criteria**
- 200 with valid URL; stores Stripe IDs.
- 400 if registration already paid/cancelled.
- 403 if user not part of team.

---

### GET `/registrations/{registrationId}`
**Acceptance Criteria**
- 200 with registration; includes payment status; 404 if missing.

---

### GET `/divisions/{divisionId}/brackets`
**Purpose:** Retrieve generated brackets for division.

**Acceptance Criteria**
- 200 with list; empty array if not generated.

---

### POST `/divisions/{divisionId}/brackets`
**Purpose:** Generate brackets for a division.

**Acceptance Criteria**
- 201 with generated brackets; respects division teams and type.
- 409 if already generated and locked; 403 unauthorized.

---

### GET `/matches/{matchId}`
**Acceptance Criteria**
- 200 with match details and sets; 404 if missing.

---

### PATCH `/matches/{matchId}`
**Purpose:** Update match result/status (admin/table official).

**Acceptance Criteria**
- 200 updated; verifies team IDs belong to bracket; recomputes winner.
- 403 unauthorized; 409 if match already finalized.

---

## 7) Webhooks

### POST `/webhooks/stripe`
**Purpose:** Consume Stripe events to update payment status and registrations.

**Tasks**
- Verify Stripe signature using webhook secret.
- Handle: `checkout.session.completed`, `payment_intent.succeeded`, `charge.refunded`.
- Update `stripe_payments` and `registrations.status` accordingly.
- Idempotency: dedupe by event `id` in a table `webhook_events`.

**Acceptance Criteria**
- Returns 200 for valid signed events.
- Ignores duplicate events idempotently.
- Updates registration to `paid` on success; to `refunded` on refund.

---

## Cross-Cutting Concerns

**Logging & Telemetry**
- Correlation ID per request; audit log for auth, finance, tournament mutations.

**Validation**
- FluentValidation classes for all Create/Update DTOs; ProblemDetails for violations.

**Error Handling**
- Consistent ProblemDetails payloads; no stack traces in production.

**Security**
- Role/Policy-based authorization (Organizer, Admin, Finance, Member).

**Database**
- Use transactions for multi-write flows (auth create session, registration + checkout).

**Idempotency**
- Consider Idempotency-Key header for POST endpoints that create resources.

**Pagination**
- Uniform `page`, `pageSize`, and `PageMeta` in list endpoints.

---

## Example DTO & Unit Test Sketch (C#)

```csharp
public sealed record SessionCreateDto(Guid ClubId, Guid VenueId, DateTimeOffset StartAt, DateTimeOffset? EndAt, string? Notes, int MaxGuestsPerMember);

public class CreateSessionHandlerTests
{
    [Fact]
    public async Task Creates_session_and_returns_201()
    {
        // Arrange
        var repo = Substitute.For<ISessionRepository>();
        var validator = new SessionCreateValidator();
        var sut = new CreateSessionHandler(repo, validator, Clock.System);

        var dto = new SessionCreateDto(Guid.NewGuid(), Guid.NewGuid(), DateTimeOffset.UtcNow.AddDays(1), null, "Afternoon", 2);

        // Act
        var result = await sut.HandleAsync(dto, userContext: OrganizerInClub());

        // Assert
        result.StatusCode.Should().Be(201);
        await repo.Received(1).CreateAsync(Arg.Is<Session>(s => s.ClubId == dto.ClubId && s.VenueId == dto.VenueId));
    }
}
```

---

## Definition of Done (per endpoint)
- ✅ DTOs and validators implemented
- ✅ Controller/Minimal API endpoint implemented with authorization
- ✅ Repository methods with unit tests (mocked DB) and (optional) Testcontainers integration tests
- ✅ Happy-path and failure-path unit tests (validation, 401/403/404/409)
- ✅ ProblemDetails responses for all errors
- ✅ OpenAPI annotations consistent with `club-api.yaml`
- ✅ Postgres migrations updated; seeded data where applicable
- ✅ Observability: structured logs, metrics counters, trace ids