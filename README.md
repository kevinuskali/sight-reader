# Sight Reader

A single-file HTML piano sight-reading trainer with optional cloud-synced history via Supabase.

## Features

- Pitch detection via microphone (Web Audio API autocorrelation)
- Treble and bass clef training, with sharps
- Practice history with daily summaries and session log
- Works without signing in — history is stored in localStorage
- Google sign-in for cloud-synced history across devices
- Local history is automatically merged into your account on first sign-in
- Each user can only see their own data (Row Level Security)

## Running locally

Serve the file with any static HTTP server:

```
python3 -m http.server 8080
```

Then open `http://localhost:8080`. No build step, no dependencies to install.

---

## Setting up Supabase (optional — for cloud history sync)

Supabase is not required. If `SUPABASE_URL` and `SUPABASE_ANON_KEY` are left as their placeholder values in `index.html`, the app runs entirely offline with localStorage.

### 1. Create a Supabase project

1. Go to [supabase.com](https://supabase.com) and sign up for a free account.
2. Click **New project** and fill in the details.
3. After the project is ready, go to **Settings → API**.
4. Copy the **Project URL** and the **anon public** key — you will need them in step 5.

### 2. Create the sessions table

In your Supabase project, open the **SQL Editor** and run the following:

```sql
create table sessions (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid references auth.users(id) on delete cascade not null,
  ts         bigint not null,
  date       text,
  day        text,
  clef       text,
  notes      int,
  sharps     boolean,
  correct    int,
  total      int,
  pct        int,
  mins       float,
  created_at timestamptz default now()
);

-- Enable Row Level Security so users can only access their own rows.
alter table sessions enable row level security;

create policy "own sessions" on sessions
  for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

### 3. Set up Google OAuth

#### In Google Cloud Console

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create or select a project.
2. Navigate to **APIs & Services → Credentials**.
3. Click **Create Credentials → OAuth 2.0 Client ID**.
4. Set **Application type** to **Web application**.
5. Under **Authorized JavaScript origins**, add the URL where you host the app (e.g. `http://localhost:8080` for local development, or your production domain).
6. Under **Authorized redirect URIs**, add your Supabase OAuth callback URL:
   ```
   https://<your-project-ref>.supabase.co/auth/v1/callback
   ```
   Replace `<your-project-ref>` with the subdomain from your Supabase project URL.
7. Click **Create** and copy the **Client ID** and **Client Secret**.

#### In Supabase

1. In your Supabase project, go to **Authentication → Providers → Google**.
2. Toggle the provider **on**.
3. Paste the **Client ID** and **Client Secret** from Google Cloud Console.
4. Save.

#### Configure allowed redirect URLs

1. Still in **Authentication**, go to **URL Configuration**.
2. Under **Redirect URLs**, add the URL where the app is served (e.g. `http://localhost:8080`). Supabase will only redirect back to approved URLs after OAuth.
3. Save.

### 4. Add your credentials to index.html

Near the top of the `<script>` block in `index.html`, replace the two placeholder values:

```js
const SUPABASE_URL      = 'https://your-project-ref.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-key-here';
```

The anon key is safe to embed in client-side code — Row Level Security ensures each user can only read and write their own rows.

That's it. Reload the page and a **Sign in with Google** button will appear below the title. Users who haven't signed in continue to use the app normally with localStorage; their data is merged into Supabase automatically the first time they sign in.

---

## Architecture notes

- Pure HTML/CSS/JS, no build step, no npm
- SVG constants: `LG=24` (line gap), `NR=11` (note radius); mobile uses narrower viewBox
- Note tables: `TREBLE_NATURALS` and `BASS_NATURALS`, `sp` = steps from bottom staff line
- `redraw()` is the main draw function, calls `drawStaffLines` / `drawTrebleClef` / `drawBassClef` / `drawNote`
- `transitioning` flag prevents double-skipping after a correct note
- `historyCache` holds the last-fetched sessions so `renderDailyView` and `renderSessionLog` share one async fetch per render cycle
- When Supabase is not configured (`db === null`), all auth UI is hidden and the code path is identical to the original localStorage-only version
