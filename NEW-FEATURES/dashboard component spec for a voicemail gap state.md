Here's the spec — a state machine for the "post-call gap" component, with the data each state pulls from Supabase, and the psychology from before mapped to concrete UI behavior.

## States

```
IDLE → DIALING → ON_CALL → {CONNECTED_WRAP | VOICEMAIL_GAP | NO_ANSWER_GAP} → PREPPING_NEXT → DIALING
                                                                      ↘ PAUSED (manual, from any gap state)
```

| State | Entered when | Typical duration |
|---|---|---|
| `ON_CALL` | `dialer_sessions.status = 'in_call'` | variable |
| `VOICEMAIL_GAP` | salesperson taps "Voicemail" mid-call | ~10–20s (script playback) |
| `NO_ANSWER_GAP` | call ends with outcome `no_answer`/`busy`/`rejected`, duration < ~5s | ~2–4s |
| `CONNECTED_WRAP` | call ends with outcome `connected`, duration > threshold | 15–60s, salesperson-paced |
| `PREPPING_NEXT` | next contact loaded, about to auto-dial | 2–5s |
| `PAUSED` | manual pause command, from any state | indefinite |

The three "gap" states share one component shell but render different content — this is the key design decision: **don't build one generic "waiting" screen**, build a shell that reads call outcome and adjusts weight/tone accordingly, per the "don't treat a reject like a pitch" point.

## Component: `PostCallGap`

**Props / data it subscribes to (via `dialer_sessions` realtime row):**
```ts
{
  gap_type: 'voicemail' | 'no_answer' | 'connected_wrap',
  previous_contact: { id, name, company, last_note },
  next_contact: { id, name, company, last_note, tags } | null,
  call_outcome: string,
  call_duration_sec: number,
  transcript_status: 'pending' | 'ready' | null,
  gap_started_at: timestamptz
}
```

**Per-state rendering:**

**`NO_ANSWER_GAP`** — minimal by design
- No prompts, no stretch reminder. Just: soft color shift, a thin filling ring (not a numeric countdown), next contact's name fading in quietly.
- Rationale: don't make the app chatty after a non-event; respect momentum.

**`VOICEMAIL_GAP`**
- Ring duration bound to actual script playback time (query it, don't guess — store `voicemail_script_duration_sec` alongside the script asset so the ring is accurate, not decorative).
- Next-contact prep card fades in immediately: name, company, one-line "why they're a fit" pulled from `contacts.notes` or last `call_logs.ai_summary`.
- Optional TTS one-liner, off by default, toggle in settings — respects that some people want eyes-free prep, others find audio noisy.
- Soft chime ~500ms before `dialer_sessions.status` flips back to `dialing`, sourced from the realtime event itself, not a client-side timer guess.

**`CONNECTED_WRAP`** — the only state with real work to review
- Pulls `call_logs.transcript` + `call_logs.ai_summary` (populated async by the transcription pipeline — show a skeleton/shimmer if `transcript_status = 'pending'`, since a real call's transcript won't be ready in 2 seconds).
- One-tap confirm on the AI-drafted note rather than a text box: `Looks good ✓` / `Edit`. Edit expands to full text area only on demand.
- This is the one state where "Resume" isn't automatic — it's a deliberate button, since the salesperson may still be typing/thinking. Don't auto-advance out of this one.
- Daily counter (calls / connects / streak) shown small, bottom corner, non-modal — visible, not demanding attention.

**`PAUSED`** (voice-note flow from earlier)
- Same shell, but the ring/timer disappears entirely — paused means paused, no implied countdown.
- Mic capture UI (record button, live waveform) + quick-action dropdown (follow-up in N days / mark status / flag).
- Explicit "Resume dialing" button — never auto-resumes from a manual pause.

## Transitions

| From | Trigger | To |
|---|---|---|
| `ON_CALL` | `call_logs` row closes with outcome | `VOICEMAIL_GAP` / `NO_ANSWER_GAP` / `CONNECTED_WRAP` (routed by outcome + duration) |
| any gap state | salesperson taps Pause | `PAUSED` |
| `PAUSED` | salesperson taps Resume | `PREPPING_NEXT` |
| `NO_ANSWER_GAP` | ring completes | `PREPPING_NEXT` → auto |
| `VOICEMAIL_GAP` | phone reports line free (realtime) | `PREPPING_NEXT` → auto |
| `CONNECTED_WRAP` | salesperson confirms note | `PREPPING_NEXT` → auto |
| `PREPPING_NEXT` | phone confirms dial started | `DIALING` |

All auto-transitions are driven by the realtime row changing underneath the component, not by client-side `setTimeout` — the phone is the source of truth for when the line actually frees; the PC just reflects it. This matters because if you time it client-side and the phone's actual script playback drifts even a second, the countdown lies and undercuts the exact calm you're trying to create.

## Supabase additions needed

```sql
alter table call_logs add column ai_summary text;
alter table call_logs add column transcript_status text default 'pending';
alter table call_logs add column outcome text; -- 'connected' | 'voicemail' | 'no_answer' | 'busy' | 'rejected'

alter table dialer_sessions add column gap_type text;
alter table dialer_sessions add column gap_started_at timestamptz;

create table voicemail_scripts (
  id uuid primary key default gen_random_uuid(),
  salesperson_id uuid references auth.users,
  audio_url text,
  duration_sec numeric,
  is_default boolean default false
);
```

Want me to write the actual `PostCallGap` React component next (states + realtime subscription wired up), or the Android-side logic that flips `gap_type`/`outcome` on the session row when a call ends?