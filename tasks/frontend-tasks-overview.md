# Frontend Tasks and API Mappings

## Auth & Identity
1. Google login button with JWT retrieval and auto-session creation  
   **API:** `POST /auth/login/google`
2. Apple login button with JWT retrieval and auto-session creation  
   **API:** `POST /auth/login/apple`
3. Phone OTP request form with validation and rate-limit feedback  
   **API:** `POST /auth/login/otp/request`
4. Phone OTP verification flow with automatic token storage  
   **API:** `POST /auth/login/otp/verify`
5. Secure token refresh handling in background  
   **API:** `POST /auth/refresh`
6. Logout with token/session invalidation and redirect  
   **API:** `POST /auth/logout`

## User Profile
7. “My Profile” screen with editable personal details  
   **API:** `GET /users/me`
8. Display membership list with club name and status  
   **API:** `GET /memberships/me`

## Clubs & Memberships
9. Club list page with search and filters  
   **API:** `GET /clubs`
10. Club detail view with venue list  
    **API:** `GET /clubs/{clubId}`, `GET /clubs/{clubId}/venues`
11. “Join club” form with membership type selection  
    **API:** `POST /clubs/{clubId}/memberships`
12. Leave club confirmation flow  
    **API:** `DELETE /memberships/{membershipId}`

## Venues & Courts
13. Display venues in a club with address and active status  
    **API:** `GET /clubs/{clubId}/venues`
14. Display courts list in a venue with active/inactive tag  
    **API:** `GET /venues/{venueId}/courts`

## Sessions & RSVP
15. Session calendar/list with date filters  
    **API:** `GET /sessions?clubId=&from=&to=`
16. Session creation form with venue and time selection  
    **API:** `POST /sessions`
17. RSVP buttons with instant status update  
    **API:** `POST /sessions/{sessionId}/rsvp`
18. Session detail view with RSVP list  
    **API:** `GET /sessions/{sessionId}`, `GET /sessions/{sessionId}/rsvps`
19. “Check-in” button for attending members  
    **API:** `POST /sessions/{sessionId}/rsvp` with `status=checked_in`

## Wallet & Transactions
20. Display wallet balance with refresh animation  
    **API:** `GET /wallet`
21. Wallet transaction list with pagination  
    **API:** `GET /wallet/transactions?page=&pageSize=`
22. Wallet top-up flow with Stripe  
    **API:** *(To be added, e.g., `POST /wallet/topup/checkout`)*

## Club Transactions
23. Display club transaction list with filter by category/date  
    **API:** `GET /clubs/{clubId}/transactions`
24. Add club transaction form with validation  
    **API:** `POST /clubs/{clubId}/transactions`

## Tournaments
25. Tournaments list with status and filter  
    **API:** `GET /tournaments?clubId=&status=`
26. Tournament detail view with divisions and rules  
    **API:** `GET /tournaments/{tournamentId}`, `GET /tournaments/{tournamentId}/divisions`
27. Division registration form with team/player selection  
    **API:** `POST /divisions/{divisionId}/registrations`
28. Stripe checkout redirect for registration payment  
    **API:** `POST /registrations/{registrationId}/checkout`
29. Display brackets visually with round/match navigation  
    **API:** `GET /divisions/{divisionId}/brackets`
30. Display match detail with live score update  
    **API:** `GET /matches/{matchId}`
31. Match result submission form for admins  
    **API:** `PATCH /matches/{matchId}`

## General UX
32. Responsive navigation bar with quick access to wallet, sessions, tournaments  
    **API:** *(No direct API — navigation links to multiple endpoints above)*
33. Error/success toast notifications for all actions  
    **API:** *(Generic, triggered by all API responses)*
34. Skeleton loaders for lists and detail pages  
    **API:** *(No direct API — tied to all list/detail endpoints)*
35. Offline mode with local caching for key data  
    **API:** Cache results from `/clubs`, `/sessions`, `/tournaments`, `/wallet`, `/venues`, `/divisions`