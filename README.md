# 🎮 Fidge — Fidget Spinner Rewards App

Fidge is a full-stack web3-enabled fidget spinner game where users earn points by spinning, convert them to gems, win prizes on the spin wheel, and withdraw $PCEDO tokens to their crypto wallet. Built as a Progressive Web App (PWA) accessible from any browser.

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Tech Stack](#tech-stack)
3. [App Pages](#app-pages)
4. [Skin System](#skin-system)
5. [Energy & Ads](#energy--ads)
6. [Quest System](#quest-system)
7. [Referral System](#referral-system)
8. [Gem Economy](#gem-economy)
9. [Spin Wheel](#spin-wheel)
10. [$PCEDO Withdrawal](#pcedo-withdrawal)
11. [Marketplace — Buy Gems with ETH](#marketplace)
12. [Ad Integration](#ad-integration)
13. [Admin Panel](#admin-panel)
14. [Project Structure](#project-structure)
15. [Environment Variables](#environment-variables)
16. [Deployment Guide](#deployment-guide)
17. [Auth Flow](#auth-flow)
18. [API Reference](#api-reference)
19. [Database Tables](#database-tables)
20. [Key Business Rules](#key-business-rules)
21. [Security](#security)

---

## Overview

Users spin a fidget spinner to earn points. Points convert to gems. Gems unlock premium skins that make you earn faster and spin longer. The spin wheel gives you a chance at $PCEDO tokens. Everything is gamified with quests, referrals, and a live leaderboard.

**Revenue model:** Ads on the energy refill flow + ETH payments in the gem marketplace.

---

## Tech Stack

| Layer | Technology | Platform |
|---|---|---|
| Frontend | Next.js 15 (App Router) · TypeScript · Tailwind | Vercel |
| Backend | Laravel 11 · PHP 8.2 | Railway |
| Database | MySQL 8 | Railway |
| Email | Mailjet SMTP (port 465 SSL) | — |
| Auth | Email + OTP signup · Email + password login | — |
| Token | Custom `AuthToken` table · `X-Auth-Token` header | — |
| Payments | ETH (Ethereum mainnet) via wallet connect | — |
| Ads | Adsterra (now) + Google AdSense (after approval) | — |

---

## App Pages

| Route | Page | Description |
|---|---|---|
| `/` | Home | Animated 3D spinner, sign-in CTA, navigation |
| `/spinner` | Spinner | Main game — spin to earn points + energy system |
| `/wheel` | Spin Wheel | Gem-based prize wheel with auto-spin |
| `/shop` | Gem Shop | Browse and buy skins with gems |
| `/profile` | Profile | Stats, convert points, manage skins, withdraw $PCEDO |
| `/leaderboard` | Leaderboard | Global points + referral rankings |
| `/marketplace` | Marketplace | Buy gems with ETH |
| `/join` | Referral Landing | Referral invite page with pre-filled code |
| `/fidge-admin` | Admin Panel | Backend management (admin only) |

---

## App Features

### 🌀 Spinner (`/spinner`)
- Drag/swipe to spin the fidget spinner
- Points earned every second the spinner is moving with energy available
- Energy drains as you spin — 100% max per skin session
- **Active skin PNG renders as the spinner graphic**
- Skin multipliers apply server-side on every sync call (cannot be spoofed)
- Points sync to backend every few seconds via `/spinner/sync`
- Session logged on spin end via `/spinner/session-end` (triggers quest checks)
- Switch skins mid-session — each skin has its own independent energy pool

### 🎡 Spin Wheel (`/wheel`)
- Costs **2 💎 per spin**
- 10 weighted prize segments: Points, Gems, $PCEDO
- **Auto-Spin**: pick 3×, 5×, 10×, 20×, or 50× — runs automatically
- Auto-spin shows count remaining, STOP button available at all times
- All gem deductions + reward credits happen server-side
- Unaffordable auto-spin options are greyed out

### 🛒 Gem Shop (`/shop`)
- All skins displayed with: name, rarity, gem cost, image, and benefit bullets
- Filter: All / Owned / Available
- Buy skin → deducted from gems immediately
- Equip skin → updates `active_skin` on user record
- Obsidian is the only free default skin (Chrome costs 30 💎)

### 👤 Profile (`/profile`)
- **Balance ring** — total points from all sources (spin + quests + wheel + coupons)
- **Spinning stat** — points earned from spinner only
- **Gems stat** — current gem balance
- **$PCEDO stat** — total $PCEDO earned, labelled "Total earned"
- **Convert tab** — stepper to convert points → gems (10,000 pts = 1 💎)
- **Skins tab** — view owned skins, equip them, see benefits per skin
- **Withdraw tab** — withdraw $PCEDO to ETH wallet (min 100, 0.5% fee)
- **Quests section** — progress tracker, confirm manual quests

### 🏆 Leaderboard (`/leaderboard`)
- Points tab + Referrals tab
- Each entry shows total points + referral count
- Sticky "YOU" bar at bottom with your rank
- Referral panel: active (green) / inactive (grey) per referral
- Cycle-based — resets every 14 days (configurable)

---

## Skin System

Skins are cosmetic upgrades that also boost gameplay performance.

| Skin | Cost | Earn Multiplier | Energy Drain | Extra Spin Time |
|---|---|---|---|---|
| Obsidian | **Free** | 1.0× (base) | 1.0× (base) | — |
| Chrome | 30 💎 | 1.0× | 1.0× | — |
| Gold | 50 💎 | **1.3×** (+30%) | 0.85× | +18% longer |
| Sapphire | 75 💎 | **1.4×** (+40%) | 0.80× | +25% longer |
| Neon | 80 💎 | **1.6×** (+60%) | 0.72× | +39% longer |
| Plasma | 100 💎 | **1.7×** (+70%) | 0.65× | +54% longer |

**Technical implementation:**
- `getSkinMultiplier()` in `SpinnerController` reads `user.active_skin` on every sync
- Points earned = `raw_points × multiplier`
- Energy drained = `raw_energy × energy_factor`
- Skins have independent energy pools — switch skins to get fresh energy
- Skin PNG images stored in `/public/skins/`, served as the spinner visual

---

## Energy & Ads

### Energy System
- Each user has 100% energy per skin session
- Energy drains as they spin (faster with lower-tier skins)
- Stored in `energy_sessions` table (per user, per day)
- Different skins = different energy pools (switch skin = fresh energy)

### Watch Ad → Refill Energy
1. User taps **Watch Ad** on Spinner page
2. `AdModal` opens (fullscreen)
3. Ad plays — Adsterra popunder fires in background tab
4. **15-second countdown** (can't skip)
5. "Claim +20% Energy" button activates
6. User taps Claim → backend awards +20% via `/spinner/watch-ad`
7. Modal closes, energy refilled

### Ad Limits
- **5 ads per batch** (each +20% = possible 100% refill)
- **2-hour cooldown** after completing a batch of 5
- Backend tracks `last_ad_at` on `energy_sessions`
- If cooldown active, backend returns remaining minutes in error

---

## Quest System

| Quest | How It Completes | Reward |
|---|---|---|
| First Spin | Auto — after 1 spin session | 10 pts |
| Spin 10 Times | Auto — after 10 total sessions | 25 pts |
| Spin 50 Times | Auto — after 50 total sessions | 75 pts |
| Earn 100 Points | Auto — when spin_points ≥ 100 | 20 pts |
| Speed Demon | Auto — 30+ seconds spinning today | 15 pts |
| Daily Grind | Auto — 7 unique spin days | 50 pts |
| First Referral | Manual confirm — after 1 referral | 50 pts |
| Watch an Ad | Manual confirm — after watching 1 ad | 5 pts |
| Wheel Spin | Manual confirm — after first wheel spin | 10 pts |

**Auto quests** — backend detects completion during spin sync or session-end.
**Manual quests** — user taps "Confirm Completion" on profile page; backend validates before awarding.

---

## Referral System

1. User shares their unique link: `https://yourdomain.com/join?ref=THEIRCODE`
2. New user arrives at `/join` — invite card shows with code pre-filled
3. New user registers — ref code sent to backend
4. Referrer's `referral_count` increments
5. When referred user hits **10,000 total points** → counts as **active referral**
6. Gem milestone bonuses auto-credited:

| Referrals | Bonus |
|---|---|
| 10 | 1 💎 |
| 15 | 3 💎 |
| 30 | 9 💎 |
| 50 | 15 💎 |
| 100 | 50 💎 |

---

## Gem Economy

### Ways to Earn Gems
| Source | Amount |
|---|---|
| Default (signup) | Obsidian skin (free) |
| Convert points | 10,000 pts = 1 💎 |
| Spin Wheel prize | 1, 2, or 10 💎 |
| Referral milestones | 1 – 50 💎 |
| Coupon redeem | Variable |
| Marketplace (ETH) | 50 – 4,000 💎 |

### Ways to Spend Gems
| Item | Cost |
|---|---|
| Wheel spin | 2 💎 |
| Chrome skin | 30 💎 |
| Gold skin | 50 💎 |
| Sapphire skin | 75 💎 |
| Neon skin | 80 💎 |
| Plasma skin | 100 💎 |

---

## Spin Wheel

### Prize Segments (10 total)
| Label | Prize | Type | Weight | Rarity |
|---|---|---|---|---|
| 50 PTS | 50 Points | points | 30 | Common |
| 250 PTS | 250 Points | points | 22 | Common |
| 500 PTS | 500 Points | points | 18 | Common |
| 1 💎 | 1 Gem | gems | 20 | Common |
| 2 💎 | 2 Gems | gems | 10 | Uncommon |
| 10 💎 | 10 Gems | gems | 3 | Rare |
| 1 PCEDO | 1 $PCEDO | pcedo | 12 | Uncommon |
| 5 PCEDO | 5 $PCEDO | pcedo | 5 | Uncommon |
| 10 PCEDO | 10 $PCEDO | pcedo | 3 | Rare |
| 100 PCEDO | 100 $PCEDO | pcedo | 1 | Ultra Rare |

*Higher weight = more likely. Total weight = 124.*

### Auto-Spin
- Options: 3×, 5×, 10×, 20×, 50×
- Shows total gem cost per option
- Runs as an async loop: spin → animation (7s) → result modal → 1.5s gap → next
- STOP button cancels at any time
- Server-side gem deduction + reward on every iteration

---

## $PCEDO Withdrawal

### Earning $PCEDO
Spin Wheel prizes: 1, 5, 10, or 100 $PCEDO per spin.

### Withdrawal Rules
- **Minimum:** 100 $PCEDO per request
- **Fee:** 0.5% of gross amount (deducted before sending)
- **Processing:** Manual by admin (24–48 hours)
- **Network:** ETH wallet (Ethereum)

### Fee Example
| Withdraw | Fee (0.5%) | You Receive |
|---|---|---|
| 100 $PCEDO | 0.5 $PCEDO | 99.5 $PCEDO |
| 500 $PCEDO | 2.5 $PCEDO | 497.5 $PCEDO |
| 1,000 $PCEDO | 5 $PCEDO | 995 $PCEDO |

### Stepper Options: 100 / 200 / 500 / 1,000 / 2,000 $PCEDO

### User Flow
1. Profile → Withdraw tab
2. Select amount with stepper
3. Fee breakdown shows automatically
4. Enter ETH wallet address
5. Submit → request queued for admin
6. Cancel any pending request → $PCEDO instantly refunded

### Admin Flow
- Admin Panel → 🪙 PCEDO OUT tab
- Filter Pending / Processed
- **CONFIRM SENT** → marks processed after you send the crypto
- **DELETE** → removes record, refunds $PCEDO if still pending

---

## Marketplace

Buy gem packages with ETH directly from the app.

### Packages
| Gems | USD | ETH (at $3,000/ETH) |
|---|---|---|
| 50 💎 | $0.99 | ~0.00033 ETH |
| 150 💎 | $2.49 | ~0.00083 ETH |
| 350 💎 | $4.99 | ~0.00166 ETH |
| 750 💎 | $9.99 | ~0.00333 ETH |
| 1,600 💎 ⭐ | $19.99 | ~0.00666 ETH |
| 4,000 💎 | $44.99 | ~0.01499 ETH |

*ETH price calculated from `ETH_PRICE_USD` env var.*

### Wallet Connect
**In wallet browser** (MetaMask, Trust, etc.):
- Wallet detected automatically → Connect button → approve → done

**In external browser** (Chrome, Safari):
- Deep links shown for 4 wallets:
  - 🦊 **MetaMask** — `metamask.app.link/dapp/...`
  - 🛡️ **Trust Wallet** — `link.trustwallet.com/open_url?...`
  - 🟣 **Farcaster** — `warpcast.com/~/frames/launch?domain=...`
  - 🟡 **Binance Web3** — `app.binance.com/cedefi/web3-wallet?url=...`

### Payment Flow
1. Pick package → wallet opens with exact ETH amount pre-filled
2. Approve transaction in wallet
3. TX hash submitted to backend
4. **Coupon auto-issued immediately** (no manual admin step)
5. User redeems coupon → gems credited instantly

---

## Ad Integration

### Architecture
Ads have **one placement** in the entire app: the **Watch Ad** button on the Spinner page tied to energy refills. No banner ads, no interstitials between pages — just the rewarded energy flow.

```
User energy low → tap "Watch Ad"
→ AdModal opens fullscreen
→ Ad fires (background tab for Adsterra / display unit for AdSense)
→ 15s countdown (Adsterra) or 5s countdown (AdSense)
→ "Claim +20% Energy" activates
→ Backend awards energy
→ 5 ads per batch → 2hr cooldown
```

### Providers

| Provider | Status | Format | Countdown | Revenue |
|---|---|---|---|---|
| **Adsterra** | ✅ Use now | Popunder + Social Bar | 15s | Per impression/click |
| **AdSense** | ⏳ After approval | Display unit in modal | 5s | Per impression/click |
| **Both** | 🎯 Long-term goal | Both simultaneously | 10s | Two streams |

### Setting Up Adsterra (Start Here)

1. Sign up at [adsterra.com](https://adsterra.com)
2. Add your site → get approved (usually same day)
3. Create two ad units:
   - **Social Bar** — copy the script `src` URL
   - **Onclick (Popunder)** — copy the script `src` URL
4. Add to Vercel env vars:
   ```
   NEXT_PUBLIC_AD_PROVIDER=adsterra
   NEXT_PUBLIC_ADSTERRA_SOCIAL_BAR=https://your-social-bar-script-url.js
   NEXT_PUBLIC_ADSTERRA_POPUNDER=https://your-popunder-script-url.js
   ```
5. Redeploy → earning immediately

### Setting Up AdSense (After Approval)

1. Get a **custom domain** — AdSense won't approve `.vercel.app`
2. Apply at [adsense.google.com](https://adsense.google.com) (1–2 weeks review)
3. Create a **Display Ad** unit → get publisher ID + slot ID
4. Update `public/ads.txt`:
   ```
   google.com, pub-YOUR_REAL_ID, DIRECT, f08c47fec0942fa0
   ```
5. Add to Vercel:
   ```
   NEXT_PUBLIC_ADSENSE_CLIENT=ca-pub-XXXXXXXXXXXXXXXX
   NEXT_PUBLIC_ADSENSE_SLOT=1234567890
   ```

### Switching / Combining Providers
Just change `NEXT_PUBLIC_AD_PROVIDER` in Vercel and redeploy:

```bash
# Adsterra only (now)
NEXT_PUBLIC_AD_PROVIDER=adsterra

# AdSense only (after approval)
NEXT_PUBLIC_AD_PROVIDER=adsense

# Both simultaneously (recommended long-term — 2 revenue streams)
NEXT_PUBLIC_AD_PROVIDER=both
```

No code changes ever needed. One env var controls everything.

---

## Admin Panel

Access at `/fidge-admin` — login with `ADMIN_USERNAME` / `ADMIN_PASSWORD` env vars.

### Tabs

| Tab | What You Can Do |
|---|---|
| 📊 Stats | Total users, active today, total points, total gems, banned users |
| 🎟️ Coupons | Create (gems/points type, value, max uses, expiry) · toggle · delete |
| 💎 Gem Orders | View ETH purchase requests · Verify (auto-issues coupon) · Reject |
| 🪙 PCEDO OUT | View withdrawal requests · Confirm Sent · Delete (refunds if pending) |
| 👥 Users | Search · view stats · ban/unban |
| ➕ Create Coupon | Form to create a new coupon code |

---

## Project Structure

```
/
├── backend/                            Laravel 11
│   ├── app/
│   │   ├── Http/
│   │   │   ├── Controllers/
│   │   │   │   ├── AuthController.php          Register (OTP) + Login
│   │   │   │   ├── SpinnerController.php       Sync, session-end, watch-ad
│   │   │   │   ├── WheelController.php         Spin wheel + prize award
│   │   │   │   ├── ProfileController.php       Profile, convert, skins, withdraw
│   │   │   │   ├── ShopController.php          Skin list + gem purchase
│   │   │   │   ├── LeaderboardController.php   Rankings
│   │   │   │   ├── MarketplaceController.php   ETH gem purchases
│   │   │   │   ├── CouponController.php        Coupon redemption
│   │   │   │   └── AdminController.php         Admin panel API
│   │   │   └── Middleware/
│   │   │       ├── SessionAuth.php     X-Auth-Token → $request->user()
│   │   │       ├── AdminAuth.php       X-Admin-Token → admin validation
│   │   │       └── BanCheck.php        Blocks banned users on every request
│   │   └── Models/
│   │       ├── User.php
│   │       ├── Skin.php
│   │       ├── Quest.php
│   │       ├── AuthToken.php           30-day token expiry
│   │       ├── EnergySession.php       Per-user per-day energy + ads
│   │       ├── SpinLog.php             Spin session history
│   │       ├── WheelSpin.php           Wheel spin history
│   │       ├── GemPurchaseRequest.php  ETH marketplace purchases
│   │       ├── LeaderboardEntry.php
│   │       ├── LeaderboardCycle.php
│   │       ├── Coupon.php
│   │       ├── CouponRedemption.php
│   │       └── PcedoWithdrawal.php
│   ├── database/
│   │   ├── migrations/                 All table definitions
│   │   └── seeders/
│   │       └── DatabaseSeeder.php      6 skins + 8 quests + 8 demo leaderboard users
│   ├── routes/
│   │   └── api.php                     All API routes
│   └── config/
│       └── fidge.php                   ETH wallet, leaderboard cycle config
│
└── frontend/                           Next.js 15
    ├── app/
    │   ├── layout.tsx                  Global layout + ad SDK scripts
    │   ├── page.tsx                    Home page
    │   ├── spinner/page.tsx            Fidget spinner game
    │   ├── wheel/page.tsx              Spin wheel + auto-spin
    │   ├── shop/page.tsx               Skin store
    │   ├── profile/page.tsx            Stats, convert, skins, withdraw, quests
    │   ├── leaderboard/page.tsx        Rankings
    │   ├── marketplace/page.tsx        Buy gems with ETH
    │   ├── join/page.tsx               Referral landing (Suspense wrapper)
    │   ├── join/JoinContent.tsx        Referral page content
    │   └── fidge-admin/page.tsx        Admin panel
    ├── components/
    │   ├── TopHeader.tsx               Header: gems badge, username, 3-dot menu
    │   ├── BottomNav.tsx               Mobile bottom navigation
    │   ├── AuthModal.tsx               Login + Register with OTP modal
    │   ├── AdModal.tsx                 Universal ad modal (Adsterra + AdSense)
    │   └── LegalModal.tsx             Terms, Privacy, Disclaimer
    ├── context/
    │   └── AuthContext.tsx             Global auth + energy + gems state
    ├── lib/
    │   └── api.ts                      All typed API calls
    └── public/
        ├── logo.png                    Fidge logo
        ├── fidge3d.png                3D spinner for home page
        ├── ads.txt                     AdSense publisher verification
        ├── sw.js                       Service worker (Monetag requirement)
        └── skins/
            ├── Fidger.png             Obsidian skin
            ├── Based.png             Chrome skin
            ├── EarlyFidger.png       Gold skin
            ├── Christmas.png         Sapphire skin
            ├── Galaxy.png            Plasma skin
            └── Pizza.png             Neon skin
```

---

## Environment Variables

### Backend (Railway)

```env
APP_NAME=Fidge
APP_ENV=production
APP_KEY=base64:your_generated_key_here
APP_DEBUG=false
APP_URL=https://your-backend.up.railway.app

LOG_CHANNEL=stderr

DB_CONNECTION=mysql
DB_HOST=your_railway_mysql_host
DB_PORT=3306
DB_DATABASE=your_database_name
DB_USERNAME=your_username
DB_PASSWORD=your_password

# Email — Mailjet only (Railway blocks port 587, use 465 SSL)
MAIL_MAILER=smtp
MAIL_HOST=in-v3.mailjet.com
MAIL_PORT=465
MAIL_ENCRYPTION=ssl
MAIL_USERNAME=your_mailjet_api_key
MAIL_PASSWORD=your_mailjet_secret_key
MAIL_FROM_ADDRESS=noreply@yourdomain.com
MAIL_FROM_NAME=Fidge

FRONTEND_URL=https://yourdomain.com

CACHE_STORE=database
QUEUE_CONNECTION=database
SESSION_DRIVER=cookie
SESSION_LIFETIME=43200

ADMIN_USERNAME=your_admin_username
ADMIN_PASSWORD=your_admin_password

# Marketplace
ETH_WALLET_ADDRESS=0xYourEthereumWalletAddress
ETH_PRICE_USD=3000

# Leaderboard
LEADERBOARD_EPOCH_START=2025-01-01
LEADERBOARD_CYCLE_DAYS=14
```

### Frontend (Vercel)

```env
NEXT_PUBLIC_API_URL=https://your-backend.up.railway.app/api
NEXT_PUBLIC_ETH_WALLET=0xYourEthereumWalletAddress

# ── Ad Provider ───────────────────────────────────────────────────────────────
# "adsterra" | "adsense" | "both" | leave empty for dev
NEXT_PUBLIC_AD_PROVIDER=adsterra

# AdSense (when AD_PROVIDER = "adsense" or "both")
NEXT_PUBLIC_ADSENSE_CLIENT=ca-pub-XXXXXXXXXXXXXXXX
NEXT_PUBLIC_ADSENSE_SLOT=1234567890

# Adsterra (when AD_PROVIDER = "adsterra" or "both")
# Social Bar script URL — from Adsterra → Social Bar ad unit
NEXT_PUBLIC_ADSTERRA_SOCIAL_BAR=https://your-adsterra-social-bar-url.js
# Popunder script URL — from Adsterra → Onclick (Popunder) ad unit
NEXT_PUBLIC_ADSTERRA_POPUNDER=https://your-adsterra-popunder-url.js
```

---

## Deployment Guide

### Railway (Backend)

```bash
# 1. Create Railway project
# 2. Add MySQL plugin
# 3. Connect GitHub repo, set root directory to: backend/
# 4. Add all backend env vars
# 5. Railway auto-runs migrations on every deploy

# First time only — seed the database:
railway run php artisan db:seed

# Generate APP_KEY if needed:
php artisan key:generate --show
```

### Vercel (Frontend)

```bash
# 1. Import GitHub repo at vercel.com
# 2. Set root directory to: frontend/
# 3. Framework: Next.js (auto-detected)
# 4. Add all frontend env vars
# 5. Deploy
# 6. Add custom domain in Vercel → Settings → Domains
```

### First Deploy Checklist

- [ ] Railway MySQL created and connected
- [ ] All backend env vars set
- [ ] `APP_KEY` generated
- [ ] Migrations ran successfully (check Railway deploy logs)
- [ ] Database seeded (`php artisan db:seed`)
- [ ] Vercel frontend env vars set
- [ ] Custom domain connected
- [ ] Adsterra account created and site approved
- [ ] Adsterra Social Bar + Popunder scripts added to Vercel env
- [ ] `public/ads.txt` updated with real AdSense publisher ID (after approval)

---

## Auth Flow

### Registration (2 steps)
```
1. POST /api/auth/register/send-otp
   Body: { email, username, password, ref_code? }
   → Sends 6-digit OTP to email (expires 10 min)

2. POST /api/auth/register/verify
   Body: { email, username, password, otp, ref_code? }
   → Creates account, returns { token, user }
```

### Login (1 step — no OTP)
```
POST /api/auth/login
Body: { email, password }
→ Returns { token, user }
```

### Token Storage
- Stored in `localStorage` as `fidge_token`
- Sent as `X-Auth-Token` header on every authenticated request
- Tokens expire after 30 days
- Logout deletes token from the database

---

## API Reference

### Auth
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/auth/register/send-otp` | ❌ | Send OTP to email |
| POST | `/auth/register/verify` | ❌ | Verify OTP, create account |
| POST | `/auth/login` | ❌ | Email + password login |
| POST | `/auth/logout` | ✅ | Delete token |
| GET | `/auth/me` | ✅ | Current user object |

### Spinner
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/spinner/sync` | ✅ | Sync points + energy drain |
| POST | `/spinner/session-end` | ✅ | Log spin session (triggers quest checks) |
| POST | `/spinner/watch-ad` | ✅ | Award +20% energy after ad |

### Profile
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/profile` | ✅ | Full profile data |
| POST | `/profile/convert-points` | ✅ | Convert points → gems |
| POST | `/profile/set-skin` | ✅ | Equip a skin |
| POST | `/profile/withdraw-pcedo` | ✅ | Submit withdrawal request |
| GET | `/profile/withdrawals` | ✅ | Withdrawal history |
| DELETE | `/profile/withdrawals/{id}` | ✅ | Cancel + refund pending withdrawal |
| POST | `/profile/quests/{id}/confirm` | ✅ | Confirm manual quest |

### Wheel
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/wheel/spin` | ✅ | Spin (costs 2 💎), returns prize |
| GET | `/wheel/segments` | ❌ | Get prize segments list |

### Shop
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/shop/skins` | ❌ | All skins (owned flag if authed) |
| POST | `/shop/skins/{id}/buy` | ✅ | Buy skin with gems |

### Marketplace
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/marketplace/packages` | ❌ | Gem packages + wallet address |
| POST | `/marketplace/initiate` | ✅ | Create purchase request |
| POST | `/marketplace/submit-tx` | ✅ | Submit TX hash (auto-issues coupon) |
| GET | `/marketplace/status` | ✅ | Purchase history |

### Leaderboard
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/leaderboard` | ❌ | Rankings + user referral list |

### Coupons
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/coupons/redeem` | ✅ | Redeem coupon code |

### Admin (requires `X-Admin-Token` header)
| Method | Endpoint | Description |
|---|---|---|
| POST | `/admin/login` | Get admin token |
| GET | `/admin/stats` | Dashboard numbers |
| GET | `/admin/coupons` | List coupons |
| POST | `/admin/coupons` | Create coupon |
| PATCH | `/admin/coupons/{id}/toggle` | Enable/disable coupon |
| DELETE | `/admin/coupons/{id}` | Delete coupon |
| GET | `/admin/users` | List users (paginated, searchable) |
| PATCH | `/admin/users/{id}/ban` | Ban or unban user |
| GET | `/admin/gem-requests` | ETH purchase requests |
| POST | `/admin/gem-requests/{id}/verify` | Verify + issue coupon |
| POST | `/admin/gem-requests/{id}/reject` | Reject request |
| GET | `/admin/withdrawals` | $PCEDO withdrawal queue |
| POST | `/admin/withdrawals/{id}/confirm` | Mark as sent |
| DELETE | `/admin/withdrawals/{id}` | Delete (refunds if pending) |

---

## Database Tables

| Table | Purpose |
|---|---|
| `users` | Accounts: email, username, points, gems, active_skin, referral_code |
| `auth_tokens` | Login tokens (30-day expiry) |
| `skins` | Skin catalog: name, rarity, gem_cost, shade, image_url, is_default |
| `user_skins` | Pivot: which skins each user owns |
| `quests` | Quest catalog: title, type, reward_points |
| `user_quests` | Pivot: quest completion status per user |
| `energy_sessions` | Per-user per-day energy: energy%, ads_watched, last_ad_at |
| `spin_logs` | Spin session history: duration, points, energy used |
| `wheel_spins` | Wheel history: segment_index, prize_type, prize_value, gems_spent |
| `leaderboard_cycles` | Time periods for leaderboard resets |
| `leaderboard_entries` | Points per user per cycle |
| `coupons` | Coupon codes: type, value, max_uses, used_count, expiry |
| `coupon_redemptions` | Who redeemed what coupon |
| `gem_purchase_requests` | ETH marketplace purchases: tx_hash, status, coupon_code |
| `pcedo_withdrawals` | $PCEDO withdrawal queue: amount, fee, wallet_address, status |

---

## Key Business Rules

| Rule | Value |
|---|---|
| Points → Gems conversion | 10,000 pts = 1 💎 |
| Wheel cost | 2 💎 per spin |
| Ads per batch | 5 maximum |
| Energy per ad | +20% |
| Ad cooldown | 2 hours between batches |
| Active referral threshold | 10,000 total points |
| Minimum $PCEDO withdrawal | 100 $PCEDO |
| Withdrawal fee | 0.5% of gross amount |
| Auth token expiry | 30 days |
| OTP expiry | 10 minutes |
| Leaderboard cycle | 14 days (configurable) |
| ETH network | Ethereum mainnet only |
| Default free skin | Obsidian only |

---

## Security

- **All game logic is server-side** — points, prizes, and energy are calculated on the backend. The frontend sends what happened; the backend decides what it's worth
- **Skin multipliers** are read from `user.active_skin` on the server — client can't spoof a higher multiplier
- **Wheel results** are determined server-side using weighted random — client only receives the result index
- **Gem deductions** for wheel spins happen in a database transaction — no double-spend possible
- **TX hash deduplication** — same hash cannot be submitted twice for gem purchases
- **Admin routes** use a separate `X-Admin-Token` completely distinct from user tokens
- **Banned users** are rejected at middleware level on every request
- **OTP codes** are single-use and expire after 10 minutes
- **CORS** — only your frontend URL is whitelisted (set via `FRONTEND_URL` env var)

---

*Built with ❤️ — @Fidge_App*
