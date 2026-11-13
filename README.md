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
| **Within 15 minutes** of the start, but before it begins (setting: "Last-minute window") | The player can still cancel. **50%** of the price becomes credit (setting: "Credit for a last-minute cancel"). The slot reopens. Unpaid bookings simply cancel. Once the game has started, players cannot cancel online. |
| Staff cancel a booking at any time | Treated like an early cancel: money refunded. |
| **No-show**: group not checked in | Staff only, and only **10 minutes after the start** (setting: "Staff can mark a no-show after"). Nothing happens automatically. The player gets credit (percentage in "Credit given for a no-show", default 50%), and the slot reopens. |

Reopened no-show slot: with automatic court assignment, the rest of that hour can be booked online (one hour only, starts at the next quarter hour, at least 20 minutes left, price prorated). Staff can also book it as a walk-in.

If a refund cannot be sent right away (for example the Razorpay keys are not added yet), the booking shows "Refund on its way" and the site retries it automatically the next time the player opens My bookings.
Database: `club_settings.late_cancel_lock_minutes`, `club_settings.last_minute_credit_percent`, `club_settings.no_show_wait_minutes`; function `reopened_now(sport)`.


## Roles: User, Staff, Admin

| | User | Staff | Admin |
|---|---|---|---|
| Tabs | Book a court, My bookings, Profile | + Front desk, Master schedule | + Customers, Settings |
| Own bookings: book, cancel, view | yes | yes | yes |
| See all bookings on the schedule | | yes | yes |
| Check in, undo check-in, mark no-show (after the wait) | | yes | yes |
| Add phone / walk-up bookings | | yes | yes |
| Cancel a walk-up booking | | yes | yes |
| Cancel an online booking (refunds money) | | | yes |
| Lessons, leagues, maintenance blocks | | | yes |
| Customer list, add/remove credit | | | yes |
| Settings (rates, courts, time rules, payments) | | | yes |
| Change someone's role | | | yes |

These rules are enforced in the database (row-level security and the functions), not only by hiding tabs.

**Making someone staff or admin:** they sign up on the site first (they start as a User). An admin opens Customers, finds them and picks a role from the Role dropdown. It takes effect the next time they open the site. Admins cannot change their own role, and the last admin cannot be removed.

**Front desk tab:** today's bookings with search and filters. Check-in opens 30 minutes before the start. Tapping Check in asks for the number of guests, an optional "I checked photo ID" box and a note, then Mark present. A no-show can be marked once the waiting time (Settings) has passed. Unchecked-in bookings that are late are highlighted. The page refreshes itself every 30 seconds.

Database: `profiles.role` ('user', 'staff', 'admin'), `is_staff()`, `is_admin()`, `admin_set_role(user, role)`.


## Staff logins and desk PINs

**Staff logins are created by an admin, not by signup.** Settings tab > **Add a staff login**: a display name, a username (for example `frontdesk`) and a password. The account is created already confirmed, with the Staff role. Staff sign in on the normal sign-in box using the username. Typing a full email address instead creates a personal login.

- Staff have no way to change the password (no screen for it, and the reset-by-email link does not work for usernames). An admin sets a new one from the person's row in Customers ("Set password").
- Edge function: `staff-accounts` (actions `create` and `reset_password`, admins only). Usernames are stored as `username@staff.smashvolley.invalid`. If Supabase ever refuses that domain, create the login with a real email address instead.

**Desk PINs for a shared login.** Tick "Shared by several people" when creating the login (or tick "Shared login (PIN)" on its row later). Then:

1. Settings > **Desk staff and PINs**: add each person with a 4 to 6 digit PIN. Two active people cannot share a PIN. You can rename, change a PIN or switch a person off at any time.
2. On the shared login, Front desk and Master schedule show a lock screen. A person enters their PIN and the header shows "Desk: Name". They press **Lock**, or it locks by itself after 5 quiet minutes (the server also expires it after 10).
3. Five wrong PINs lock the screen for 2 minutes.
4. Every check-in, undo, no-show, phone booking and staff cancel is recorded with the name. The schedule panel shows "Booked by" and "Checked in ... by"; the Front desk list shows "Present · by Name".

The PIN is for accountability, not high security: someone who learns another person's PIN can act as them. Change a PIN or switch the person off if that happens. Admins and personal (non-shared) staff accounts are never asked for a PIN.

Database: tables `staff_pins`, `desk_sessions`, `pin_attempts` (not readable by the site, only through functions), `profiles.needs_pin`, functions `unlock_desk`, `lock_desk`, `admin_save_pin`, `admin_list_pins`, `admin_set_needs_pin`, `admin_shared_logins`. The action functions `check_in`, `undo_check_in`, `mark_no_show`, `admin_create_booking` and `cancel_booking` take an optional `p_token` and refuse shared logins that don't have a valid one.

## Cancel confirmation and test mode

Pressing **Cancel** on My bookings now opens a confirmation that says exactly what will happen: nothing to refund (unpaid, pay-at-the-club booking), a card refund, or credit (full price, or the last-minute percentage when the game starts in under the last-minute window). Pressing "Yes, cancel it" then cancels.

Admins also see a **Test only: pretend this was paid online** box in that confirmation. With it ticked, the text shows what a player would get if the booking had been paid through the gateway, and the button becomes "Show the result (no changes)". Nothing is cancelled and nothing is written. Use it to check the rules while payments are still off.

## Court view and appearance (Settings > Appearance)

**Court view.** On Book a court, below the "Any court" timeline, a plan of the courts shows which are free at the hour you last tapped on the timeline, including fully booked, passed or closed hours (or the current hour if none is tapped). The tapped hour is underlined in the timeline header. Free courts show the lines of the sport selected above (white pickleball lines, or thin light-blue badminton lines). A booked court keeps the lines of the game it is booked for, whatever sport is selected. When players choose their own court (Settings > Club), tapping a free court selects it. Turn the whole view on or off with "Show the court view" in Settings > Appearance.

**Website colour theme.** Settings > Appearance > Website colour theme: Forest green (default), Midnight and orange, Ocean blue, Rust and sand, Slate and gold, Plum and lilac, or a custom main colour. It applies to everyone, follows each person's light or dark mode, and previews live before you press Save.

**Court view colours.** Ten presets (including Blue court with orange kitchen on midnight, the default) or three custom colours (court, kitchen, surround), plus a switch for the club name in the kitchens, with its own font (8 choices), size (7 to 18 px) and letter colour. Very large text is squeezed to fit inside the court.

Database: `club_settings.show_court_view` (boolean) and `club_settings.appearance` (small JSON: theme, primary, court, cp, ck, co, wordmark).

## Settings tiles and Master schedule markers

All Settings tiles are the same size when closed (an open tile spans the full width). Desk staff and PINs and Add a staff login now live in Settings, since they are club configuration; the Customers tab keeps roles and credit. In the Master schedule, booked squares now keep the sport icon from Settings > Booked slots alongside the source code (for example WEB and BD).

## Future plans notepad

Settings > **Future plans** is a private notepad for ideas. Click the tile, type, and press **Save notes**. It is stored in the `admin_notes` table and only admins can read or write it. **Promotions** is an empty placeholder tile for now.

## Phone layout

The header menu used to be wider than a phone screen (staff and admin have many tabs), which made phones zoom the whole page out. The menu now wraps onto several lines on small screens and the page fits the phone width.

## Light and dark mode

The site is always light by default, so phones in dark mode look the same as the computer. Settings > Appearance > **Light or dark** can switch the whole club site to follow each person's device (dark on dark-mode phones).

## Phone menu

On phones, staff and admins see a single Menu button showing the current page. Tapping it opens the list of pages; picking one closes it. Players (three tabs) and larger screens keep the normal tab row.

## Credit: promo and paid (new)
Every player has two kinds of credit.
- **Credit (paid)**: from cancellations, no-show returns, or an admin adjustment. Never expires.
- **Promo credit**: given by an admin under Customers (amount, expiry date, reason). Each grant is its own line with its own expiry. It becomes zero at the end of the expiry day (club time) and is lost if not used.

**Order of use at booking:** promo (soonest expiry first), then paid credit, then the payment method. Credit is applied even while online payments are off; whatever is left shows as "pay X at the club" (front desk and schedule show "Collect X at the club").

**Cancel / no-show:** money you paid comes back as a card refund or credit; credit you used goes back to the bucket it came from. Promo credit returns with its original expiry; if it has already expired it is lost. For late and no-show returns (percentage rules), real money is counted first and promo last. Promo credit is never refunded to a card or transferred.

Database: `credit_lots`, `credit_uses`, `bookings.promo_applied`, functions `my_credit_split`, `admin_grant_promo`. Upload the new `index.html` to GitHub to use it.

## Equipment rental (new)
Players can add a rental pack when they book (off by default).
- **Large pack:** 4 paddles + 2 balls (pickleball) or 4 racquets + 2 shuttles (badminton). **Small pack:** 2 paddles + 1 ball or 2 racquets + 1 shuttle.
- **Settings → Equipment rental:** switch it on and set the two prices (per booking, not per hour).
- **Kept separate from the court price.** Credit, promo credit, online payment, refunds and the cancel/no-show percentages apply to the court only. The rental is always paid with money **at the club**; the confirmation and the front desk show "collect ₹X at the club".
- **Changing it later:** staff can add or remove a rental on any booking (including walk-ups) from the Master schedule panel or the Front desk list until the game ends. A player can remove their own rental under My bookings before the free-cancellation cutoff.
- **Cancel or no-show:** the rental is dropped with the booking and is never charged.
- Database: `club_settings.rental_enabled / rental_large_price / rental_small_price`, `bookings.rental / rental_price`, function `set_booking_rental`. The server sets the price from Settings, so a player cannot change it.
- Not tracked yet: a "collected" tick for the rental money.

### Rental at check-in, history and report (new)
- **Check-in step:** the check-in panel (Front desk and Master schedule) has a "Rental at check-in" choice preset to what was booked: No rental, Large pack or Small pack. Staff can change it (for example to a smaller pack, or none). If a pack is chosen, "Rental handed out and ₹X collected" must be ticked before they can check in. After check-in the rental is locked; Undo check-in unlocks it.
- **History:** every rental change is saved in `rental_events` (added, removed, changed, confirmed, unconfirmed) with the pack before and after, the price at the time, sport, peak or off-peak, who did it (player or staff name) and when. Admin read only.
- **Report:** Settings → Rental report (admin). Pick dates and Run: online bookings, share booked with a rental, rentals taken and revenue by sport, pack and peak, changes after booking, and counts by staff member. "Download history (CSV)" saves the event list. Revenue counts rentals confirmed at check-in only.
- **Rental report summary:** the top of the report answers one question for rentals booked in advance: at check-in, did people keep it, change the pack (smaller or larger), or cancel it? It shows counts and shares for checked-in bookings, plus rentals added at the desk with none booked, and booked value against money collected. "Show details" opens the full tables (taken by sport and pack, every change, by staff, bookings that were not checked in). "Download history (CSV)" saves every individual change.

## Header for staff and admins (new)
Staff and admins get a shorter tab bar and a name menu.
- **Players:** unchanged (Book a court, My bookings, Profile).
- **Staff:** Book a court, Front desk, Master schedule.
- **Admin:** the staff tabs plus Customers and Settings.
- **Name menu** (button showing "Name · Staff/Admin ▾"): My bookings, Profile, Sign out, and for admins "Preview as". It closes with Escape, a click elsewhere, or choosing a link. The desk "Lock" button stays beside it.
- **Phones:** the Menu button lists the same tabs, with My bookings and Profile at the end.

## What staff can and cannot do
- **Can:** use the Front desk (check in, undo, mark no-show), use the Master schedule to add phone and walk-up bookings (with an optional rental, paid at the counter), cancel phone and walk-up bookings, and add or remove a rental on any booking until check-in opens.
- **Cannot:** book a court through Book a court (the server refuses it: "Staff book courts for players on the Master schedule"; the page shows a pointer to the schedule instead of the booking form), cancel online bookings, create lessons, leagues or blocks, or open Customers, Settings or the Rental report.
- **Admins** can do all of the above, including booking through Book a court, and cancelling anything.
- Phone and walk-up bookings have no player account, no credit and no online payment: the court and any rental are paid at the counter.

## Cancellation record (new)
- **Reason required:** when staff or an admin cancel a booking that is not their own, the Master schedule panel asks "Why is it being cancelled?" (player called, booked by mistake or wrong time, moved to another time, duplicate, weather, court or club unavailable, or other with a note) plus an optional note. The server refuses the cancel without a reason, and "Other" needs a note.
- **Kept on the booking:** reason, note, who cancelled (name and role) and when. A player cancelling their own booking is recorded as "by player" with no reason needed. No-shows are recorded separately by the existing no-show flow.
- **Report:** Settings → Cancellation record (admin). Pick dates and Run: totals by staff, player and no-show, counts by reason, counts by person, and a list of cancelled bookings (game, court, booking, type, reason and note, who, when). "Download (CSV)" saves the full list.
- Not covered yet: removing a maintenance block or a lesson/league slot has no reason, and cancellations made automatically when a payment hold runs out are not listed with a reason.

## Booking sources

Settings → **Booking sources** (admin only) controls where bookings can come from. Each source has a name, a short code (e.g. HUD) and a colour. Code and colour are assigned automatically, and they show on the Master schedule and its legend.

- **Website** and **Front desk** are built in and cannot be changed.
- **Outside platform (needs a connection)**, e.g. Playtomic or Hudle: not connected yet, so an admin enters these bookings by hand. Price is 0 because they are paid on the platform ("Paid on Hudle"). Only an admin can cancel them.
- **Run by the club (no connection)**, e.g. lessons or leagues: added by an admin. Tick "Collect court price at the counter" to charge the court price at the counter; leave it off for no charge.
- Sources are switched off, never deleted. A source that already has bookings cannot change type.
- Staff only ever see Website, Front desk and sources that appear that day; only admins can add bookings for external or club-run sources.

To add Hudle: Settings → Booking sources → name "Hudle", type "Outside platform" → Add source.

## Settings layout (admin)

Admins have these tabs: Book a court, Front desk, Master schedule, Customers, **Settings**.

Settings has seven groups in a list on the left (a dropdown on phones). Each group shows its sections as plain headed blocks with fields in two columns, and each section has its own Save button. Edits stay if you switch groups before saving:

| Group | Contains |
|---|---|
| Club | Club (currency, time zone, how courts are assigned, changeover, online payment), Booking rules (longest booking, upcoming bookings per player, bookable days ahead, payment hold), Cancellations and no-shows |
| Courts and pricing | Courts, Rates (with the peak start time), Equipment rental, Promotions (placeholder) |
| Booking sources | Source list and Add source |
| Team | Desk staff and PINs, Add a staff login |
| Display | Appearance, Booked slots |
| Reports | Cancellation record, Rental report (date range and CSV) |
| Future plans | Private notepad |

The Rates Save button also saves the peak start time.
