# Lightning Storm Leaf Strategy Backtest Dashboard

## Project Overview
Single-file HTML dashboard for backtesting the "Leaf strategy" on Evolution Gaming's Lightning Storm casino game. Data comes from casinodrop.com CSV exports. No backend, no build step — open `dashboard.html` in any browser.

**File:** `D:\lighting strom\dashboard.html`

---

## Game & Strategy

### Lightning Storm outcomes
`Leaf`, `BatteryCharger`, `MonsterMash`, `HotSpot`, `Fireball`, `StormBonus`

### Strategy rules
1. **WAITING_FOR_LEAF** — watch every round, do not bet
2. First Leaf → **SIGNAL** (no bet placed) → enter **LEAF_STREAK_ACTIVE**
3. **LEAF_STREAK_ACTIVE** — bet every round using Martingale progression:
   - Level 1 → ₹100, Level 2 → ₹200, Level 3 → ₹400
   - **Leaf result = WIN** → collect stake × payout, reset to Level 1, stay active
   - **Non-Leaf result = LOSS** → advance level; if was Level 3 → enter cooldown (if enabled) else back to WAITING_FOR_LEAF
4. **COOLDOWN** (optional) — after a full 3-step loss cycle (₹100+₹200+₹400 all lose), pause betting for the configured cooldown period before returning to WAITING_FOR_LEAF

### Key rule: Signal Leaf is NOT a bet
The first Leaf after waiting only activates the streak — no money placed on it.

### Progression carries across streaks
After a non-Leaf loss at Level 2, the next streak starts at Level 3 (not reset).

---

## Data Source
- **URL:** casinodrop.com/tracker/lightningstorm/
- **Method:** Manual CSV export (live fetch blocked by CORS)
- **CSV headers:** `settled_at_utc, outcome, number, payout, lightning_boost, paid_out_eur, winners, best_win_eur, best_win_player`
- **Timestamp column detection:** checks for `time`, `date`, `timestamp`, `utc`, `settled` in header name
- **Outcome column detection:** checks for `result`, `outcome`

---

## Features Implemented

### Strategy engine (`runBacktest`)
- 3-state machine: `WAITING_FOR_LEAF` → `LEAF_STREAK_ACTIVE` → `COOLDOWN`
- Tracks: bankroll, cumulative P/L, max drawdown, win rate, streak lengths, cooldown periods
- Cooldown configurable: None / 30min / 45min / 1hr / 1.5hr / 2hr / 3hr
- Deduplication by `roundId` or timestamp+result
- Input normalization (sorts by timestamp, handles newest-first CSV)
- 25 automated tests (all must pass — check testsCard on load)

### Settings (auto-saved to localStorage)
- Time window filter: 1h / 6h / 12h / 24h / 3d / 7d / All
- Starting bankroll (₹)
- Payout multiplier (1:1 / 0.97 / 0.95)
- Cooldown after 3-loss cycle
- Sessions (see below)
- Theme preference

### IST Time Analysis (UTC+5:30)
- 24-hour table showing bets/wins/losses/P/L per hour in 12-hour IST format
- Best 3 / Worst 3 hours highlighted
- Day-of-week breakdown (7 tiles)
- All timestamps in round history shown in IST

### Betting Sessions
- User defines daily time windows (e.g. 12:00 AM – 2:00 AM IST)
- Multiple sessions supported
- Session P/L table: per-day, per-session breakdown
- Summary bar: total session bets, wins, losses, net P/L, win rate
- Sessions persist in localStorage

### Charts
- Bankroll over time (line, Chart.js 4.4.0)
- Net P/L per bet (bar)

### Stats grid (12 cards)
Net P/L, Bankroll, Win Rate, Total Wagered, Leaf Signals, Longest Streak, Max Drawdown, Longest Loss Run, Total Rounds, Next Bet Stake, **3-Loss Cycles**, **Rounds in Cooldown**

### Round history table
Columns: # | Time (IST) | Result | Action | Stake | P/L | Cum P/L | Bankroll | Next ₹
Action badges: Wait (grey) | Signal (blue) | Win ₹X (green) | Loss ₹X (red) | Cooldown (amber)

### PWA / Mobile
- Manifest injected via JS data URI
- iOS `apple-mobile-web-app-capable` meta tag (Add to Home Screen → standalone)
- Android `beforeinstallprompt` handler → Install button in header
- Mobile install banner on first open (dismissible, saved to localStorage)
- Install Guide card (step-by-step for iOS Safari + Android Chrome)
- Floating Action Button (▶ Run) fixed bottom-right on screens ≤640px
- Safe area insets for iPhone notch/home indicator

---

## Color Scheme (Indigo / Midnight)

| Token | Light | Dark |
|---|---|---|
| Primary (acc) | Indigo `#4F46E5` | Soft indigo `#818CF8` |
| Header | Gradient `#3730A3→#6366F1` | same |
| Background | Lavender-white `#F5F5FA` | Midnight `#0C0B16` |
| Surface | White `#FFFFFF` | Navy `#131221` |
| Win/Profit | Emerald `#059669` | Mint `#34D399` |
| Loss | Rose `#DC2626` | Soft red `#F87171` |
| Signal | Blue `#2563EB` | Sky `#60A5FA` |
| Cooldown/Warn | Amber `#B45309` | Gold `#FBBF24` |

Indigo is intentionally different from win/loss colors to avoid visual confusion.

---

## Technical Notes

### IST conversion
```javascript
var IST_MS = 19800000; // UTC+5:30 in ms
function toIST(ts) { return new Date(new Date(ts).getTime() + IST_MS); }
```
Reads UTC hours/day from the shifted Date — avoids all locale/timezone issues.

### Stake levels
```javascript
function stakeFor(lv) { return lv===1?100 : lv===2?200 : 400; }
function nextLv(lv)   { return lv===1?2   : lv===2?3   : 1;   }
```

### isLeaf detection
```javascript
function isLeaf(r) {
  var n = r.trim().toLowerCase().replace(/\s+/g,' ');
  return n==='leaf' || n==='evo leaf' || n==='evoleaf';
}
```

### localStorage keys
| Key | Contents |
|---|---|
| `ls_theme` | `'system'` / `'dark'` / `'light'` |
| `ls_cfg` | `{win, bank, pay, cd}` — backtest settings |
| `ls_sessions` | Array of `{start, end}` in IST minutes from midnight |
| `ls_ib_dismissed` | `'1'` when install banner dismissed |

### External dependencies
- **Font:** Google Fonts — Plus Jakarta Sans (400/500/600/700/800)
- **Charts:** Chart.js 4.4.0 UMD from cdnjs.cloudflare.com
- No other external dependencies

---

## What Does NOT Work
- **Live data fetch:** casinodrop.com blocks browser fetch via CORS — always fails gracefully with instructions to use CSV export
- **Service Worker:** Cannot register from `file://` protocol — PWA offline mode unavailable unless served via HTTP

---

## User Preferences
- Real data only — no demo data, no Load Demo button
- Indian Rupee (₹) throughout
- IST (UTC+5:30) for all timestamps and time analysis
- 12-hour time format for IST display
- Settings must persist between sessions (localStorage)
- Mobile-first, installable as home screen app
