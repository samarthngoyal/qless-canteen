# QLess Canteen

> **Digital canteen ordering system for college students** — skip the queue, order smart, pick up fast.

A mobile-first web application replacing paper bills and physical food coupons with a real-time digital ordering workflow. Students browse menus, place orders with UPI or cash, track live preparation status with ETAs, and collect food when it's ready — no waiting in line.

---

## Features

### Student-Facing
- **OTP → password onboarding** — SAP ID lookup, demo OTP verification, one-time password set; subsequent logins use the password directly
- **Multi-counter menu** — 8 food counters, each with their own menu and availability flags
- **Cart** — add items from a single counter, adjust quantities, clear cart; cross-counter mixing blocked
- **Payment method selection** — choose UPI or Cash at checkout
- **Order tracking** — live per-item status (PREPARING → READY), ETA countdown, running-late indicator
- **Pending & completed order history** — separate views for active and past orders

### Staff-Facing
- **Counter-scoped dashboard** — staff only see orders for their own counter; live via Supabase Realtime
- **Per-item ready marking** — tap each item individually as it's prepared
- **Auto-promotion** — when the last item is marked READY, the order automatically becomes READY_FOR_PICKUP
- **Order completion** — staff verify the student's physical ID, then mark the order COMPLETE (triggers audit log)

### Public Display Screen
- **`/display/[counterId]`** — full-screen, dark-themed TV board for each counter
- **Real-time updates** — orders appear and change state without a refresh
- **Running-late detection** — ETA-past orders are highlighted for immediate attention
- **Two columns** — PREPARING (yellow) and READY FOR PICKUP (glowing green)

### System-Wide
- **Rush-hour ETA engine** — 2× prep-time multiplier during 10–11 AM and 1–2 PM; computed at order placement
- **Audit log** — every state change recorded with actor type, actor ID, and timestamp
- **JWT session management** — `httpOnly` cookies, 7-day expiry, separate tokens for students and staff
- **Row-Level Security** — all Supabase tables have RLS enabled; API routes use service-role key exclusively

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router, TypeScript) |
| Database | Supabase (PostgreSQL) |
| Realtime | Supabase Realtime (Postgres CDC) |
| Auth | Custom JWT via `jose` + `bcryptjs` |
| Styling | Tailwind CSS + shadcn/ui |
| Notifications | Sonner (toast library) |
| Hosting | Vercel (recommended) |

---

## Quick Start

### 1. Supabase Setup

1. Create a free project at [https://supabase.com](https://supabase.com)
2. Go to **SQL Editor** and run each migration in order:
   - `supabase/migrations/001_schema.sql` — all tables, enums, RLS, realtime
   - `supabase/migrations/002_payment_method.sql` — `payment_method` column on orders
   - `supabase/migrations/003_prep_time.sql` — `prep_time_minutes` on menu items, `estimated_ready_at` on orders
3. Run `supabase/seed.sql` — populates all counters, menu items, students, and staff accounts
4. Go to **Project Settings → API** and copy:
   - `Project URL`
   - `anon public` key
   - `service_role` key

### 2. Environment Variables

Create `.env.local` in the project root:

```env
NEXT_PUBLIC_SUPABASE_URL=https://your-project-id.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
JWT_SECRET=any_random_32_char_string
```

### 3. Run Locally

```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000)

---

## Demo Instructions

### Flow 1: Student Orders Food

1. Open `http://localhost:3000/login` in a **mobile-sized** browser window (DevTools → Toggle device toolbar)
2. Enter SAP ID: `71725123018`
3. Enter OTP: **`123456`** (demo OTP, always works)
4. Set a password (anything 6+ chars)
5. Browse **Sandwich Counter** → Add items to cart
6. Go to **Cart** → select a payment method (UPI or Cash) → Place Order
7. Go to **Pending Orders** → see the order with PREPARING status and ETA

### Flow 2: Staff Processes Order

1. In a new tab, open `http://localhost:3000/staff/login`
2. Login: `sandwich_staff` / `password123`
3. The new order appears on the dashboard in real-time
4. Click the order → Mark each item as Ready individually
5. After the last item, the order auto-promotes to **READY_FOR_PICKUP**

### Flow 3: Display Screen

1. In another tab, open `http://localhost:3000/display/11111111-1111-1111-1111-111111111111`
   - (That UUID is the Sandwich Counter — see the seed file for other counter IDs)
2. Watch orders appear and transition states without refreshing
3. PREPARING = dark card with amber tag
4. READY_FOR_PICKUP = glowing green card

### Flow 4: Complete Order

1. Student goes to the counter and shows their ID card
2. Staff verifies name + SAP ID on screen
3. Staff clicks **Mark Order Complete**
4. Order moves to the student's Completed Orders history

---

## Demo Accounts

### Students
| SAP ID | Name |
|--------|------|
| 71725123018 | Samarth Goyal |
| 71725123019 | Ayushmaan Goyal |
| 71725123021 | Ananya Gupta |
| 71725123029 | Anushka Kavle |

> **OTP for all students:** `123456`
> **Password:** Set on first login (anything 6+ chars)

### Staff
| Username | Password | Counter |
|----------|----------|---------|
| sandwich_staff | password123 | Sandwich Counter |
| chinese_staff | password123 | Chinese Counter |
| juice_staff | password123 | Juice Counter |
| southindian_staff | password123 | South Indian Counter |
| snacks_staff | password123 | Snacks Counter |
| bakery_staff | password123 | Bakery Counter |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser / Client                         │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │  Student App  │  │  Staff App   │  │    Display Screen     │ │
│  │  /app/**     │  │  /staff/**   │  │    /display/[id]      │ │
│  │  (mobile-    │  │  (desktop-   │  │    (TV board,         │ │
│  │   first UI)  │  │   optimised) │  │     public, no auth)  │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────┘ │
│         │                 │                       │             │
│         └────────── Supabase Realtime ────────────┘             │
│                    (postgres_changes CDC)                        │
└─────────────────────┬───────────────────────────────────────────┘
                      │ fetch() / REST
┌─────────────────────▼───────────────────────────────────────────┐
│                    Next.js API Routes  (app/api/**)              │
│                                                                 │
│  Auth                Orders            Staff            Public   │
│  /auth/student/*     /orders           /staff/orders    /counters│
│  /auth/staff/*       /orders/[id]      /staff/orders/[id]        │
│  /auth/logout        /display/[id]     .../complete              │
│                                        .../items/[itemId]        │
│                                                                 │
│  All routes use SUPABASE_SERVICE_ROLE_KEY (never exposed)       │
│  Student/staff identity verified via httpOnly JWT cookie         │
└─────────────────────┬───────────────────────────────────────────┘
                      │ supabase-js (admin client)
┌─────────────────────▼───────────────────────────────────────────┐
│                         Supabase                                │
│                                                                 │
│  PostgreSQL Database          Realtime Engine                   │
│  (7 tables, RLS enabled)      (orders + order_items CDC)        │
└─────────────────────────────────────────────────────────────────┘
```

### Authentication Flow

```
Student first visit:
  /login  →  SAP ID entry  →  POST /api/auth/student/check
          →  OTP entry     →  POST /api/auth/student/verify-otp
          →  Set password  →  POST /api/auth/student/set-password
          →  JWT cookie set (qless_student_session, 7 days)
          →  /app (home)

Student returning:
  /login  →  SAP ID + password  →  POST /api/auth/student/login
          →  JWT cookie set     →  /app

Staff login:
  /staff/login  →  username + password  →  POST /api/auth/staff/login
               →  JWT cookie set (qless_staff_session, 7 days)
               →  /staff/dashboard
```

### Order Lifecycle

```
Student places order (POST /api/orders)
  │
  ▼
Order created  →  status: PREPARING
                  estimated_ready_at: now + Σ(prep_time × qty) [× 2 if rush hour]
  │
  ▼ (Staff marks each item READY via PATCH /api/staff/orders/[id]/items/[itemId])
All items READY?
  │
  ├── No  →  item.status = READY, order remains PREPARING
  │
  └── Yes →  order.status = READY_FOR_PICKUP  (auto-promoted)
               │
               ▼ (Staff verifies student ID, clicks Complete)
             order.status = COMPLETE
             order.completed_at = now
             audit_log entry created
```

### Rush-Hour ETA Engine

```
Rush windows (Asia/Kolkata):
  10:00 – 11:00 AM  →  multiplier: 2×
  01:00 – 02:00 PM  →  multiplier: 2×
  All other times   →  multiplier: 1×

ETA = placement_time + ROUND(Σ(prep_time_minutes × quantity) × multiplier) minutes

Example (non-rush):
  Veg Grilled Sandwich × 1  →  7 min
  Cold Coffee × 1           →  4 min
  Total ETA                 →  11 min from placement

Same order during rush (1 PM):
  Total ETA                 →  22 min from placement
```

---

## Database Schema

```sql
-- ENUMS
order_status      : PREPARING | READY_FOR_PICKUP | COMPLETE
order_item_status : PREPARING | READY
payment_method    : UPI | CASH

-- TABLES & RELATIONSHIPS

counters
  id               uuid PK
  name             text
  description      text
  display_order    integer
  active           boolean
  created_at       timestamptz

students
  id                      uuid PK
  sap_id                  text UNIQUE
  full_name               text
  phone_number            text
  password_hash           text          -- bcrypt, null until set
  first_login_completed   boolean
  created_at              timestamptz

staff_users
  id             uuid PK
  username       text UNIQUE
  password_hash  text                   -- bcrypt
  full_name      text
  counter_id     uuid FK → counters(id) ON DELETE SET NULL
  role           text                   -- 'staff'
  created_at     timestamptz

menu_items
  id                uuid PK
  counter_id        uuid FK → counters(id) ON DELETE CASCADE
  name              text
  description       text
  price             numeric(10,2)
  is_available      boolean
  image_url         text
  prep_time_minutes integer             -- base prep time (non-rush)
  created_at        timestamptz

orders
  id                  uuid PK
  order_number        text UNIQUE       -- human-readable (e.g. ORD-0001)
  student_id          uuid FK → students(id) ON DELETE RESTRICT
  counter_id          uuid FK → counters(id) ON DELETE RESTRICT
  status              order_status      -- PREPARING | READY_FOR_PICKUP | COMPLETE
  payment_method      payment_method    -- UPI | CASH
  placed_at           timestamptz
  completed_at        timestamptz
  estimated_ready_at  timestamptz       -- rush-adjusted ETA
  total_amount        numeric(10,2)

order_items
  id                    uuid PK
  order_id              uuid FK → orders(id) ON DELETE CASCADE
  menu_item_id          uuid FK → menu_items(id) ON DELETE SET NULL
  item_name_snapshot    text            -- captured at order time (immutable)
  item_price_snapshot   numeric(10,2)  -- captured at order time (immutable)
  quantity              integer CHECK (quantity > 0)
  status                order_item_status -- PREPARING | READY
  ready_at              timestamptz
  created_at            timestamptz

audit_logs
  id             uuid PK
  order_id       uuid FK → orders(id) ON DELETE CASCADE
  order_item_id  uuid FK → order_items(id) ON DELETE CASCADE
  actor_type     text                   -- 'student' | 'staff' | 'system'
  actor_id       uuid
  action         text                   -- e.g. ORDER_PLACED, ITEM_READY, ORDER_COMPLETE
  timestamp      timestamptz
  notes          text

-- INDEXES
idx_orders_student_id    on orders(student_id)
idx_orders_counter_id    on orders(counter_id)
idx_orders_status        on orders(status)
idx_order_items_order_id on order_items(order_id)
idx_menu_items_counter_id on menu_items(counter_id)

-- REALTIME
supabase_realtime publication: orders, order_items
```

---

## API Reference

### Student Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/student/check` | Check if SAP ID is registered |
| POST | `/api/auth/student/verify-otp` | Verify SAP ID + OTP |
| POST | `/api/auth/student/set-password` | Set password (first login) |
| POST | `/api/auth/student/login` | Password-based login |
| GET | `/api/auth/student/me` | Get current student session |

### Staff Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/staff/login` | Username + password login |
| GET | `/api/auth/staff/me` | Get current staff session |
| POST | `/api/auth/logout` | Clear session cookie |

### Orders (Student)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/orders?status=active` | Active orders (PREPARING or READY_FOR_PICKUP) |
| GET | `/api/orders?status=COMPLETE` | Completed order history |
| POST | `/api/orders` | Place a new order |
| GET | `/api/orders/[id]` | Single order detail |

### Orders (Staff)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/staff/orders` | Active orders for staff's counter |
| GET | `/api/staff/orders/[id]` | Single order detail |
| PATCH | `/api/staff/orders/[id]/items/[itemId]` | Mark item as READY |
| PATCH | `/api/staff/orders/[id]/complete` | Mark order COMPLETE |

### Public
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/counters` | List all active counters |
| GET | `/api/counters/[id]/menu` | Menu items for a counter |
| GET | `/api/display/[counterId]` | Active orders for display screen |

---

## Project Structure

```
app/
  api/               ← Next.js API routes (server-side)
  app/               ← Student-facing pages (mobile UI)
    cart/
    counter/[counterId]/
    counters/
    orders/[orderId]/
    orders/completed/
    orders/pending/
  display/[counterId]/   ← Public TV display board
  login/                 ← Student login & onboarding
  set-password/
  verify-otp/
  staff/                 ← Staff-facing pages
    dashboard/
    login/
    orders/[id]/

components/
  BottomNav.tsx          ← Student app bottom navigation
  CartContext.tsx        ← Client-side cart state (React context)
  LoadingSkeletons.tsx   ← Skeleton loaders for all list views
  StatusBadge.tsx        ← Coloured order/item status pill
  ui/                    ← shadcn/ui primitives

lib/
  auth/session.ts        ← JWT create/read/clear helpers
  prep-time.ts           ← ETA calculation & rush-hour logic
  types.ts               ← Shared TypeScript interfaces
  utils.ts               ← formatCurrency, formatDate, formatTime
  supabase/
    client.ts            ← Browser Supabase client (anon key)
    server.ts            ← Server Supabase admin client (service role)

supabase/
  migrations/
    001_schema.sql       ← Tables, enums, RLS, realtime
    002_payment_method.sql
    003_prep_time.sql
  seed.sql               ← Demo data (counters, menus, users)
```

---

## Available Counters

| Counter | UUID (seed) | Sample Items |
|---------|-------------|--------------|
| Sandwich Counter | `11111111-...` | Veg Grilled, Paneer, Club Sandwich |
| Chinese Counter | `22222222-...` | Veg Noodles, Manchurian, Momos |
| Juice Counter | `33333333-...` | Orange Juice, Cold Coffee, Mango Shake |
| South Indian Counter | `44444444-...` | Masala Dosa, Idli, Medu Vada |
| Snacks Counter | `55555555-...` | Samosa, Pav Bhaji, Pani Puri |
| Bakery Counter | `66666666-...` | Veg Puff, Chocolate Muffin |
| Fast Food Counter | `77777777-...` | Burgers, Wraps, Fries |
| Beverages Counter | `88888888-...` | Hot & cold drinks, Shakes |

### Display Screens (fixed UUIDs in seed data)
| Counter | URL |
|---------|-----|
| Sandwich Counter | `/display/11111111-1111-1111-1111-111111111111` |
| Chinese Counter | `/display/22222222-2222-2222-2222-222222222222` |
| Juice Counter | `/display/33333333-3333-3333-3333-333333333333` |

---

## Architecture

```
qless-canteen/
├── app/                        # Next.js App Router
│   ├── login/                  # Student login
│   ├── verify-otp/             # OTP verification
│   ├── set-password/           # First-time password setup
│   ├── app/                    # Student app (protected)
│   │   ├── counters/           # Counter listing
│   │   ├── counter/[id]/       # Menu items
│   │   ├── cart/               # Cart page
│   │   └── orders/             # Pending + completed + detail
│   ├── staff/                  # Staff portal
│   │   ├── login/
│   │   ├── dashboard/
│   │   └── orders/[id]/
│   ├── display/[counterId]/    # Kitchen display screen
│   └── api/                    # API routes
│       ├── auth/               # Student + staff auth
│       ├── counters/           # Counter + menu APIs
│       ├── orders/             # Student order CRUD
│       ├── staff/orders/       # Staff order management
│       └── display/            # Public display data
├── components/                 # Shared UI components
│   ├── CartContext.tsx          # Cart state + one-counter rule
│   ├── BottomNav.tsx            # Student bottom navigation
│   ├── StatusBadge.tsx          # Color-coded status badges
│   ├── LoadingSkeletons.tsx     # Loading states
│   └── ui/                     # shadcn/ui primitives
├── lib/
│   ├── types.ts                 # TypeScript interfaces
│   ├── utils.ts                 # cn, formatCurrency, formatDate
│   ├── auth/session.ts          # JWT session management
│   └── supabase/                # Supabase clients
├── middleware.ts                # Route protection
└── supabase/
    ├── migrations/001_schema.sql
    └── seed.sql
```

### Key Design Decisions

- **JWT cookies** for sessions (not Supabase Auth) — keeps auth simple and fully custom
- **Service role key in API routes only** — client only has anon key for realtime subscriptions
- **CartContext enforces one-counter rule** on the client — server double-checks on order placement
- **Auto-promotion logic** server-side: when last item is marked READY, the order status immediately becomes READY_FOR_PICKUP
- **Supabase Realtime** on `orders` and `order_items` tables for live updates across all views

---

## Business Rules Enforced

| Rule | Where enforced |
|------|----------------|
| One counter per order | CartContext (client) + `/api/orders` POST (server) |
| No cancellation | No cancel API route exists |
| No edit after placement | No edit API route exists |
| Item READY → auto order promotion | `/api/staff/orders/[id]/items/[itemId]` |
| Complete only when READY_FOR_PICKUP | `/api/staff/orders/[id]/complete` + UI button disabled |

---

## Test Checklist

- [ ] Student can log in with SAP ID + OTP `123456`
- [ ] Student sets password on first login
- [ ] Student can browse counters and menu
- [ ] Items from different counter show error toast
- [ ] Cart shows correct totals
- [ ] Order appears on staff dashboard in real-time
- [ ] Order appears on display screen in real-time
- [ ] Staff can mark items READY one by one
- [ ] Order auto-promotes to READY_FOR_PICKUP when all items ready
- [ ] Display screen turns green for READY_FOR_PICKUP orders
- [ ] Complete button is greyed out when order is PREPARING
- [ ] Complete button works when order is READY_FOR_PICKUP
- [ ] Completed order moves to student's History tab
- [ ] Student can place multiple simultaneous orders
