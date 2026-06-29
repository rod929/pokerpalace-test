# Poker Palace — fixes for Claude Code

Two issues to resolve. **Issue 1 is mostly a database/config fix (SQL), Issue 2 is a code fix** in `admin_view_cloud_test.html`.

Repo: `rod929/pokerpalace-test` · Live: https://rod929.github.io/pokerpalace-test/
File to edit: `admin_view_cloud_test.html` (the "complete tournament report" page)

⚠️ **Validation discipline (do this before committing):** the browser runs `<script type="module">` (strict mode). `node --check` on a plain `.js` is lenient and misses errors. Extract the module script and check it:
```bash
python3 -c "import re; s=open('admin_view_cloud_test.html').read(); m=re.search(r'<script type=\"module\">(.*)</script>',s,re.S); open('/tmp/mod.mjs','w').write(m.group(1))"
node --check /tmp/mod.mjs
```

---

## ISSUE 1 — Club Marconi $15,000/month shown as one game's payment (wrong per-game P&L)

**Root cause:** the report ALREADY splits a fixed-monthly club fee correctly ($15k ÷ games that month, per game). So the venue "Club Marconi" is almost certainly configured as `per_game` with a $15,000 rate, instead of `fixed_monthly`.

### Fix A — set the venue config (SQL, run in Supabase SQL editor)
```sql
-- check current setting first
select name, club_money_basis, club_money_rate from venues where name ilike '%marconi%';

-- set it correctly (adjust the name match if needed)
update venues
set club_money_basis = 'fixed_monthly',
    club_money_rate  = 15000
where name ilike '%marconi%';
```
After this, every tournament report for Club Marconi will spread the $15k across that month's games automatically. No code change needed for this part.

### Fix B — (code, already prepared) make the fixed-monthly game count consistent
In `admin_view_cloud_test.html` there are THREE places that compute the fixed-monthly per-game split. Two excluded cash games from the divisor; one did not. Make them all exclude cash games so the divisor matches.

Find this block (around line 314, the main detail path):
```js
    } else if(basis==='fixed_monthly'){
      const ym=(c.game_date||'').slice(0,7);
      const gamesThatMonth=CO_DATA.filter(x=>x.venue_name===c.venue_name && (x.game_date||'').slice(0,7)===ym).length || 1;
      club=rate2/gamesThatMonth; clubLbl=' (monthly '+money(rate2)+' ÷ '+gamesThatMonth+' games)';
```
Replace with (adds `&& !(x.is_cash_game===true||x.game_name==='Rake Free Cash Game')` to the filter):
```js
    } else if(basis==='fixed_monthly'){
      const ym=(c.game_date||'').slice(0,7);
      const gamesThatMonth=CO_DATA.filter(x=>x.venue_name===c.venue_name && (x.game_date||'').slice(0,7)===ym && !(x.is_cash_game===true||x.game_name==='Rake Free Cash Game')).length || 1;
      club=rate2/gamesThatMonth; clubLbl=' (monthly '+money(rate2)+' ÷ '+gamesThatMonth+' games)';
```
(The other two fixed_monthly spots — near the day-total helper and `unifiedCashSection` — already exclude cash games; leave them.)

---

## ISSUE 2 — "Double Barrel Fridays" (closed out by Rob Jackson) not showing on the report

The report HIDES any close-out flagged as a cash game (`is_cash_game===true` or name `Rake Free Cash Game`) because cash games normally fold into the tournament card for the same venue+date. If Double Barrel Fridays was flagged as a cash game but has NO matching tournament that night, it became an orphan and vanished.

### Fix — show orphan cash games (code)
In `admin_view_cloud_test.html`, find (around line 166):
```js
      // helper: is this close-out the parallel rake-free cash game (saved as its own record)?
      const isCashCo=c=> c.is_cash_game===true || c.game_name==='Rake Free Cash Game';
      // build the visible list: hide standalone cash-game records (they fold into the tournament card for that venue+date)
      const visible=CO_DATA.filter(c=>!isCashCo(c));
      // map each visible close-out index back to its CO_DATA index, and find its paired cash game
      const findCash=(c)=> CO_DATA.find(x=>isCashCo(x) && x.venue_name===c.venue_name && x.game_date===c.game_date) || null;
      $('coList').innerHTML = visible.map((c)=>{
```
Replace with:
```js
      // helper: is this close-out the parallel rake-free cash game (saved as its own record)?
      const isCashCo=c=> c.is_cash_game===true || c.game_name==='Rake Free Cash Game';
      // a cash game folds into a tournament card only if a NON-cash close-out exists same venue+date.
      const hasTournamentSameNight=c=> CO_DATA.some(x=>!isCashCo(x) && x.venue_name===c.venue_name && x.game_date===c.game_date);
      // visible = all non-cash close-outs, PLUS any "orphan" cash games with no tournament that night
      // (otherwise a cash-flagged game with no paired tournament would vanish entirely).
      const visible=CO_DATA.filter(c=>!isCashCo(c) || !hasTournamentSameNight(c));
      // map each visible close-out index back to its CO_DATA index, and find its paired cash game
      const findCash=(c)=> isCashCo(c) ? null : (CO_DATA.find(x=>isCashCo(x) && x.venue_name===c.venue_name && x.game_date===c.game_date) || null);
      $('coList').innerHTML = visible.map((c)=>{
```

### If it STILL doesn't show after the above, it's one of these — diagnose:
1. **Saved as a draft** — it will now appear with a yellow `DRAFT` tag. Rob may not have done the final lock. Have him reopen and lock it.
2. **The game was genuinely mis-flagged** (`is_cash_game=true`) when it's a tournament. Fix the data:
   ```sql
   select id, game_name, venue_name, game_date, is_cash_game, is_draft, submitted_by
   from closeouts where game_name ilike '%double barrel%';
   -- if wrongly flagged as cash:
   update closeouts set is_cash_game=false where game_name ilike '%double barrel%';
   ```
3. **RLS visibility** — the admin login can't see Rob's row. Check the closeouts SELECT policy allows admins to read all rows:
   ```sql
   -- example read-all policy (adjust to your role model if needed)
   create policy if not exists "admins read all closeouts"
   on closeouts for select to authenticated using (true);
   ```

---

## After editing
1. Strict-validate (see top of this doc).
2. Commit and push `admin_view_cloud_test.html`.
   - NOTE: this repo has an org-level 403 on pushes — if the push is blocked, upload the file manually via GitHub (Add file → Upload files → drag → Commit), named exactly `admin_view_cloud_test.html`.
3. Run the SQL fixes in Supabase.
4. Hard-refresh the live page (Ctrl+Shift+R) or open in incognito to bypass cache.
5. Verify: Club Marconi games show $15k ÷ (games that month) each; Double Barrel Fridays appears on the report.
