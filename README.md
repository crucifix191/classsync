# ClassSync

**Smart Spaces. Seamless Scheduling.**

Development of a Real-Time Classroom Availability and Reservation System.

ClassSync uses vanilla JavaScript ES modules, Vite, modern CSS, Lucide, Supabase Auth, PostgreSQL and Realtime. No frontend framework, mock database, service-role key, or client-controlled role assignment is used.

## Run locally

Requires Node.js 22.12+ (verified with Node 24).

```sh
cp .env.example .env
# Set VITE_SUPABASE_URL and VITE_SUPABASE_PUBLISHABLE_KEY
npm install
npm run dev
npm test
npm run build
npm run preview
```

The supplied workspace `.env` is already connected to the dedicated ClassSync project. It is ignored by Git. For reproducible installation use `pnpm@11.25.0 install --frozen-lockfile` with the committed lockfile. The GitHub workflow includes only the intended public browser configuration; its publishable key is not a secret.

## Project structure

```
src/pages/          Landing, auth, rooms, workspace and administration
src/components/     Shared layout, icons, dialogs, forms and feedback
src/lib/            Supabase client, data access, timezone helpers, Realtime lifecycle
src/styles/         Responsive design system and light/dark themes
supabase/migrations/ Ordered database schema and fixes
supabase/tests/      Transactional security acceptance suite
supabase/seed.sql    Clearly labeled SAMPLE classrooms
tests/              Node unit tests and live SDK test harness
.github/workflows/  GitHub Pages build, test and deployment
```

## Supabase setup

The dedicated project is **ClassSync**, reference `kntdzpaafgehxyyerbzn`, in Kaelix / Singapore. Its database was provisioned during development. Do not rerun the initial migration on an already-provisioned database.

For a fresh project, run every SQL file in `supabase/migrations` in filename order using the SQL Editor or Supabase CLI migrations. Then optionally run `supabase/seed.sql`. Sample inventory is explicitly named `SAMPLE-*`; replace it with verified campus rooms through administrator room management. Room photographs are illustrative Unsplash images, not photographs of the listed campus rooms.

The schema includes profiles, classrooms, booking policy, reservations, maintenance blocks, notifications, reservation audit and privacy-preserving schedule invalidation. Every exposed table has RLS. All table grants and RPC EXECUTE grants are explicit. The migration adds the subscribed tables to `supabase_realtime` when that publication exists. In a custom installation create `supabase_realtime` first or manually add `profiles`, `classrooms`, `booking_policy`, `reservations`, `schedule_versions`, and `notifications` in Database → Publications.

### Auth settings

Enable Email sign-in and keep email confirmation enabled. Disable anonymous sign-ins. Set a minimum password length of 12, enable leaked-password protection where supported, and configure a production SMTP sender before a campus rollout. Supabase's default mail service has delivery/recipient/rate limits; sending real confirmation and recovery mail requires a working sender.

Site URL:

```
https://crucifix191.github.io/classsync/
```

Allowed redirect URLs:

```
https://crucifix191.github.io/classsync/
https://crucifix191.github.io/classsync/?recovery=1
http://127.0.0.1:5173/**
http://localhost:5173/**
```

The client uses PKCE. Open verification/recovery links in the same browser that initiated the flow. A missing/expired verifier requires requesting a new link in that browser. The password recovery screen updates the authenticated recovery session. No service-role key belongs in `.env`, Vite variables, JavaScript, or a Pages workflow.

### Trusted administrator bootstrap

1. Register the designated administrator email (`admin@gmail.com`) using ClassSync.
2. Verify the email using the inbox. All public signups are students regardless of submitted metadata.
3. Run `supabase/bootstrap-admin.sql` in the trusted SQL Editor. It refuses to promote an unverified or absent account.
4. Sign in again. Use **People & policies** to verify faculty accounts. The web UI cannot create another administrator; repeat a trusted SQL bootstrap for additional verified administrators.

## Security and scheduling

- `auth.users` insert trigger creates profiles with the literal role `student`; user metadata supplies a display name only.
- Members can update only their own `full_name` column. Roles are read from the current database row, not user-editable JWT metadata.
- Reservation writes are possible only through guarded RPCs. New requests are always pending. Review requires administrator access and a future pending request. Rejection requires a reason. Cancellation requires ownership (or administrator access), a future start, and pending/approved status.
- Approved reservations have a GiST exclusion constraint over `classroom_id` and `[starts_at, ends_at)`. Adjacent bookings coexist. Competing approvals cannot both succeed.
- Occupancy mutations acquire the same classroom row lock before validating either bookings or maintenance; triggers provide additional cross-table conflict checks. RPCs run in PostgreSQL's default READ COMMITTED isolation. Direct table writes are denied to application roles.
- Pending requests never occupy a room. Availability is always attached to an explicit interval. The schedule RPC exposes only room IDs, slot IDs, times and status; it does not expose owners or purposes.
- In-app notifications and immutable audit rows are generated transactionally. Normal users cannot read other users' reservations/notifications or any audit records.
- Privileged RPCs deliberately use SECURITY DEFINER, explicit empty search paths, authentication/role checks, and revoked PUBLIC/anon execution. Supabase's advisor flags seven authenticated-callable definer RPCs; this is intentional and reviewed, not an assertion of zero advisor warnings.
- All stored instants are `timestamptz`; date/time input is converted with the explicit `+08:00` offset. Display formatting always uses `Asia/Manila`, independent of the device timezone.

## Realtime behavior

The app listens to schedule invalidation, reservation, room, profile, policy and notification changes. It refetches after subscription/reconnection, when returning to the tab, when the network returns, and every 30 seconds as a fallback. Subscriptions and timers are removed on logout/navigation away. Refresh is deferred while a form field or modal is being edited. Directory/calendar/report selections and reservation status filters are preserved on refetch. Offline/error indicators avoid presenting stale data as a confirmed live result. Final availability is revalidated in the database at request/approval time.

## GitHub Pages

Target repository: `https://github.com/crucifix191/classsync`.

1. Push these project contents to the repository root on `main` (not an extra `classsync/` directory).
2. Repository **Settings → Pages → Source → GitHub Actions**.
3. The supplied workflow installs locked dependencies, runs tests, builds Vite, uploads `dist`, and deploys to Pages.
4. Visit `https://crucifix191.github.io/classsync/` after the Actions deployment succeeds.

`vite.config.js` sets the production base to `/classsync/`. Hash routes such as `/classsync/#/rooms` work after refresh without server rewrites. Auth callback URLs are calculated from Vite's base, so they retain the repository path. If renaming the repository or using a custom domain, change the build base, workflow configuration, and Supabase redirect URLs together.

References: [Vite deployment guide](https://vite.dev/guide/static-deploy.html#github-pages), [GitHub Pages workflows](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages), [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security), [Supabase Realtime](https://supabase.com/docs/guides/realtime/postgres-changes).

## Acceptance checklist

See `VERIFICATION.md` for observed results and remaining manual checks. Do not equate build success with production readiness.

- Register a student; attempt `role: admin` metadata; verify resulting role remains student.
- Confirm the email in the initiating browser, sign in, sign out, and complete password recovery.
- Bootstrap a verified admin; assign faculty; confirm students cannot access admin routes or invoke privileged RPC actions.
- Search/filter rooms by name, building, capacity, equipment and explicit interval.
- Submit a future request within policy; confirm pending status and notification.
- Reject with a reason; confirm it appears to the owner. Approve an eligible request; confirm schedule occupancy.
- Approve competing overlapping requests simultaneously from two sessions; exactly one succeeds. Approve adjacent requests; both succeed.
- Race maintenance creation against approval; exactly one succeeds. Remove maintenance and verify refetch.
- Cancel eligible own requests; reject cancellation of another user's or already-started request.
- Attempt direct REST changes to role, status, owner, audit and notification ownership; reject them.
- Open two sessions, change a reservation, and confirm the second updates. Disconnect/reconnect and confirm recovery.
- Verify room create/edit/deactivation, faculty assignment, policy validation, reports/CSV and audit history.
- Inspect keyboard navigation, dialog focus, light theme, 375px phone, tablet and desktop layouts.

`npm test` runs timezone/interval/policy tests. Run `supabase/tests/security.sql` as postgres in SQL Editor for a self-contained suite that rolls back its fixtures. `tests/live.mjs` is the SDK/concurrency/Realtime harness for dedicated disposable QA fixtures; it reads a QA password from stdin, never a service-role key. It is not run against real users by CI.

## Operational limits

This is a functional capstone MVP with a real backend. It still needs real campus inventory, verified account ownership, production SMTP, backups/retention policy, monitoring, accessibility testing with assistive technology, and expected-load testing before a real campus rollout. Reports use current policy operating hours as their denominator and do not subtract maintenance; changing policy changes that comparison. Audit UI shows the latest 200 actions. Member lists and booking datasets paginate through the Data API rather than silently truncating at 1,000 rows; interactive retrieval stops with an error above 100,000 rows.
