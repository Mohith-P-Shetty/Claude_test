# Pantry App — Build Spec

Single source of truth. Living document — update it when a decision changes, and keep it in `/docs/SPEC.md` in the repo so it's always in context.

**Product in one line:** a shared pantry list for three flatmates, that tells us what we can cook right now.

---

## 0. How to use this document

Read sections 1–3 before writing any code. Section 1 contains hard constraints that override any default you'd otherwise reach for. Build strictly in the order given in section 5 — do not scaffold later features early.

When a detail isn't specified here, choose the simpler option and note the choice in section 10.

---

## 1. Hard constraints

These are not preferences. Violating any one of them is a build failure.

1. **$0/month, no payment card on file anywhere.** If a step asks for card details, stop and flag it.
2. **No third-party recipe API at runtime.** Recipe data lives in our own database. See section 4.
3. **The Supabase `service_role` key never reaches the browser.** No `NEXT_PUBLIC_` prefix on any secret. All database access goes through server-side route handlers.
4. **No quantities or units on pantry items.** This is deliberate — see section 6.
5. **No user accounts.** One shared passcode, one shared pantry.
6. **Never write real secret values into any committed file**, including `.env.example`, docs, or code comments.

---

## 2. Non-goals for v1

Quantities, expiry dates, barcode scanning, shopping lists, meal plans, cost splitting, user accounts, push notifications, offline sync.

Do not build these. Do not leave scaffolding for them.

---

## 3. Core design principle

The app owns its recipe data.

Third-party recipe APIs get metered, repriced, or withdrawn, and their terms usually restrict how long you may cache responses — so the data is never yours. Their coverage also skews Western, which doesn't match this flat's actual shelf.

Instead, recipes are rows in our Postgres. They arrive three ways: URL import (F5), manual entry (F6), and a one-time seed script (F7). Once stored they're permanent, free to query, and can't be taken away.

---

## 4. Feature spec

### F1 — Pantry list

**Action**
- View every item currently in the flat's pantry, grouped by category (produce, dairy, grain, spice, other).
- Tap an item to cycle its state: `in` → `low` → removed.
- Shared across all three of us. No per-person pantries.

**Look**
- Single scrolling list, category headers, sticky add-bar pinned to the bottom.
- `low` items render dimmed with a small marker — still present, visually distinct.
- Each item shows who added it, small and secondary. Enough to answer "whose eggs are these".

**Depends on** `items` table; F3 normalizer.

---

### F2 — Bulk add

**Action**
- One textarea. Paste or type a shopping haul, comma or newline separated.
- One submit → one server round-trip → all items normalized and inserted.
- A duplicate of an existing `in` item is silently ignored. A duplicate of a `low` item flips it back to `in`.

**Look**
- Expands from the add-bar. Placeholder shows the expected format.
- After submit, a confirmation strip listing what went in and anything the normalizer couldn't match.

**Depends on** F3.

**Why it exists:** if adding 14 items from a shop takes 42 taps, the app dies in week two. This feature decides whether the product survives.

---

### F3 — Ingredient normalizer

**Action**
- Maps free text ("2 cloves garlic, minced", "Garlic", "lehsun") to one canonical ingredient (`garlic`).
- Strips leading quantities, units, and trailing prep instructions, lowercases, then looks up the alias table.
- Runs server-side against local data. **Zero network calls.**
- Unmatched input is still stored, flagged, and surfaced in a review list so aliases can be added over time.

**Look**
- Invisible in normal use. One plain screen listing unmatched terms with a "map this to…" action.

**Depends on** `ingredients` and `ingredient_aliases` tables.

**Design note:** the alias table is the app's real long-term asset. It's what makes Indian pantry staples work where a generic API wouldn't.

---

### F4 — Cook (what can I make)

**Action**
- One button. Sends the pantry's canonical ingredient IDs, returns recipes ranked by fewest missing ingredients.
- Staples (salt, water, oil, sugar) are assumed present and never counted missing.
- Ingredients marked optional in a recipe are never counted missing.
- `low` items count as present in v1.
- Results show: title, missing count, and which ingredients are missing.

**Look**
- Result cards. The "missing 0" group sits first and is visually distinct — that group is the entire point of the app.
- Tap a card → full recipe: ingredients (owned vs missing marked), instructions, source attribution.

**Depends on** `recipes`, `recipe_ingredients`; F3 for consistent canonical names on both sides of the match.

---

### F5 — Recipe import by URL

**Action**
- Paste a recipe URL. Server fetches the page, reads its schema.org Recipe markup (JSON-LD first, microdata as fallback), extracts title, ingredients, instructions, servings, and stores them.
- Ingredient lines pass through F3 on the way in.
- On parse failure, degrade to the F6 form pre-filled with whatever was extractable.

**Look**
- Single input + Import button on the Recipes screen. Show a parsed preview before saving so garbage doesn't get committed.

**Depends on** server-side fetch + `cheerio`. No API key.

**Why it exists:** this is the escape hatch from vendor dependency. Any recipe found anywhere becomes permanently ours.

---

### F6 — Manual recipe entry

**Action**
- Form: title, ingredient lines (one per row, each markable optional), instructions, servings.
- Saved identically to imported recipes; participates in matching the same way.

**Look** Plain form. Add/remove ingredient rows.

**Depends on** F3.

**Why it exists:** family recipes exist in no database on earth. Without this the app never suggests anything we actually eat.

---

### F7 — Seed import (developer-run, one time)

**Action**
- A script in `scripts/`, not a UI. Loads a CC-licensed recipe dump so the app isn't empty on day one.
- **Filter the import**: only keep recipes where every non-optional ingredient resolves to a known canonical name. Two hundred matchable recipes beat a hundred thousand unmatchable ones.
- Store `license` and `attribution` per recipe.

---

### F8 — Access gate

**Action**
- Single shared passcode. Correct entry sets a signed, httpOnly, long-lived session cookie. No accounts, no email, no reset flow.
- "Who added this" comes from a display name picked once per device and stored in that cookie.

**Look** One screen, one field, one button. Seen roughly once per device, ever.

---

## 5. Build order

Build and verify each step before starting the next.

1. **F3 normalizer** — everything depends on it. Unit-test it against a list of messy real inputs first.
2. **F1 + F2 pantry** — useful on its own. Stop here and actually use it for a few days if possible.
3. **F6 + F4** — the loop closes. Tag `v1` here.
4. **F5 URL import**
5. **F7 seed import**
6. **F8 gate** — must be done before the URL goes to anyone.

Ship 1–3 before touching 4. A pantry list that works beats a recipe engine that half-works.

---

## 6. Stack and technical decisions

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js (App Router) + TypeScript | One repo for UI and server routes; route handlers keep secrets server-side |
| UI | Tailwind CSS | No design system to maintain for a two-screen app |
| Database | Supabase Postgres, free plan | Free Postgres with a web SQL editor; relational matching is the core of F4 |
| Hosting | Vercel Hobby | Free, git-push deploys, serverless functions included |
| Recipe data | Self-hosted in our own DB | No vendor, no quota, no repricing risk |
| HTML parsing | `cheerio` | Reads schema.org JSON-LD out of imported pages (F5) |
| Version control | Git + GitHub, private repo | Vercel deploys from it |

**Why no quantities on items:** nobody updates "500g rice → 380g" after cooking, so the field would be permanently wrong and actively misleading. Presence plus a `low` flag is the honest model.

**Provider abstraction:** keep all recipe access behind `lib/recipes.ts`, exporting `findByPantry(ingredientIds: number[])` and `importFromUrl(url: string)`. If storage or import strategy ever changes, one file changes.

**Licensing hygiene:** store `source_url`, `license`, and `attribution` on every recipe, and render attribution in the recipe view. This app is private among three people, but recording provenance costs nothing now.

---

## 7. Data model

Put this in `supabase/schema.sql` — it is the source of truth, committed to git, run by hand in the Supabase SQL editor.

```sql
-- canonical vocabulary
create table ingredients (
  id         bigserial primary key,
  name       text unique not null,        -- 'garlic', 'toor dal'
  category   text,                        -- produce | dairy | grain | spice | other
  is_staple  boolean not null default false  -- salt, water, oil: never counted missing
);

create table ingredient_aliases (
  alias         text primary key,         -- 'lehsun', 'garlic cloves'
  ingredient_id bigint not null references ingredients(id) on delete cascade
);

-- the flat's pantry
create table items (
  id            bigserial primary key,
  display_name  text not null,            -- exactly what was typed
  ingredient_id bigint references ingredients(id),  -- null = unmatched
  status        text not null default 'in' check (status in ('in','low')),
  added_by      text not null,
  created_at    timestamptz not null default now()
);

create table recipes (
  id           bigserial primary key,
  title        text not null,
  instructions text,
  servings     int,
  source_url   text,
  license      text,
  attribution  text,
  created_by   text,
  created_at   timestamptz not null default now()
);

create table recipe_ingredients (
  recipe_id     bigint not null references recipes(id) on delete cascade,
  raw_text      text not null,            -- '2 cloves garlic, minced'
  ingredient_id bigint references ingredients(id),
  is_optional   boolean not null default false
);

create index on recipe_ingredients (recipe_id);
create index on recipe_ingredients (ingredient_id);
create index on items (ingredient_id);

-- RLS: deny-all to anon. All access is via server route handlers
-- using the service_role key, which bypasses RLS.
alter table ingredients        enable row level security;
alter table ingredient_aliases enable row level security;
alter table items              enable row level security;
alter table recipes            enable row level security;
alter table recipe_ingredients enable row level security;
-- no policies created = no access for anon. This is intentional.
```

---

## 8. The matching query (F4)

Rank recipes by how few of their non-optional, non-staple ingredients are absent from the pantry.

```sql
select r.id, r.title,
       count(*) filter (
         where ri.ingredient_id not in (
           select ingredient_id from items where ingredient_id is not null
         )
       ) as missing_count
from recipes r
join recipe_ingredients ri on ri.recipe_id = r.id
join ingredients i on i.id = ri.ingredient_id
where not ri.is_optional and not i.is_staple
group by r.id, r.title
order by missing_count asc
limit 30;
```

Run this in Postgres. **Do not** fetch all recipes into JavaScript and filter there.

---

## 9. Security model

No accounts, but the database must not be world-writable.

- **RLS enabled, no policies** → the `anon` role can touch nothing.
- **All reads and writes go through Next.js route handlers** using the `service_role` key from a server-only environment variable.
- **In Next.js, any variable prefixed `NEXT_PUBLIC_` is bundled into the browser.** No secret may ever carry that prefix.
- **Gate (F8):** `middleware.ts` checks a signed httpOnly cookie; missing or invalid redirects to `/gate`.

This is the entire reason the app is Next.js rather than a static SPA talking directly to Supabase.

---

## 10. Repository layout

```
pantry/
├── docs/SPEC.md                   # this file
├── app/
│   ├── layout.tsx                 # shell, fonts, PWA meta
│   ├── page.tsx                   # F1 pantry list (main screen)
│   ├── cook/page.tsx              # F4 results
│   ├── recipes/
│   │   ├── page.tsx               # recipe list
│   │   ├── new/page.tsx           # F6 manual entry
│   │   └── [id]/page.tsx          # full recipe view
│   ├── gate/page.tsx              # F8 passcode screen
│   └── api/
│       ├── items/route.ts         # GET list, POST bulk add, PATCH status
│       ├── cook/route.ts          # F4 matching
│       ├── recipes/route.ts       # POST manual create
│       ├── import/route.ts        # F5 URL import
│       ├── gate/route.ts          # passcode check, sets cookie
│       └── health/route.ts        # keep-alive ping target
├── lib/
│   ├── supabase.ts                # server-side client — SERVER ONLY
│   ├── recipes.ts                 # findByPantry(), importFromUrl()
│   ├── normalize.ts               # F3
│   └── session.ts                 # cookie signing / reading
├── data/
│   ├── ingredients.seed.json      # canonical list + categories + staple flags
│   └── aliases.seed.json          # starting alias map
├── scripts/
│   ├── seed-ingredients.ts
│   └── seed-recipes.ts            # F7
├── supabase/schema.sql            # source of truth for the DB
├── components/                    # ItemRow, AddBar, RecipeCard…
├── middleware.ts                  # F8 gate
├── public/manifest.json           # PWA install
├── .env.example                   # key NAMES only — committed
└── .github/workflows/
    ├── keepalive.yml              # daily ping to /api/health
    └── backup.yml                 # weekly JSON dump of recipes + aliases
```

---

## 11. Credentials — where each value is entered

Four distinct places. Confusing them is the most common way a secret leaks.

### Supabase dashboard
- Create the project; choose the Sydney region.
- SQL Editor → paste and run `supabase/schema.sql`.
- Settings → API → copy the Project URL and the `service_role` key (secret; bypasses RLS).
- Store the database password in a password manager. The app doesn't need it.

### `.env.local` (your machine, gitignored, never committed)
```
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
APP_PASSCODE=
SESSION_SECRET=
```
Generate `SESSION_SECRET` with `openssl rand -base64 32`.

### Vercel dashboard
Settings → Environment Variables → add the same four keys for Production. Vercel cannot read your `.env.local`. If a build works locally and 500s in production, a missing variable here is the first thing to check.

### GitHub repo settings
Keep the repo private. Settings → Secrets and variables → Actions → add `HEALTH_URL` for the keep-alive workflow.

### What never goes where

| Never | Because |
|---|---|
| `service_role` key in a client component | Client components ship to the browser |
| Any secret pasted into a chat, including with Claude Code | It ends up in logs and session history |
| `.env.local` committed | Even a deleted commit keeps it in history forever |
| Real values in `.env.example` | It's committed by design |

If a key is ever committed, rotate it in Supabase immediately. Deleting the commit is not sufficient.

---

## 12. Operations

**Supabase pauses free projects after about a week without database activity.** Three people using it daily normally prevents that, but a trip home breaks the streak. Mitigation: a GitHub Actions cron running daily against `/api/health`, which performs one trivial `select`.

**The free plan has zero backup retention.** A second weekly workflow dumps `recipes`, `recipe_ingredients`, and `ingredient_aliases` to JSON and commits them. The pantry contents are replaceable; the recipe collection and alias table are not.

**Git conventions:** `main` is what's deployed. Feature branch per build-order step, PR so there's a diff to read. Tag `v1` after step 3.

---

## 13. First-run checklist

1. `npx create-next-app@latest pantry --typescript --tailwind --app`
2. `git init`, first commit, push to a new **private** GitHub repo
3. Create the Supabase project, run `schema.sql`
4. Create `.env.local`, fill all four values
5. `npm run dev`, confirm it starts and can read from Supabase
6. Run `scripts/seed-ingredients.ts`
7. Build the build order (section 5)
8. Connect Vercel, add env vars, push to `main`, confirm the deployed URL works
9. Add the gate (F8) **before** sharing the URL
10. Add both GitHub Actions workflows

**Installing on phones:** add `public/manifest.json` with name, icons, `"display": "standalone"`, and a theme colour. iOS: Safari → Share → Add to Home Screen. Android Chrome offers the install prompt. No app store, no developer account, no cost.

---

## 14. Open decisions

- **Which CC-licensed dataset to seed from** (F7), and how many recipes to keep after filtering. Decide before step 5 of the build order, not before.
- **Whether `low` should count as missing in F4.** Defaulted to *present* for v1; revisit after real use.
