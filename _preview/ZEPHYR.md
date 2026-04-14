# ☕ Zephyr Coffee Co.

Welcome to Zephyr. You just joined as their first real data engineer. Here's what you've walked into.

---

## The Company

Zephyr Coffee Co. opened in 2019 with one store in Portland. By 2024 they have **200+ locations** across the US — kiosks in office buildings, full cafés in downtowns, a few drive-thrus in suburbs. They grew fast. Their data grew faster, and nobody was paying attention.

Now they have dashboards nobody trusts, revenue numbers that don't match finance, and a loyalty app that sends nested JSON nobody can parse. That's where you come in.

---

## Your New Colleagues (who will keep pinging you)

| Person | Role | What they want from you |
|---|---|---|
| **Priya** | PM, Growth | Asks Monday morning, needs by Tuesday. Relentless but nice. |
| **Marcus** | CFO | Paranoid about revenue. Finds bugs everyone missed. If Marcus questions a number, you'd better be sure. |
| **Jen** | Marketing Analyst | Cohorts, retention, churn. Loves window functions even if she doesn't know it. |
| **Dev** | Store Ops Lead | Runs the 200 stores. Hates the loyalty app. Will yell about it. |
| **You** | New Data Engineer | Hired to make sense of the mess. Good luck. |

Every notebook in this repo is a **request from one of these people**. You get a Slack message, you get the data, you solve the problem.

---

## The Tables

### `stores` — ~200 rows
Basic metadata about each location.

| column | type | notes |
|---|---|---|
| `store_id` | int | primary key |
| `city` | string | |
| `region` | string | West / Central / East |
| `store_type` | string | `kiosk` / `cafe` / `drive_thru` |
| `opened_date` | date | |
| `manager_name` | string | |

### `products` — ~80 rows
The menu. **SKUs are stable, but product names get rebranded occasionally** (e.g. `"Iced Latte"` → `"Zephyr Latte"` in 2023). Always join on SKU, never on name.

| column | type | notes |
|---|---|---|
| `sku` | string | primary key |
| `name` | string | may change over time |
| `category` | string | `coffee` / `pastry` / `merch` / `cold_brew` |
| `price` | double | current price |
| `cost` | double | COGS |

### `transactions` — ~500K rows (the main fact table)
Every sale. **⚠️ This table is cursed. Read the Drama section below before touching it.**

| column | type | notes |
|---|---|---|
| `tx_id` | string | supposed to be unique. isn't always. |
| `store_id` | int | |
| `ts` | timestamp | **three different formats in the raw data** |
| `customer_id` | int | **nullable — walk-ins don't identify** |
| `payment_method` | string | `card` / `cash` / `mobile` / `gift_card` |
| `total_amount` | double | **can be negative (returns)** |

### `transaction_items` — ~1.5M rows
Line items per transaction.

| column | type | notes |
|---|---|---|
| `tx_id` | string | foreign key to `transactions` |
| `sku` | string | foreign key to `products` |
| `quantity` | int | |
| `unit_price` | double | |
| `discount_amount` | double | |

### `customers` — ~50K rows
Loyalty members only. Walk-ins aren't in here.

| column | type | notes |
|---|---|---|
| `customer_id` | int | primary key |
| `signup_date` | date | |
| `email` | string | **some emails are duplicated across customer_ids** |
| `tier` | string | `bronze` / `silver` / `gold` — **current state, not historical** |
| `home_store_id` | int | |

### `loyalty_events` — ~200K rows (nested)
Events from the loyalty app. The `payload` column is a **struct that varies by event type**.

| column | type | notes |
|---|---|---|
| `customer_id` | int | |
| `event_type` | string | `points_earned` / `redeemed` / `tier_changed` |
| `ts` | timestamp | |
| `payload` | struct | `{points: int}` OR `{from_tier: str, to_tier: str}` OR `{reward_sku: str}` |

---

## 🎭 The Drama (what's actually wrong with the data)

You'll discover most of these the hard way, but here's the honest list:

### 1. The 2023 Duplicate Incident 💀
On **Aug 14–15, 2023**, the POS system double-wrote every transaction during a Kafka outage. You'll find duplicate `tx_id`s from those two days. **Dedupe or die.** Marcus still brings this up in retros.

### 2. Three Timestamp Formats 🕰️
- Old stores (pre-2021) log **ISO strings**: `"2023-07-14T14:22:01Z"`
- Newer stores log **epoch seconds**: `1689345721`
- A batch of refurbished POS terminals logs `"MM/DD/YYYY HH:MM"`: `"07/14/2023 14:22"`

Fun. You'll need to handle all three to get a unified timeline.

### 3. Walk-ins Are Invisible 👻
About **60% of transactions have `customer_id = NULL`** because walk-ins don't use the loyalty app. If you `INNER JOIN` transactions with customers to enrich data, you'll accidentally hide most of Zephyr's revenue. **Marcus will notice within an hour.** Use left joins.

### 4. Store 042 Is a Ghost 👻
Store 042's POS died for **all of February 2024**. No transactions. It's not a bug in your pipeline — it's the data. Anyone building month-over-month comparisons needs to either exclude store 042 or flag it.

### 5. Returns Exist 🔄
`total_amount` can be negative. Naively filtering `total_amount > 0` to "clean the data" will silently overstate net revenue. Think about what you actually want before filtering.

### 6. Nested Loyalty Events 🎁
`payload.points` exists for `points_earned` events. `payload.from_tier`/`to_tier` exist for `tier_changed` events. `payload.reward_sku` exists for `redeemed` events. You'll need `selectExpr` or careful `explode` handling.

### 7. Tier Is "Current State", Not Historical ⏳
A customer might be `"gold"` today but their transaction last year was while they were `"silver"`. There's no tier-at-time-of-purchase column. Marcus once asked for "revenue by tier over time" and the old DE team quit. You'll need to reconstruct tier history from `loyalty_events`.

### 8. Duplicate Emails 📧
Some customers have **multiple `customer_id`s for the same email** (signup form had no dedup check until 2022). When Jen asks for "unique customers", she means unique *people*, not unique IDs. Dedup on email, keep the earliest signup.

---

## How to use this repo

Every notebook is a **Slack message from a colleague** → **the relevant Zephyr data** → **you solve it**. The concept (window functions, joins, whatever) gets taught *through* solving Zephyr's actual mess, not through abstract toy examples.

Difficulty badges:
- 🟢 **Basics** — you're new, follow the walkthrough
- 🟡 **Intermediate** — concept + some "figure it out" space
- ⚡ **Advanced** — internals, optimization, "why" not just "how"

Every notebook ends with a 🏆 **Boss Level** — a harder variant for people who want to push themselves.

---

Welcome aboard. Priya's already pinging you.
