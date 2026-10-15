# 🎄 Secret Santa

A tiny Secret Santa organizer. The organizer enters the participants (with optional exclusions), the app draws the pairs, and every participant gets a **personal link** that reveals only their own match.

- Pure static site (two HTML files, no build step)
- Supabase (Postgres) stores the draws
- Hosted on GitHub Pages: <https://tgnine1.github.io/secret-santa/>
- The interface is in Polish

## How it works

1. The organizer opens `index.html`, enters the names, exclusions, budget and an optional wish-list link.
2. The browser draws the pairs (reshuffles until all exclusions are satisfied).
3. The pairs are sent to the database function `create_draw`, which stores them and generates a random secret token for every person. The organizer gets one link per person.
4. The organizer sends each person only their own link: `p.html?drawId=…&name=…&token=…`
5. A participant opens the link, taps **Pokaż mój wynik**, and sees who they buy a gift for. The browser calls `get_assignment`, and the server returns that one result only if the token matches.

## Files

| File | Purpose |
|---|---|
| `index.html` | Organizer page: set up the draw, get the links |
| `p.html` | Participant page: reveal your match |
| `supabase-setup.sql` | Locks the tables and creates the two database functions |
| `LICENSE` | MIT License |
| `README.md` | This file |

## Security model

This repo is safe to keep public. It contains no names, no draws and no private keys.

- The Supabase URL and the `anon` key in the HTML files are **public by design**. Anyone can see them in the page source, so hiding them in the repo would not help.
- What protects the data is the database. `supabase-setup.sql` enables Row Level Security, removes direct table access for the public key, and exposes only two functions:
  - `create_draw(...)` creates a draw and generates the tokens on the server.
  - `get_assignment(draw_id, name, token)` returns only the matching person's result.
- Without the right token nobody can read a pairing, and the tables cannot be listed with the public key.
- Never commit a Supabase `service_role` key. It bypasses all of the above.

### Known limitations

- The organizer page has no login, so anyone who knows the URL could create draws. The function limits input (up to 60 people, length limits), but it is not a hard barrier. Add a passcode check inside `create_draw` if this matters to you.
- The links are shown only once, right after the draw. They cannot be recovered later from the page, so copy them (there is a **Kopiuj wszystkie linki** button).
- Whoever has all the links, or access to the Supabase dashboard, can see all pairings, including the organizer.
- Exclusions work in one direction only: `Ania: Tomasz` means Ania will not draw Tomasz, not the other way round.
- The draw itself is computed in the organizer's browser.

## Setup from scratch

### 1. Create a Supabase project

Create a free project at <https://supabase.com>. In the project's API settings, copy the **Project URL** and the **anon public** key.

### 2. Create the tables

In **SQL Editor**, run a schema like this (adjust it if you already have tables):

```sql
create table public.draws (
  id uuid primary key default gen_random_uuid(),
  slug text not null unique,
  budget text,
  wish_link text,
  created_at timestamptz default now()
);

create table public.assignments (
  id uuid primary key default gen_random_uuid(),
  draw_id uuid not null references public.draws(id) on delete cascade,
  name text not null,
  token text not null,
  receiver text not null
);
```

The functions only rely on these columns: `draws(id, slug, budget, wish_link)` and `assignments(draw_id, name, token, receiver)`.

### 3. Lock it down

Open `supabase-setup.sql`, paste it into the SQL Editor and run it. Then check that direct access is blocked. On your live `p.html`, open the browser console (F12) and run:

```js
await _supabase.from('assignments').select('*')
```

You should get a `permission denied` error or an empty list, never a list of people.

### 4. Add your keys

In both `index.html` and `p.html`, set these two constants near the top of the `<script>` block:

```js
const SUPABASE_URL = "https://YOUR-PROJECT.supabase.co";
const SUPABASE_ANON_KEY = "YOUR-ANON-PUBLIC-KEY";
```

### 5. Deploy

**GitHub Pages:** repo **Settings → Pages → Deploy from a branch**, choose `main` and `/ (root)`.

**Cloudflare Pages** (or any static host): connect the repo, no framework, no build command, output directory `/`.

The participant link is built relative to the organizer page's address, so it works on any host. Links already sent keep pointing at the address they were created on.

### Run locally

```bash
python -m http.server 8000
# open http://localhost:8000/
```

## Cleaning up after the event

Names and pairings stay in your Supabase project until you delete them. To remove a draw, run this in the SQL Editor (replace the name):

```sql
delete from public.assignments
where draw_id in (select id from public.draws where slug = 'Superaki 2025');

delete from public.draws where slug = 'Superaki 2025';
```

## Tech notes

- `@supabase/supabase-js` is loaded from a CDN and pinned to an exact version (`2.45.4`).
- The pages are marked `noindex`, and the participant page sends no referrer, so the token in the URL is not leaked when the wish-list link is opened.
- Accessibility basics: labelled form fields, visible keyboard focus, announced status messages, and reduced-motion support.

## License

Released under the [MIT License](LICENSE). You are free to use, copy, modify and share the code, as long as the copyright notice and license text are kept. It comes without any warranty.
