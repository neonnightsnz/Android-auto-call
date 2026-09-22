That's a great feature, and it maps well onto the stack you've already got (Supabase + Next.js admin + Kotlin app). The key insight is that your phone and PC don't need to talk directly — they both just talk to Supabase, and Supabase Realtime pushes updates to whichever side needs them. Here's how I'd structure it.

**1. Add a "live session" table**

This is the shared state both sides read/write:

```sql
create table dialer_sessions (
  id uuid primary key default gen_random_uuid(),
  salesperson_id uuid references auth.users not null,
  status text not null default 'stopped', -- 'running' | 'paused' | 'stopped'
  current_contact_id uuid references contacts,
  current_call_started_at timestamptz,
  updated_at timestamptz default now()
);
```

One row per salesperson/device. The Android app updates `current_contact_id` and `status` as it dials; the dashboard just subscribes to this row.

**2. Add a lightweight "commands" table**

This is how the PC tells the phone what to do — start, pause, stop, or "call this specific contact now" (your click-to-call):

```sql
create table dialer_commands (
  id uuid primary key default gen_random_uuid(),
  salesperson_id uuid references auth.users not null,
  command text not null, -- 'start' | 'pause' | 'stop' | 'call_now'
  target_contact_id uuid references contacts,
  created_at timestamptz default now(),
  consumed boolean default false
);
```

**3. Wire up Supabase Realtime both ways**

- **Android app:** subscribe to `dialer_commands` (filtered by its own `salesperson_id`). When a new row appears, act on it immediately — dial `target_contact_id`, or pause/resume the sequential loop. Mark it `consumed`. It also subscribes to nothing else; it just *writes* to `dialer_sessions` after every call-state change (already has this logic via your `PhoneStateListener`).
- **Admin dashboard:** subscribes to `dialer_sessions` for that salesperson via Supabase's `postgres_changes` listener. The moment the phone updates the row, the dashboard re-renders with the contact's name, company, notes, history, whatever you want on screen — live, while the phone is at your ear.

This gives you near-instant (sub-second) sync without polling, and Supabase Realtime is already part of your stack, so no new infrastructure.

**4. Click-to-call from the dashboard**

A button next to any contact that just does:

```ts
await supabase.from('dialer_commands').insert({
  salesperson_id: user.id,
  command: 'call_now',
  target_contact_id: contact.id
});
```

The phone picks it up over the realtime subscription and calls immediately via `TelecomManager.placeCall()` / `ACTION_CALL`, same as your existing auto-dial logic, just triggered manually instead of by "previous call ended."

**5. Start/stop from the dashboard**

Same pattern — a Start/Pause/Stop button inserts a `start`/`pause`/`stop` command row. Your existing foreground service just needs a small state machine: `RUNNING` → keeps auto-advancing after each call ends; `PAUSED` → finishes current call but doesn't dial the next; `STOPPED` → same, plus tears down the service.

**A few practical notes:**

- **Latency:** realtime updates typically land in a few hundred ms, which is plenty fast for "see who's being called."
- **RLS:** lock both tables down so a salesperson can only read/write their own `salesperson_id` rows, and admins can read everyone's — this matters since you already have multi-tenant RLS set up.
- **Reconnects:** handle the case where the phone loses network mid-call (realtime channel drops) — buffer status updates and flush on reconnect, or just always write current state to `dialer_sessions` on every state change so the next successful write self-heals things.
- **Presence (optional nice-to-have):** Supabase Realtime also has a "presence" feature so the dashboard can show whether the phone/app is even online, not just what it's doing.

Want me to write the actual code for this — the Android side (Realtime subscription + service state machine) and the Next.js dashboard component (live contact card + start/stop/click-to-call buttons)? I can build both as a starting point you can drop into your fork.