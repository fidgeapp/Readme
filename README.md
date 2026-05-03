# 🌀 Fidge — Web3 Fidget Spinner Rewards Platform

**fidge.app** · [@FIDGE_APP](https://linktr.ee/cedomisofficial)

Fidge is a gamified Web3 spinner platform where users earn real rewards by spinning a fidget spinner. Spin to earn points, convert points to gems, unlock premium skins that boost your earnings, compete on the leaderboard for prize cycles, win $PCEDO tokens on the bonus wheel, and withdraw crypto to your wallet — all from a mobile-first PWA.

> **Revenue model:** Ad-gated energy refills (Adsterra + AdSense) + ETH gem purchases via the marketplace.

---

## 📋 Table of Contents

1. [How Fidge Works](#how-fidge-works)
2. [Tech Stack](#tech-stack)
3. [App Pages](#app-pages)
4. [Spinner & Energy System](#spinner--energy-system)
5. [Skin System](#skin-system)
6. [Watch Ads → Refill Energy](#watch-ads--refill-energy)
7. [Gem Economy](#gem-economy)
8. [Spin Wheel](#spin-wheel)
9. [Quest System](#quest-system)
10. [Referral Program](#referral-program)
11. [$PCEDO Withdrawal](#pcedo-withdrawal)
12. [Gem Marketplace](#gem-marketplace)
13. [Ad Integration](#ad-integration)
14. [Admin Panel](#admin-panel)
15. [Auth Flow](#auth-flow)
16. [Project Structure](#project-structure)
17. [Environment Variables](#environment-variables)
18. [Deployment Guide](#deployment-guide)
19. [API Reference](#api-reference)
20. [Database Tables](#database-tables)
21. [Key Business Rules](#key-business-rules)
22. [Security](#security)

---

## How Fidge Works

```
Spin the fidget spinner
        ↓
Earn points every second while spinning
        ↓
Higher-tier skins → more points + slower energy drain
        ↓
Energy runs out? → Watch ads to refill (5 ads per batch, 2hr cooldown)
        ↓
Convert points → gems (10,000 pts = 1 💎)
        ↓
Spend gems on skins (in Shop) or spin the Bonus Wheel
        ↓
Bonus Wheel prizes: Points / Gems / $PCEDO tokens
        ↓
Withdraw $PCEDO to your ETH wallet
        ↓
Climb the leaderboard → win prize cycles every 14 days
```

---

## Tech Stack

| Layer | Technology | Hosted On |
|---|---|---|
| Frontend | Next.js 15 (App Router) · TypeScript | Vercel |
| Backend | Laravel 11 · PHP 8.4 | Railway |
| Database | MySQL 8 | Railway |
| Email | Brevo (Sendinblue) SMTP — OTP delivery | — |
| Auth | Email + OTP registration · Email + password login | — |
| Session | Custom `AuthToken` table · `X-Auth-Token` header (30-day expiry) | — |
| Admin Auth | 3-factor: password + email OTP + Google Authenticator (TOTP) | — |
| Payments | ETH (Ethereum mainnet) via injected Web3 wallet | — |
| Ads | Adsterra (now) + Google AdSense (after approval) | — |
| Build | Nixpacks on Railway · auto-migrate + auto-seed on deploy | — |

---

## App Pages

| Route | Page | Auth Required |
|---|---|---|
| `/` | Home — animated spinner, sign-in CTA | ❌ |
| `/spinner` | **Main Game** — spin to earn, energy system, watch ads | ✅ (gate shown if not logged in) |
| `/wheel` | **Bonus Wheel** — gem-based prize spins | ✅ |
| `/shop` | **Gem Shop** — browse and buy skins | ✅ (gate shown if not logged in) |
| `/profile` | **Profile** — stats, convert points, skins, withdraw $PCEDO, quests | ✅ |
| `/leaderboard` | **Leaderboard** — points + referrals rankings, prize cycle | ✅ (gate shown if not logged in) |
| `/marketplace` | **Gem Store** — buy gems with ETH via Web3 wallet | ✅ |
| `/coming-soon` | **Coming Soon** — placeholder for unreleased features | ❌ |
| `/join` | **Referral Landing** — invite link with pre-filled referral code | ❌ |
| `/fidge-admin` | **Admin Panel** — full backend management (3-factor protected) | Admin only |

---

## Spinner & Energy System

### How Spinning Works
- Drag or swipe the fidget spinner to make it spin
- Points are earned **every second** the spinner is actively rotating
- Points are synced to the backend every few seconds via `/spinner/sync`
- Session is logged on spin end via `/spinner/session-end` (triggers quest checks)
- The active skin PNG renders as the spinner graphic

### Energy
- Each user has **100% energy per skin session**
- Energy drains as you spin — lower-tier skins drain faster
- Energy is stored in the `energy_sessions` table (per user, per day)
- **Different skins have independent energy pools** — switching skins gives you fresh energy
- Skin multipliers are applied server-side on every sync — they cannot be spoofed

---

## Skin System

Skins are cosmetic upgrades that also boost gameplay performance. Higher rarity = more points earned + slower energy drain = longer sessions.

| Skin | Gem Cost | Earn Multiplier | Energy Drain | Extra Spin Time |
|---|---|---|---|---|
| Obsidian | **Free (default)** | 1.0× | 1.0× (base) | — |
| Chrome | 30 💎 | 1.0× | 1.0× | — |
| Gold | 50 💎 | **1.3×** (+30%) | 0.85× | +18% longer |
| Sapphire | 75 💎 | **1.4×** (+40%) | 0.80× | +25% longer |
| Neon | 80 💎 | **1.6×** (+60%) | 0.72× | +39% longer |
| Plasma | 100 💎 | **1.7×** (+70%) | 0.65× | +54% longer ← best |

**How multipliers work:**
- `getSkinMultiplier()` in `SpinnerController` reads `user.active_skin` on every sync call
- Points earned = `raw_points × multiplier`
- Energy drained = `raw_energy × energy_factor`
- Skin PNG images are stored in `/public/skins/` and rendered as the spinner visual

---

## Watch Ads → Refill Energy

Ads are the **only** way to refill energy mid-session (outside of switching skins or waiting for midnight reset).

### Flow
```
User taps "Watch Ad" on Spinner page
        ↓
AdModal opens fullscreen
        ↓
Real ad loads (Adsterra banner or AdSense unit)
        ↓
15-second countdown — cannot skip
        ↓
"Claim +20% Energy" button activates
        ↓
User taps Claim → POST /spinner/watch-ad
        ↓
Backend awards +20% energy, updates energy_sessions
        ↓
Modal closes — energy refills instantly
```

### Batch Rules
- **5 ads per batch** — each ad restores +20% (full batch = 100% energy restored)
- **2-hour cooldown** after completing all 5 ads in a batch
- After the 2-hour cooldown, the next batch unlocks **automatically** — no user action needed
- Backend tracks `ads_watched` and `last_ad_at` on `energy_sessions`
- Daily ad count resets at midnight

---

## Gem Economy

### Earning Gems
| Source | Amount |
|---|---|
| Convert points | 10,000 pts = 1 💎 |
| Spin Wheel prize | 1, 2, or 10 💎 |
| Referral milestones | 1 – 50 💎 |
| Coupon redemption | Variable |
| Marketplace (ETH purchase) | 50 – 4,000 💎 |

### Spending Gems
| Item | Cost |
|---|---|
| Bonus Wheel spin | 2 💎 |
| Chrome skin | 30 💎 |
| Gold skin | 50 💎 |
| Sapphire skin | 75 💎 |
| Neon skin | 80 💎 |
| Plasma skin | 100 💎 |

---

## Spin Wheel

The bonus wheel costs **2 💎 per spin** and awards randomised prizes server-side.

### Prize Segments (10 total, weighted)
| Prize | Type | Weight | Rarity |
|---|---|---|---|
| 50 Points | points | 30 | Common |
| 250 Points | points | 22 | Common |
| 500 Points | points | 18 | Common |
| 1 💎 | gems | 20 | Common |
| 2 💎 | gems | 10 | Uncommon |
| 10 💎 | gems | 3 | Rare |
| 1 $PCEDO | pcedo | 12 | Uncommon |
| 5 $PCEDO | pcedo | 5 | Uncommon |
| 10 $PCEDO | pcedo | 3 | Rare |
| 100 $PCEDO | pcedo | 1 | **Ultra Rare** |

*Higher weight = more likely. Total weight = 124.*

### Auto-Spin *(Coming Soon)*
- Options: 3×, 5×, 10×, 20×, or 50× spins
- Runs as an async loop with animation between each spin
- STOP button available at any time
- Greyed-out options if gems are insufficient

---

## Quest System

Quests reward users for engaging with different parts of the platform.

| Quest | Trigger | Reward |
|---|---|---|
| First Spin | Auto — after 1 spin session | 10 pts |
| Spin 10 Times | Auto — after 10 total sessions | 25 pts |
| Spin 50 Times | Auto — after 50 total sessions | 75 pts |
| Earn 100 Points | Auto — when spin_points ≥ 100 | 20 pts |
| Speed Demon | Auto — 30+ seconds spinning today | 15 pts |
| Daily Grind | Auto — 7 unique spin days | 50 pts |
| First Referral | Manual confirm — after 1 referral | 50 pts |
| Watch an Ad | Manual confirm — after first ad | 5 pts |
| Wheel Spin | Manual confirm — after first wheel spin | 10 pts |

- **Auto quests** — detected during `/spinner/sync` or `/spinner/session-end`
- **Manual quests** — user taps "Confirm Completion" on Profile; backend validates before awarding

---

## Referral Program

1. User shares their unique link: `https://fidge.app/join?ref=THEIRCODE`
2. Friend arrives at `/join` — referral code pre-filled in the sign-up form
3. Friend registers — ref code stored on their account
4. Referrer's `referral_count` increments when the referred user hits **10,000 total points** (counts as an "active referral")

### Gem Milestone Bonuses
| Active Referrals | Bonus Gems |
|---|---|
| 10 | 1 💎 |
| 15 | 3 💎 |
| 30 | 9 💎 |
| 50 | 15 💎 |
| 100 | 50 💎 |

---

## $PCEDO Withdrawal

### Earning $PCEDO
Exclusively from the Bonus Wheel: 1, 5, 10, or 100 $PCEDO per spin.

### Withdrawal Rules
| Rule | Value |
|---|---|
| Minimum | 100 $PCEDO |
| Fee | 0.5% deducted before sending |
| Processing | Manual by admin (24–48 hrs) |
| Network | Ethereum (ETH wallet address) |

### Fee Examples
| Withdraw | Fee (0.5%) | You Receive |
|---|---|---|
| 100 $PCEDO | 0.5 | 99.5 $PCEDO |
| 500 $PCEDO | 2.5 | 497.5 $PCEDO |
| 1,000 $PCEDO | 5.0 | 995 $PCEDO |

### User Flow
1. Profile → Withdraw tab → select amount → enter ETH wallet → submit
2. Request queued for admin review
3. Cancel any pending request → $PCEDO instantly refunded

### Admin Flow
Admin Panel → **🪙 PCEDO OUT** tab → filter Pending / Processed → **CONFIRM SENT** after sending crypto.

---

## Gem Marketplace

Buy gem packages directly with ETH via Web3 wallet. *(Launching soon — currently shows Coming Soon page)*

### Gem Packages
| Gems | USD | ETH (≈ at $3,000/ETH) |
|---|---|---|
| 50 💎 | $0.99 | ~0.00033 ETH |
| 150 💎 | $2.49 | ~0.00083 ETH |
| 350 💎 | $4.99 | ~0.00166 ETH |
| 750 💎 | $9.99 | ~0.00333 ETH |
| 1,600 💎 ⭐ | $19.99 | ~0.00666 ETH |
| 4,000 💎 | $44.99 | ~0.01499 ETH |

*ETH price set via `ETH_PRICE_USD` env var. Best value package highlighted.*

### Wallet Support
**Inside a wallet browser** (MetaMask, Trust Wallet, any EIP-1193 wallet):
- Wallet detected automatically via `window.ethereum`
- Multiple injected providers supported — picks Trust > Binance > MetaMask > first available
- Connect → approve transaction → done

**In external browser** (Chrome, Safari, etc.):
- Deep links shown for MetaMask and Trust Wallet
- **"Other Wallet"** option — works with any Web3 wallet browser via universal EIP-1193 connect

### Payment Flow
1. Select package → wallet opens with ETH amount pre-filled
2. Approve transaction in wallet
3. TX hash auto-submitted to backend
4. Admin verifies → gem coupon code issued
5. User redeems coupon → gems credited instantly

---

## Ad Integration

Ads have **one placement** in the entire app: the **Watch Ad** button on the Spinner page, tied exclusively to energy refills. No banner ads between pages, no interstitials — just the rewarded energy flow.

### Providers

| Provider | Status | Format | Watch Time | Notes |
|---|---|---|---|---|
| **Adsterra** | ✅ Active | Banner (script or iframe) | 15s | Use now — same-day approval |
| **AdSense** | ⏳ Pending | Display unit in modal | 5s | Requires custom domain + review |
| **Both** | 🎯 Long-term | Both simultaneously | 10s | Two revenue streams |

### Setting Up Adsterra

1. Sign up at [adsterra.com](https://adsterra.com) → add your site → get approved
2. Create a **Banner** zone → copy the `invoke.js` script URL
   - OR create a **Direct Link** zone → copy the direct link URL
3. Add to Vercel env vars:
```
NEXT_PUBLIC_AD_PROVIDER=adsterra
NEXT_PUBLIC_ADSTERRA_DIRECT_LINK=https://www.highperformanceformat.com/YOUR_ZONE/invoke.js
```
4. Redeploy → real ads serve immediately

> The `AdModal` auto-detects the URL format: `.js` / `invoke` URLs are injected as scripts; all other URLs are loaded in an iframe.

### Setting Up AdSense (After Approval)

1. Get a custom domain — AdSense won't approve `.vercel.app`
2. Apply at [adsense.google.com](https://adsense.google.com) — 1–2 week review
3. Create a Display Ad unit → get publisher ID + slot ID
4. Update `public/ads.txt`:
```
google.com, pub-YOUR_REAL_ID, DIRECT, f08c47fec0942fa0
```
5. Add to Vercel:
```
NEXT_PUBLIC_ADSENSE_CLIENT=ca-pub-XXXXXXXXXXXXXXXX
NEXT_PUBLIC_ADSENSE_SLOT=1234567890
```

### Switching Providers
Change one env var in Vercel and redeploy — no code changes needed:
```
NEXT_PUBLIC_AD_PROVIDER=adsterra   # now
NEXT_PUBLIC_AD_PROVIDER=adsense    # after AdSense approval
NEXT_PUBLIC_AD_PROVIDER=both       # both simultaneously (recommended)
```

---

## Admin Panel

Access at `/fidge-admin` — protected by **3-factor authentication**:
1. Admin username + password
2. OTP sent to admin email (via Brevo)
3. Google Authenticator TOTP code

### Admin Tabs

| Tab | Capabilities |
|---|---|
| 📊 **Stats** | Total users · active today · total spins · wheel spins · coupons |
| 🎟️ **Coupons** | Create (gems/points, value, max uses, expiry) · toggle active · delete |
| 💎 **Gem Orders** | View ETH purchase requests · Verify (issues coupon) · Reject |
| 🪙 **PCEDO OUT** | View withdrawal queue · Confirm Sent · Delete (refunds if pending) |
| 👥 **Users** | Search by username/email · view balances · ban/unban |
| ➕ **Create Coupon** | Create new coupon codes for distribution |

### Admin Env Vars
```
ADMIN_USERNAME=your_admin_username
ADMIN_PASSWORD=your_admin_password        # or use ADMIN_PASSWORD_HASH for bcrypt
ADMIN_USERNAME_2=second_admin             # optional second admin account
ADMIN_PASSWORD_2=second_password
ADMIN_EMAIL=fidgeappofficial@gmail.com    # OTP sent here
ADMIN_TOTP_SECRET=your_google_auth_secret # from Google Authenticator setup
BREVO_API_KEY=your_brevo_api_key
```

---

## Auth Flow

### Registration (2 steps)
```
1. POST /api/auth/register/send-otp
   Body: { email, username, password, ref_code? }
   → Sends 6-digit OTP to email via Brevo (expires 10 min)

2. POST /api/auth/register/verify
   Body: { email, username, password, otp, ref_code? }
   → Creates account, logs session, returns { token, user }
```

### Login (1 step)
```
POST /api/auth/login
Body: { email, password }
→ Returns { token, user }
```

> **Waitlist period:** Sign up is temporarily disabled. Only waitlist members (seeded via `WaitlistSeeder`) can log in. Sign up will be re-enabled at full launch.

### Token Storage
- Stored in `localStorage` as `fidge_token`
- Sent as `X-Auth-Token` header on every authenticated request
- Tokens expire after 30 days
- Logout deletes the token from the database

### Admin Auth (3-factor)
```
1. POST /api/admin/login  { username, password }
   → Returns { requires_2fa: true } (202) + sends OTP to ADMIN_EMAIL

2. POST /api/admin/login  { username, password, otp, totp_code }
   → Verifies OTP + Google Authenticator code
   → Returns { token } on success
```

---

## Project Structure

```
/
├── backend/                                Laravel 11
│   ├── app/Http/Controllers/
│   │   ├── AuthController.php              Register (OTP) + Login
│   │   ├── SpinnerController.php           Sync, session-end, watch-ad
│   │   ├── WheelController.php             Bonus wheel + prize award
│   │   ├── ProfileController.php           Profile, convert, skins, withdraw $PCEDO
│   │   ├── ShopController.php              Skin list + gem purchase
│   │   ├── LeaderboardController.php       Rankings + cycle management
│   │   ├── MarketplaceController.php       ETH gem purchases
│   │   ├── CouponController.php            Coupon redemption
│   │   └── AdminController.php             3-factor admin panel API
│   ├── app/Http/Middleware/
│   │   ├── SessionAuth.php                 X-Auth-Token → $request->user()
│   │   ├── AdminAuth.php                   X-Admin-Token validation (HMAC sig + expiry)
│   │   └── BanCheck.php                    Blocks banned users on every request
│   ├── app/Models/
│   │   ├── User.php
│   │   ├── AuthToken.php                   30-day session tokens
│   │   ├── EnergySession.php               Per-user per-day energy + ad tracking
│   │   ├── SpinLog.php                     Spin session history
│   │   ├── WheelSpin.php                   Wheel spin history + prizes
│   │   ├── GemPurchaseRequest.php          ETH marketplace purchase requests
│   │   ├── Coupon.php / CouponRedemption.php
│   │   └── PcedoWithdrawal.php             Withdrawal queue
│   ├── database/
│   │   ├── migrations/                     All table definitions
│   │   └── seeders/
│   │       ├── DatabaseSeeder.php          Calls all seeders on deploy
│   │       └── WaitlistSeeder.php          Seeds 5,955 waitlist users from CSV
│   ├── routes/api.php                      All API routes
│   └── nixpacks.toml                       Railway build + start config
│
└── frontend/                               Next.js 15
    ├── app/
    │   ├── page.tsx                        Home page
    │   ├── spinner/page.tsx                Main game — spinner + energy + ads
    │   ├── wheel/page.tsx                  Bonus wheel + auto-spin
    │   ├── shop/page.tsx                   Skin store (login-gated)
    │   ├── profile/                        Stats, convert, skins, quests, withdraw
    │   ├── leaderboard/page.tsx            Rankings (login-gated)
    │   ├── marketplace/page.tsx            ETH gem purchase (login-gated)
    │   ├── coming-soon/page.tsx            Placeholder for unreleased features
    │   ├── join/page.tsx                   Referral invite landing page
    │   └── fidge-admin/page.tsx            Admin panel (3-factor auth)
    ├── components/
    │   ├── TopHeader.tsx                   Gems badge, username, 3-dot menu
    │   ├── BottomNav.tsx                   Mobile bottom navigation
    │   ├── AuthModal.tsx                   Login modal (sign up disabled during waitlist)
    │   ├── AdModal.tsx                     Universal ad modal (Adsterra + AdSense)
    │   └── LegalModal.tsx                  Terms, Privacy Policy, Disclaimer
    ├── context/AuthContext.tsx             Global auth + energy + gems + cooldown state
    ├── lib/api.ts                          Typed API client (apiFetch + adminFetch)
    └── public/
        ├── logo.png / fidge3d.png
        ├── ads.txt                         AdSense verification
        ├── sw.js                           Service worker
        └── skins/                          Skin PNG images
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

DB_CONNECTION=mysql
DB_HOST=your_railway_mysql_host
DB_PORT=3306
DB_DATABASE=railway
DB_USERNAME=root
DB_PASSWORD=your_password

# Email — Brevo (used for user OTPs + admin OTP)
MAIL_MAILER=smtp
MAIL_HOST=smtp-relay.brevo.com
MAIL_PORT=587
MAIL_ENCRYPTION=tls
MAIL_USERNAME=your_brevo_login_email
MAIL_PASSWORD=your_brevo_smtp_key
MAIL_FROM_ADDRESS=noreply@fidge.app
MAIL_FROM_NAME=Fidge
BREVO_API_KEY=your_brevo_api_key

FRONTEND_URL=https://fidge.app
CACHE_STORE=database
SESSION_DRIVER=cookie

# Admin (3-factor)
ADMIN_USERNAME=your_admin_username
ADMIN_PASSWORD=your_admin_password
ADMIN_EMAIL=fidgeappofficial@gmail.com
ADMIN_TOTP_SECRET=your_google_authenticator_secret

# Second admin (optional)
ADMIN_USERNAME_2=second_admin
ADMIN_PASSWORD_2=second_password

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

# Ad provider: "adsterra" | "adsense" | "both" | (empty = dev placeholder)
NEXT_PUBLIC_AD_PROVIDER=adsterra

# Adsterra — Banner zone invoke.js URL or Direct Link URL
NEXT_PUBLIC_ADSTERRA_DIRECT_LINK=https://www.highperformanceformat.com/YOUR_ZONE/invoke.js

# AdSense (after approval)
NEXT_PUBLIC_ADSENSE_CLIENT=ca-pub-XXXXXXXXXXXXXXXX
NEXT_PUBLIC_ADSENSE_SLOT=1234567890
```

---

## Deployment Guide

### Railway (Backend)
```bash
# 1. Create Railway project → add MySQL plugin
# 2. Connect GitHub repo → set root directory: backend/
# 3. Add all backend env vars
# 4. Railway reads nixpacks.toml automatically

# The start command in nixpacks.toml runs on every deploy:
# php84 artisan migrate --force && php84 artisan db:seed --force && php84 artisan serve ...
# This means migrations AND seeders (including WaitlistSeeder) run automatically.

# Generate APP_KEY if needed:
php artisan key:generate --show

# Set up Google Authenticator for admin TOTP:
# Hit GET /api/admin/setup-totp once → scan QR code → add ADMIN_TOTP_SECRET to env
```

### Vercel (Frontend)
```bash
# 1. Import GitHub repo at vercel.com
# 2. Set root directory: frontend/
# 3. Framework: Next.js (auto-detected)
# 4. Add all frontend env vars
# 5. Add custom domain: fidge.app
```

### First Deploy Checklist
- [ ] Railway MySQL created and connected
- [ ] All backend env vars set (including `ADMIN_TOTP_SECRET`)
- [ ] `APP_KEY` generated and set
- [ ] Migrations ran successfully (check Railway deploy logs)
- [ ] `WaitlistSeeder` ran — 5,955 waitlist users seeded
- [ ] Vercel frontend env vars set
- [ ] Custom domain `fidge.app` connected in Vercel
- [ ] Adsterra account created, site approved, zone URL added to Vercel env
- [ ] Admin TOTP set up — scanned QR, working in Google Authenticator
- [ ] Tested admin login (password → email OTP → TOTP → dashboard)

---

## API Reference

### Auth
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/auth/register/send-otp` | ❌ | Send OTP to email |
| POST | `/auth/register/verify` | ❌ | Verify OTP → create account |
| POST | `/auth/login` | ❌ | Email + password login |
| POST | `/auth/logout` | ✅ | Delete session token |
| GET | `/auth/me` | ✅ | Current user object |

### Spinner
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/spinner/sync` | ✅ | Sync points earned + energy used |
| POST | `/spinner/session-end` | ✅ | Log session (triggers quest checks) |
| POST | `/spinner/watch-ad` | ✅ | Award +20% energy after ad |

### Profile
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/profile` | ✅ | Full profile + quests + skins |
| POST | `/profile/convert-points` | ✅ | Convert points → gems |
| POST | `/profile/set-skin` | ✅ | Equip a skin |
| POST | `/profile/withdraw-pcedo` | ✅ | Submit $PCEDO withdrawal |
| GET | `/profile/withdrawals` | ✅ | Withdrawal history |
| DELETE | `/profile/withdrawals/{id}` | ✅ | Cancel + refund pending withdrawal |
| POST | `/profile/quests/{id}/confirm` | ✅ | Confirm manual quest completion |

### Wheel
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/wheel/segments` | ❌ | Prize segment list |
| POST | `/wheel/spin` | ✅ | Spin (costs 2 💎) → returns prize |

### Shop
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/shop/skins` | ❌ | All skins (with owned flag if authed) |
| POST | `/shop/skins/{id}/purchase` | ✅ | Buy skin with gems |

### Leaderboard
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/leaderboard` | ❌ | Rankings + user referral data |

### Coupons
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/coupons/redeem` | ✅ | Redeem a coupon code |

### Marketplace
| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/marketplace/packages` | ❌ | Gem packages + ETH wallet address |
| POST | `/marketplace/initiate` | ✅ | Create ETH purchase request |
| POST | `/marketplace/submit-tx` | ✅ | Submit TX hash after payment |
| GET | `/marketplace/status` | ✅ | Purchase history + coupon codes |

### Admin (`X-Admin-Token` header required)
| Method | Endpoint | Description |
|---|---|---|
| POST | `/admin/login` | 3-factor login → get admin token |
| GET | `/admin/stats` | Dashboard numbers |
| GET | `/admin/users` | Paginated user list (searchable) |
| PATCH | `/admin/users/{id}/ban` | Ban or unban user |
| GET | `/admin/coupons` | List coupons |
| POST | `/admin/coupons` | Create coupon |
| PATCH | `/admin/coupons/{id}/toggle` | Enable/disable coupon |
| DELETE | `/admin/coupons/{id}` | Delete coupon |
| GET | `/admin/gem-requests` | ETH purchase requests |
| POST | `/admin/gem-requests/{id}/verify` | Verify + auto-issue gem coupon |
| POST | `/admin/gem-requests/{id}/reject` | Reject request |
| GET | `/admin/withdrawals` | $PCEDO withdrawal queue |
| POST | `/admin/withdrawals/{id}/confirm` | Mark withdrawal as sent |
| DELETE | `/admin/withdrawals/{id}` | Delete (refunds $PCEDO if pending) |

---

## Database Tables

| Table | Purpose |
|---|---|
| `users` | Accounts: email, username, password (bcrypt), points, gems, pcedo_earned, active_skin, referral_code, is_banned, email_verified |
| `auth_tokens` | Session tokens — 30-day expiry, one per login |
| `skins` | Skin catalog: name, rarity, gem_cost, shade, image_url, multiplier, is_default |
| `user_skins` | Pivot: which skins each user owns |
| `quests` | Quest catalog: title, type, threshold, reward_points |
| `user_quests` | Pivot: completion status per user |
| `energy_sessions` | Per-user per-day: energy%, ads_watched, last_ad_at (2hr cooldown tracking) |
| `spin_logs` | Spin session history: duration, points earned, energy used |
| `wheel_spins` | Wheel spin history: segment_index, prize_type, prize_value, gems_spent |
| `leaderboard_cycles` | Prize cycle periods (14-day default) |
| `leaderboard_entries` | Points per user per cycle |
| `coupons` | Coupon codes: type (gems/points), value, max_uses, used_count, expiry_date, active |
| `coupon_redemptions` | Redemption audit log |
| `gem_purchase_requests` | ETH marketplace: tx_hash, gem_amount, eth_amount, status, coupon_code |
| `pcedo_withdrawals` | Withdrawal queue: amount, fee, wallet_address, status, processed_at |

---

## Key Business Rules

| Rule | Value |
|---|---|
| Points → Gems | 10,000 pts = 1 💎 |
| Bonus Wheel cost | 2 💎 per spin |
| Ads per batch | 5 |
| Energy per ad | +20% |
| Ad batch cooldown | **2 hours** (auto-resets — no user action needed) |
| Active referral threshold | 10,000 total points earned |
| Min $PCEDO withdrawal | 100 $PCEDO |
| Withdrawal fee | 0.5% of gross |
| Auth token expiry | 30 days |
| OTP expiry | 10 minutes |
| Admin token expiry | 8 hours |
| Leaderboard cycle | 14 days (configurable via `LEADERBOARD_CYCLE_DAYS`) |
| ETH network | Ethereum mainnet only |
| Default free skin | Obsidian only |
| Waitlist users seeded | 5,955 (from `testers_rows_fox.csv`) |

---

## Security

- **All game logic is server-side** — points, prizes, and energy are calculated on the backend. The frontend reports what happened; the server decides what it's worth
- **Skin multipliers** are read from `user.active_skin` server-side on every sync — client cannot spoof a higher multiplier
- **Wheel prizes** are determined server-side with weighted random — client only receives the result index
- **Gem deductions** for wheel spins happen in a database transaction — no double-spend
- **TX hash deduplication** — the same ETH transaction hash cannot be submitted twice
- **Admin 3-factor auth** — password + email OTP (10-min expiry) + Google Authenticator TOTP. Admin tokens use HMAC-SHA256 signature with `APP_KEY` and expire after 8 hours
- **Banned users** are rejected at middleware level (`BanCheck`) on every request
- **OTP codes** are single-use and expire after 10 minutes
- **Passwords** are bcrypt hashed — never stored in plaintext
- **CORS** — only the `FRONTEND_URL` env var is whitelisted
- **Ban enforcement** — banned users forfeit all pending rewards and cannot access any authenticated endpoint

---

*Built by the Fidge team · fidge.app · [@FIDGE_APP](https://linktr.ee/cedomisofficial)*
