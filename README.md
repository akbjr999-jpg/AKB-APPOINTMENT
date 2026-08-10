# AKB Retouch — Live Appointment Booking

This package keeps the existing visual design and connects the booking system to Supabase Postgres, Supabase Auth, a Supabase Edge Function, and Resend for automatic email notifications.

## 1. Create a Supabase project

Create a project at the official Supabase dashboard. Then open **SQL Editor** and run:

`supabase/schema.sql`

The SQL creates the `bookings` and `admin_users` tables, the unique date/time constraint, and Row Level Security policies. Public browsers do not receive direct access to customer booking rows; the Edge Function creates bookings server-side.

## 2. Create the admin account

In Supabase Dashboard → Authentication → Users, create an email/password user for the site owner.

Copy that user's UUID. Then run:

```sql
insert into public.admin_users (user_id)
values ('PASTE_ADMIN_USER_UUID_HERE');
```

The admin login in the website uses this Supabase Auth account.

## 3. Configure Resend

Create a Resend API key and verify the sending domain you intend to use. For production, set `RESEND_FROM_EMAIL` to an address on that verified domain, for example:

`AKB <appointments@yourdomain.com>`

## 4. Configure Edge Function secrets

Set these Supabase Edge Function secrets:

```text
SUPABASE_SERVICE_ROLE_KEY=YOUR_SERVER_ONLY_SERVICE_ROLE_KEY
RESEND_API_KEY=YOUR_RESEND_API_KEY
RESEND_FROM_EMAIL=AKB <appointments@yourdomain.com>
OWNER_EMAIL=akbjr999@gmail.com
```

NEVER put `SUPABASE_SERVICE_ROLE_KEY` or `RESEND_API_KEY` into `index.html`.

## 5. Deploy the Edge Function

Deploy `supabase/functions/booking-api/index.ts` as the public booking endpoint. Because visitors are not signed in when they make an appointment, deploy this particular function with JWT verification disabled; the function performs its own strict validation and uses only server-side secrets for database writes and email.

Using the Supabase CLI:

```bash
supabase functions deploy booking-api --no-verify-jwt
```

Or deploy the function from the Supabase Dashboard's Edge Functions editor and configure the function accordingly.

The function supports:


- `availability` — returns already-booked times for a date without exposing customer details.
- `create` — validates the date/time, prevents duplicate bookings using the database unique constraint, saves the booking, and sends owner + customer emails.

## 6. Configure the frontend

Open `index.html` and replace:

```js
const SUPABASE_URL = "YOUR_SUPABASE_PROJECT_URL";
const SUPABASE_PUBLISHABLE_KEY = "YOUR_SUPABASE_PUBLISHABLE_KEY";
```

with your Supabase Project URL and **publishable** key from Supabase Settings → API Keys.

Only the publishable/anon key belongs in the browser. Never use the service-role/secret key in this file.

## 7. Host the site

The frontend is a static HTML site. Upload `index.html` to any static hosting provider that supports HTTPS.

## 8. Admin access

On the booking page, tap the `Retouch .` text five times quickly to open Admin Access. Sign in with the Supabase Auth email/password account that was added to `admin_users`.

## 9. Important behavior

- Monday–Friday schedule is preserved exactly as displayed.
- Past dates and weekends are disabled.
- A date/time can only be confirmed once.
- The database unique constraint protects against simultaneous double-booking.
- Customer booking rows are not publicly readable.
- Admin reads/updates are protected by Supabase Auth + RLS.
- Email failures do not create duplicate bookings; the booking remains stored with `email_status = failed` for admin visibility.
