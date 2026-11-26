# Cipher of the Day

A daily cryptogram game. Everyone gets the same encrypted quote each day, cracks it one letter at a time (or gambles on the whole phrase), and builds a streak. Players sign in with their own email and password, and their stats are stored in your Supabase database.

It is a static site: `index.html` + `config.js`. No build step, no server to run.

## Try it now (guest mode)

Open `index.html` in a browser. Until you add Supabase keys the game runs in **offline mode**: you can play as a guest, or create an account and sign in to see your stats. Offline accounts live in that browser only (passwords are hashed with PBKDF2), so they are for trying the game out. Real accounts that work on any device, and the player standings on the results screen, need Supabase (below).

## Set up real accounts (about 10 minutes)

1. Create a free project at https://supabase.com.
2. In the dashboard open **SQL Editor > New query**, paste all of `supabase/schema.sql`, and press **Run**. This creates the tables, the security rules and the server-side score check. Then open a **new query**, paste all of `supabase/puzzles.sql` (the answer key for the first 1,100 days, about three years) and Run it. If you skip `puzzles.sql` the game still works, but scores fall back to being browser-reported.
3. Open **Project Settings > API**. Copy the **Project URL** and the **anon / publishable** key into `config.js`. Never use the `service_role` key in the browser.
4. Open **Authentication > Providers** and make sure **Email** is enabled. Leave "Confirm email" on, so people must verify their address.
5. Open **Authentication > URL Configuration**. Set **Site URL** to the address where you host the game, and add that same address under **Redirect URLs**. Do this again with your real domain after you deploy (and keep `http://localhost:8000` for local testing). Confirmation and password-reset emails send people back to this address.
6. Optional but recommended under **Authentication > Sign In / Providers**: raise the minimum password length to 8 to match the game, and turn on leaked-password protection if your plan offers it.

### Test locally

Browsers block some features on `file://` pages, so serve the folder:

```
cd cipher-of-the-day
python3 -m http.server 8000
```

Then open http://localhost:8000.

## Deploy

Any static host works. Upload the folder (without `supabase/` if you like) to one of:

- **Netlify Drop**: drag the folder onto https://app.netlify.com/drop.
- **Vercel**: `vercel` in the folder, accept the defaults.
- **GitHub Pages**: push to a repo and enable Pages.
- **Cloudflare Pages**: connect the repo, no build command, output directory `/`.

After deploying, go back to step 5 and add your live address.

## Optional: Google sign-in

1. In Google Cloud Console create an OAuth client (type: Web application). Add the redirect URL shown in Supabase under **Authentication > Providers > Google**.
2. Paste the client ID and secret into that Supabase provider page and enable it.
3. Set `GOOGLE: true` in `config.js`. A "Continue with Google" button appears.

## How the data is stored

| Table | One row per | Holds |
|---|---|---|
| `results` | player per day | outcome, score, rank, mistakes |
| `progress` | player | the game in progress, so play continues on another device |

Every stat on the profile screen (current streak, longest streak, wins, win rate, rank counts) is computed from `results`, so there is no separate stats row that can drift out of sync. Row Level Security means a signed-in player can only read and write their own rows. Results can be added but never edited or deleted, and only for today's puzzle (plus or minus one day for time zones).

### Standings

The results screen shows where the player placed among everyone who played the same puzzle ("#12 of 48 players, top 25%") plus how many players got each clearance rank. This comes from the `day_standing` function in `schema.sql`, which returns only totals. If you set up the database before this feature existed, run the new block at the bottom of `schema.sql` once.

## Score checking

The browser records every move a player makes and sends the list to the `submit_result` database function. The server replays those moves against its own copy of the puzzle and stores **its own** score, rank and mistake count, so editing a score in the browser changes nothing. Impossible move lists (moves after the end, a repeated guess, an unfinished game, a wrong day) are rejected. The answer key lives in a table players cannot read.

**Honest limit:** the quotes are still visible in `index.html`, so someone who reads the page source can submit a *genuine-looking perfect run*. The server stops invented scores, not perfect play. A fully cheat-proof game would keep the answers on the server and check each guess there, which is a redesign. This is fine for a streak game; think twice before offering prizes.

**Lock it down (optional, after you have tested):** once you have finished a day with a real account and seen your standing appear, run `supabase/lockdown.sql`. It removes the players' direct write access to `results`, so `submit_result` becomes the only way a score can be stored.

## Before you go live: checklist

1. `schema.sql` and `puzzles.sql` both ran without errors in the SQL Editor.
2. `config.js` has your Project URL and publishable key (never the secret key).
3. Auth: Email provider on, Confirm email on, Site URL and Redirect URLs set to your real domain.
4. Play one full game on the deployed site with a real account, then open the profile and check the result appears. In the dashboard, **Table Editor > results** should show your row.
5. Run `supabase/lockdown.sql`.
6. Test **Delete account** with a throwaway account (profile screen, tap twice). The user should disappear under **Authentication > Users**.
7. Make sure you have more than a few days of quotes ahead: the game repeats its 13 quotes in order. See "Adding quotes".

## Things to know before launch

- **Changing the launch date.** The puzzle number is counted from `LAUNCH` in `index.html`. The same date lives in the `game_config` table (`launch` row) in the database. To change it: edit `LAUNCH`, run `update public.game_config set value = 'YYYY-MM-DD' where key = 'launch';`, then regenerate and re-run `puzzles.sql`.
- **Adding quotes.** Add new quotes at the end of the `QUOTES` array in `index.html` (so past days do not change), run `node scripts/gen-puzzles.js` to rebuild `supabase/puzzles.sql`, and run that file in the SQL Editor again. If the two ever disagree, the server rejects the score (23514) and the player's result is kept only on their device.
- **Account deletion.** Players can delete their own account from the profile screen (it calls the `delete_my_account` function, which removes the login and every row). Offline accounts have no such button; clearing the site data removes them.
