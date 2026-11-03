# Smash & Volley: court booking (Supabase version)

`index.html` is the whole site: one file, no build step. It talks to the Supabase project **pickleball-booking** (`hzgsdpemrzrxwtuiodbq`).

## What the database guarantees
- A court cannot be booked twice for overlapping times. Postgres refuses the second booking itself, so this holds for the website, the front desk and any future outside source.
- Players cannot write bookings directly. They call `create_booking`, which checks opening hours, sport, length, the 14-day window and the 4-upcoming-bookings limit, and sets the price on the server.
- Staff use `admin_create_booking` and can block courts for maintenance. Blocks and bookings can never overlap either.
- Cancelling goes through `cancel_booking` (players until 2 hours before start, staff any time).
- Every create and cancel is recorded in `booking_events`.
- Users cannot make themselves admin.

## To go live
1. **Host it.** Upload this folder to any static host (Netlify, Vercel, Cloudflare Pages, or GitHub Pages).
2. **Allow your site address for sign-in.** Supabase dashboard > Authentication > URL Configuration: set Site URL to your live address and add it under Redirect URLs.
3. **Set up email sending.** Supabase's built-in email is limited to a few messages an hour. Before launch add your own SMTP provider (Authentication > Emails > SMTP settings), or sign-in links will fail for real players.
4. **Create your admin login (no email needed).** In Supabase open Authentication > Users > Add user > Create new user. Enter your email and a password, and tick "Auto Confirm User". Then, in the SQL editor, run:
   `update public.profiles set is_admin = true where id = (select id from auth.users where email = 'YOUR@EMAIL');`
   Sign in on the site with that email and password and the Master schedule tab appears. Other staff: they create an account on the site (Sign in > Create an account), then run the same command for their email.
5. **Check the numbers.** Sign in as admin and open the **Settings** tab. There you set the time zone, peak start time, longest booking, upcoming-bookings limit, cancellation cutoff, days bookable ahead, the four hourly rates (pickleball and badminton, off-peak and peak), and each court's name, hours and on/off. Changes apply to new bookings straight away; existing bookings keep their price.

## Signing in
Players can sign in with email and password, create an account, reset a forgotten password, or ask for an emailed sign-in link. Reset and link emails go through Supabase's built-in email, which only delivers to your Supabase team's addresses. Connect your own email service (Authentication > Emails > SMTP settings) before real players use them. Google sign-in can be added later.

## Online payment (Razorpay, off until you add keys)
Players pay the full price when they book. The slot is held for 10 minutes (adjustable in Settings) while they pay, then released if they don't. Cancelling at least the cutoff number of hours before the start turns the full amount into **credit** on the player's profile, not money back. Credit is used automatically on their next booking. After the cutoff the booking can't be cancelled. Staff cancellations also give credit.

Setup, about 15 minutes:
1. Create a free account at razorpay.com and, in test mode, open Account & Settings > API keys > Generate key. You get a Key ID and a Key Secret.
2. In Supabase open Edge Functions > Secrets and add `RAZORPAY_KEY_ID` and `RAZORPAY_KEY_SECRET`.
3. In Razorpay open Webhooks > Add new webhook. URL: `https://hzgsdpemrzrxwtuiodbq.supabase.co/functions/v1/razorpay-webhook`. Tick `payment.captured` and `order.paid`. Choose a secret, then add it in Supabase as `RAZORPAY_WEBHOOK_SECRET`.
4. In the site open Settings > Club, tick "Take payment online when players book", set Currency to INR and Time zone to Asia/Kolkata, and enter your rates in rupees. Save.
5. Test with Razorpay's test cards or test UPI. When happy, complete Razorpay's KYC, swap in the live keys and webhook, and test once with a small real booking.

Notes: credit is stored in the club's currency and isn't converted if you change currency. The webhook is the safety net, so a booking is confirmed even if the player closes the page after paying. The only money refund left is a payment that arrives after the hold expired; it is refunded automatically. If that refund fails, refund it from the Razorpay dashboard.

## Automatic court assignment
By default players pick a date, sport and hour, not a court. The database picks the court inside the same step that creates the booking, so two players can never get the same court. It scores each free court: a same-sport booking touching the hour before or after scores highest, a court already used for that sport that day scores next, a different sport touching the hour scores lowest, and it avoids leaving a lone unsellable hour. Ties go to the lower court number. A 2-hour booking always stays on one court, and confirmed bookings are never moved. The confirmation tells the player their court. In Settings > Club, "Courts are assigned" switches to "Player picks the court", which brings back the court-by-court grid (and the Booked slots icons below). On the Master schedule a purple ⇄ marks a booking that follows a different sport on the same court, so staff can switch the net in time. **Changeover buffer:** the system always prefers a court that needs no sport change. If the only free court has a different sport touching the hour before (or after), the new booking starts 15 minutes later (or ends 15 minutes earlier) and is priced pro rata, for example 45/60 of that hour's rate. The booker sees this in the booking panel, with the new times and price, before confirming. The earlier booking is never changed, and staff-made bookings stay whole hours.

## Booked slots on the player grid
In Settings > Booked slots an admin picks what players see on a booked square: just "Taken", a racket or paddle, a ball or shuttle, or the paired icons (paddle and ball for pickleball, racket and shuttle for badminton), with or without the sport name. A live preview shows the choice. Slots held while a player is paying show with a dashed border and "Held". Icons are from Google's Material Symbols and Material Design Icons (Apache 2.0).

## Profiles, credit and no-shows
- **Profile** (signed-in players): edit name and phone, see the credit balance and its history.
- **Credit** comes from timely cancellations, no-shows (a percentage you set in Settings, default 100%), or an admin adding it. It is spent automatically at booking, and a booking fully covered by credit needs no payment.
- **Check-in:** staff click a booking on the Master schedule, confirm the person is who booked (reference or photo ID), enter how many guests came with them, and press **Check in**. It opens 30 minutes before the start. Checked-in bookings show a tick, bookings 15+ minutes late with no check-in get an orange outline, and a checked-in booking can't be marked a no-show (undo the check-in first).
- **No-shows:** on the Master schedule, click a booking that has started and press **Mark no-show**. The player gets the credit percentage, the no-show is recorded against them, and the slot reopens so staff can book walk-ins for the rest of the hour. (Online booking still starts at the next full hour.)
- **Customers** (admins): see every customer's credit and no-show count, and add or remove credit with a reason. The database never lets credit go below zero.

## Not built yet
- Confirmation emails.
- Playtomic and CourtReserve: they would send bookings in through a Supabase Edge Function that uses the `source` and `external_id` columns, so repeated deliveries are ignored.


## Changeover between sports (setting)

Settings > Club settings > **Changeover between sports**: Off, 5, 10, 15 or 20 minutes (default 15).
It only applies with automatic court assignment, when two different sports are back to back on the same court and no other court works.
The player who books second gets the shorter session and a prorated price. Off removes the buffer entirely.
Database column: `club_settings.changeover_minutes` (0 = off).


## Cancellation, credit and no-show rules

Settings > Club settings controls the three times below. Defaults shown.

| When | What happens |
|---|---|
| Player cancels **more than 1 hour** before the start (setting: "Full refund if cancelled more than") | The money is refunded to the original payment method through Razorpay. Any credit the player had used goes back to their credit. The slot reopens. |
| Player cancels **inside that hour but more than 15 minutes** before the start | Everything paid becomes credit on their profile (rupees). The slot reopens. |
| **Within 15 minutes** of the start (setting: "Players cannot cancel within") | Players cannot cancel online. Staff still can. |
| Staff cancel a booking at any time | Treated like an early cancel: money refunded. |
| **No-show**: group not checked in | Staff only, and only **20 minutes after the start** (setting: "Staff can mark a no-show after"). Nothing happens automatically. The player gets credit (percentage in "Credit given for a no-show", default 100%), and the slot reopens. |

Reopened no-show slot: with automatic court assignment, the rest of that hour can be booked online (one hour only, starts at the next quarter hour, at least 20 minutes left, price prorated). Staff can also book it as a walk-in.

If a refund cannot be sent right away (for example the Razorpay keys are not added yet), the booking shows "Refund on its way" and the site retries it automatically the next time the player opens My bookings.
Database: `club_settings.late_cancel_lock_minutes`, `club_settings.no_show_wait_minutes`; function `reopened_now(sport)`.
