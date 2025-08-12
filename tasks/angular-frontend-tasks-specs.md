# Angular Frontend Task Specification — Club & Tournament System

**Framework:** Angular 18+, Standalone APIs, Angular Router, HttpClient, Signals/NgRx (optional)  
**UI:** Angular Material (M3) or Tailwind; responsive layouts (mobile-first)  
**Testing:** Jasmine/Karma unit tests (components/services/guards/interceptors) + Playwright e2e  
**Access Control:** Route guards (AuthGuard), HTTP interceptor (JWT, error handling, retry)  
**State:** Query-first via services; cache with signals/NgRx; SW for offline (optional)

> This plan maps to the previously defined API endpoints and tasks. See “Frontend Tasks and API Mappings”.

---

## Conventions (apply to all tasks)
- **Accessibility:** WCAG 2.1 AA; semantic HTML, ARIA where needed; focus management; keyboard navigability.
- **i18n:** Extract strings with `$localize`; support en/de by default.
- **Error UX:** Non-blocking snackbars for transient errors; inline messages for form validation.
- **Loading UX:** Skeletons for lists; progress indicators for async actions; optimistic updates where safe.
- **Security:** Never store tokens in localStorage without guarding; prefer memory + refresh; if persisted, use Web Crypto and HttpOnly cookies where applicable.
- **Testing DoD:** Each task lists **Unit** (Jasmine) + **Playwright** acceptance scenarios. Aim for 80%+ coverage per feature module.

---

## 1) Google login button with JWT retrieval and auto-session
**Description:** Google One-Tap/redirect sign-in; exchange `idToken` for app JWT.  
**API:** `POST /auth/login/google`  
**UI/UX:** Prominent button; loading state; error toast; redirects to dashboard on success.  
**State:** Store `accessToken`,`refreshToken`,`user`; emit auth signal.  
**Acceptance (Unit):**
- AuthService calls `/auth/login/google` with provided idToken; returns tokens+user.
- Tokens stored via AuthStore; navigation to `/dashboard` occurs.
- Errors surface snackbar + no token stored.
**Playwright:**
- Clicking “Continue with Google” (mock provider) leads to dashboard with welcome text.
- On API 401, remains on login; shows error toast.
**Notes:** Use Google Identity Services; test with mock provider.

---

## 2) Apple login button with JWT retrieval and auto-session
**API:** `POST /auth/login/apple`  
**Acceptance (Unit):**
- AuthService posts idToken; persists tokens; navigates on success.
- Error path handled as above.
**Playwright:**
- Mock Apple flow; verify success navigation and error handling.

---

## 3) Phone OTP request form (validation + rate-limits)
**API:** `POST /auth/login/otp/request`  
**UI/UX:** E.164 input with country selector; disabled button during request; success hint (“Code sent”).  
**Validation:** E.164 regex; disabled if invalid.  
**Acceptance (Unit):**
- Form invalid blocks submit.
- Service posts phone; success state shown.
- 429 response shows cooldown and disables button.
**Playwright:**
- Enter invalid phone → validation message.
- Enter valid phone → “Code sent” message.
- Rate-limit response displays retry timer.

---

## 4) Phone OTP verification flow (token storage)
**API:** `POST /auth/login/otp/verify`  
**UI/UX:** 6-digit code input; paste support; “Resend” link with timer.  
**Acceptance (Unit):**
- Submits phone+code; tokens saved; redirect to dashboard.
- Wrong code shows inline error; does not store token.
**Playwright:**
- Valid code logs in and lands on dashboard.
- Resend disabled until timer completes.

---

## 5) Background token refresh
**API:** `POST /auth/refresh`  
**Implementation:** HttpInterceptor handles 401; queues requests; refresh once; retries.  
**Acceptance (Unit):**
- Interceptor requests refresh on expired access token.
- Retries original request after refresh; fails-through on refresh 401 (logs out).
**Playwright:**
- Expire token via mock; verify transparent retry then success.  
- If refresh fails, user is redirected to login.

---

## 6) Logout flow
**API:** `POST /auth/logout`  
**UI/UX:** Profile menu → Logout; confirm dialog.  
**Acceptance (Unit):**
- Calls logout endpoint; clears tokens/store; navigates to `/login`.
**Playwright:**
- Clicking Logout clears session; visiting `/wallet` redirects to login.

---

## 7) “My Profile” screen
**API:** `GET /users/me`  
**UI/UX:** Display name, email/phone, last login; edit triggers future endpoint (placeholder).  
**Acceptance (Unit):**
- Loads user from API; renders fields; shows skeleton while loading.
- Error shows retry button.
**Playwright:**
- Page displays correct user info; refresh works offline with cache (optional).

---

## 8) Memberships list
**API:** `GET /memberships/me`  
**UI/UX:** List with club name, status, role badges; responsive cards.  
**Acceptance (Unit):**
- Fetches memberships; displays status chips; empty state message.  
**Playwright:**
- Displays memberships; filters by status client-side.

---

## 9) Club list with search/filter
**API:** `GET /clubs`  
**UI/UX:** Search bar, pagination, sort; skeletons; infinite scroll optional.  
**Acceptance (Unit):**
- Debounced search triggers API with query params.
- Rendering of results; pagination works.
**Playwright:**
- Typing filters results; next page loads on scroll/button.

---

## 10) Club detail with venues
**API:** `GET /clubs/{clubId}`, `GET /clubs/{clubId}/venues`  
**UI/UX:** Header with name; tabs for Venues/Info.  
**Acceptance (Unit):**
- Concurrently loads club+venues; error recovery for either.
**Playwright:**
- Navigating to club detail shows venues list; back navigation preserves scroll.

---

## 11) Join club form
**API:** `POST /clubs/{clubId}/memberships`  
**UI/UX:** Select membership type; confirm dialog; success toast.  
**Acceptance (Unit):**
- Validates selection; posts; updates memberships cache.
**Playwright:**
- Join creates membership; user sees it in “My Memberships”.

---

## 12) Leave club flow
**API:** `DELETE /memberships/{membershipId}`  
**UI/UX:** Danger dialog; require typing “LEAVE” to confirm.  
**Acceptance (Unit):**
- Calls delete; removes item from UI; handles 403 properly.
**Playwright:**
- Leaving removes membership and shows toast.

---

## 13) Venues list (per club)
**API:** `GET /clubs/{clubId}/venues`  
**UI/UX:** Card grid; address; active chip.  
**Acceptance (Unit):**
- Renders only club venues; skeletons; empty state.  
**Playwright:**
- Mobile layout stacks cards; desktop uses grid.

---

## 14) Courts list (per venue)
**API:** `GET /venues/{venueId}/courts`  
**UI/UX:** Table with code/status; responsive collapse on mobile.  
**Acceptance (Unit):**
- Calls API; displays active/inactive correctly.
**Playwright:**
- Sorting by code works; columns adapt on small screens.

---

## 15) Session calendar/list with date filters
**API:** `GET /sessions?clubId=&from=&to=`  
**UI/UX:** Toggle calendar/list; date range picker; club filter.  
**Acceptance (Unit):**
- Builds query params correctly; handles timezone; renders items.  
**Playwright:**
- Changing dates updates results; bookmarking preserves filters.

---

## 16) Session creation form
**API:** `POST /sessions`  
**UI/UX:** Form with club, venue, time; validation; success redirect to detail.  
**Acceptance (Unit):**
- Required fields enforced; posting creates session; error states shown.
**Playwright:**
- Filling valid data creates session; user lands on details page.

---

## 17) RSVP buttons with instant update
**API:** `POST /sessions/{sessionId}/rsvp`  
**UI/UX:** Yes/No/Maybe buttons with optimistic update and undo.  
**Acceptance (Unit):**
- Sends correct status; on error, rolls back UI.
**Playwright:**
- Clicking “Yes” updates status; page refresh keeps state.

---

## 18) Session detail with RSVP list
**API:** `GET /sessions/{sessionId}`, `GET /sessions/{sessionId}/rsvps`  
**UI/UX:** Header, time, venue; attendees list with avatars; organizer badge.  
**Acceptance (Unit):**
- Parallel loads detail+RSVPs; merges into view model.
**Playwright:**
- RSVPs update after another browser submits (polling/websocket mock).

---

## 19) Check-in button
**API:** `POST /sessions/{sessionId}/rsvp` with `status=checked_in`  
**UI/UX:** Check-in button appears only if RSVP=Yes and within time window.  
**Acceptance (Unit):**
- Button visibility logic; posts checked_in; displays chip.  
**Playwright:**
- Tapping “Check-in” marks user and disables button.

---

## 20) Wallet balance
**API:** `GET /wallet`  
**UI/UX:** Balance card with last updated; pull-to-refresh (mobile).  
**Acceptance (Unit):**
- Renders balance; refresh calls API; handles errors.
**Playwright:**
- Balance updates after transaction is posted (mock).

---

## 21) Wallet transactions list
**API:** `GET /wallet/transactions`  
**UI/UX:** Infinite scroll or paginator; type chips; date grouping.  
**Acceptance (Unit):**
- Fetches pages; appends results; shows empty state.
**Playwright:**
- Scrolling loads next page; filters by type.

---

## 22) Wallet top-up with Stripe (UI placeholder)
**API:** `POST /wallet/topup/checkout` (to add)  
**UI/UX:** Amount input; redirect to Stripe; returns to success screen.  
**Acceptance (Unit):**
- Validates amount; posts; navigates to Stripe URL.  
**Playwright:**
- Completing mocked checkout returns to app with confirmation.

---

## 23) Club transactions list
**API:** `GET /clubs/{clubId}/transactions`  
**UI/UX:** Filters by date/category; export CSV.  
**Acceptance (Unit):**
- Fetches transactions; applies filters; CSV includes visible columns.
**Playwright:**
- Filter reduces list; export creates downloadable file.

---

## 24) Add club transaction form
**API:** `POST /clubs/{clubId}/transactions`  
**UI/UX:** Direction toggle; amount, category, notes; validation.  
**Acceptance (Unit):**
- Posts valid payload; updates list; handles 403.
**Playwright:**
- Submitting valid form shows success and new row.

---

## 25) Tournaments list
**API:** `GET /tournaments?clubId=&status=`  
**UI/UX:** Cards with status; filters; responsive grid.  
**Acceptance (Unit):**
- Correct query params; renders results; empty state.
**Playwright:**
- Status filter updates list; mobile layout stacks cards.

---

## 26) Tournament detail with divisions
**API:** `GET /tournaments/{tournamentId}`, `GET /tournaments/{tournamentId}/divisions`  
**UI/UX:** Tabs: Overview, Divisions, Rules.  
**Acceptance (Unit):**
- Loads both endpoints; errors handled separately.
**Playwright:**
- Divisions tab shows list; rules link opens in new tab.

---

## 27) Division registration form
**API:** `POST /divisions/{divisionId}/registrations`  
**UI/UX:** Team selection; agree to rules; submit; success redirect to payment.  
**Acceptance (Unit):**
- Validates selection; posts; receives registration id.
**Playwright:**
- Submitting creates registration; navigates to payment step.

---

## 28) Stripe checkout redirect (registration)
**API:** `POST /registrations/{registrationId}/checkout`  
**UI/UX:** “Pay Now” button; disable while posting; redirect to Stripe.  
**Acceptance (Unit):**
- Posts and receives `url`; sets `window.location` to Stripe URL.
**Playwright:**
- Button redirects; on API error, shows toast and remains on page.

---

## 29) Brackets view
**API:** `GET /divisions/{divisionId}/brackets`  
**UI/UX:** Visual bracket; collapsible rounds; RR table/SE tree.  
**Acceptance (Unit):**
- Renders structure for rr/se/de/americano; placeholder for unknown.
**Playwright:**
- Switching rounds updates matches; mobile allows horizontal scroll.

---

## 30) Match detail with live score
**API:** `GET /matches/{matchId}`  
**UI/UX:** Set scores; status; refresh button; optional live updates.  
**Acceptance (Unit):**
- Loads match and sets; updates on refresh.
**Playwright:**
- Live update mock changes displayed score within 2s.

---

## 31) Match result submission (admin)
**API:** `PATCH /matches/{matchId}`  
**UI/UX:** Form with set inputs; validation; winner preview.  
**Acceptance (Unit):**
- Validates integer scores; posts; UI shows updated match.
**Playwright:**
- Changing scores updates winner; save persists result.

---

## 32) Responsive navigation
**API:** N/A  
**UI/UX:** AppShell with toolbar, side nav (desktop), bottom nav (mobile).  
**Acceptance (Unit):**
- Navigation component renders correct links based on auth.
**Playwright:**
- Resizing window switches between drawer and bottom nav.

---

## 33) Error/success toasts
**API:** N/A  
**Implementation:** Central SnackbarService; interceptor pushes errors; success on create/update.  
**Acceptance (Unit):**
- Service shows messages; interceptor triggers on 4xx/5xx.
**Playwright:**
- Error toast appears after forced 500; success after valid post.

---

## 34) Skeleton loaders
**API:** N/A  
**UI/UX:** Reusable SkeletonList and SkeletonCard components.  
**Acceptance (Unit):**
- Components render placeholders when `loading=true`.
**Playwright:**
- Loading state visible then replaced by content upon fetch completion.

---

## 35) Offline mode with local caching
**API:** Cache: `/clubs`, `/sessions`, `/tournaments`, `/wallet`, `/venues`, `/divisions`  
**Implementation:** IndexedDB via `idb-keyval` (or Angular PWA); background sync.  
**Acceptance (Unit):**
- Cache layer fetches from network then caches; fallback to cache offline.
**Playwright:**
- Simulate offline: previously visited lists still render; badge shows “Offline”.

---

## Shared Angular Artifacts

### Services
- `AuthService`, `UserService`, `ClubService`, `VenueService`, `CourtService`, `SessionService`, `RsvpService`, `WalletService`, `ClubFinanceService`, `TournamentService`, `DivisionService`, `RegistrationService`, `MatchService`.

### Interceptors
- `AuthInterceptor` (attach JWT, refresh on 401), `ErrorInterceptor` (ProblemDetails → toast), `RetryInterceptor` (idempotent GET retry with backoff).

### Guards/Resolvers
- `AuthGuard`, `RoleGuard` (feature-specific flags), `SessionResolver`, `TournamentResolver`.

### Reusable Components
- `AppShell`, `SkeletonList`, `SkeletonCard`, `EmptyState`, `ConfirmDialog`, `PhoneInput`, `OtpInput`, `Paginator`, `BracketView`.

---

## Example Unit Test Snippets

```ts
// RSVP optimistic update
it('updates RSVP optimistically and rolls back on error', fakeAsync(() => {
  service.setRsvp(sessionId, 'yes').subscribe();
  expect(store.rsvp(sessionId)).toBe('yes');
  httpMock.expectOne(`/sessions/${sessionId}/rsvp`).flush({}, { status: 500, statusText: 'Server Error' });
  tick();
  expect(store.rsvp(sessionId)).not.toBe('yes');
}));
```

```ts
// Auth interceptor refresh flow
it('refreshes token on 401 and retries request', fakeAsync(() => {
  http.get('/wallet').subscribe();
  const first = httpMock.expectOne('/wallet');
  first.flush({}, { status: 401, statusText: 'Unauthorized' });

  const refresh = httpMock.expectOne('/auth/refresh');
  refresh.flush({ tokens: { accessToken: 'new', refreshToken: 'newr' } });

  const retry = httpMock.expectOne('/wallet');
  retry.flush({ balance: 10 });

  tick();
  expect(snackbar.lastShown).toBeUndefined();
}));
```

```ts
// Playwright example (pseudo)
test('user can join a club', async ({ page }) => {
  await page.goto('/clubs');
  await page.getByPlaceholder('Search clubs').fill('UN Pickleball');
  await page.getByRole('link', { name: 'UN Pickleball' }).click();
  await page.getByRole('button', { name: 'Join Club' }).click();
  await page.getByLabel('Membership Type').selectOption('regular');
  await page.getByRole('button', { name: 'Confirm' }).click();
  await expect(page.getByText('Membership created')).toBeVisible();
});
```

---

## Definition of Done (Frontend)
- ✅ Angular components/services/guards/interceptors implemented with strict typing
- ✅ Responsive UI verified at 360px, 768px, 1024px, 1440px breakpoints
- ✅ Unit tests for components/services with HttpTestingController and ComponentHarness (Material) as applicable
- ✅ Playwright e2e covering happy path and at least one failure path per feature
- ✅ Accessibility checks (axe-core in CI or Lighthouse)
- ✅ API contracts aligned with `club-api.yaml`; error states handled
- ✅ Loading/error/empty states covered; analytics events sent (optional)