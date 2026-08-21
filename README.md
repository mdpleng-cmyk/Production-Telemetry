/**
 * TELEMETRY — MODULE MAP (Supabase rebuild, Aug 2026)
 * ==========================================================
 * MDPL bottling — multi-line, multi-product web app.
 * Rebuilt from a Google Apps Script + Sheets version (see "ORIGIN" below)
 * onto Supabase (Postgres + Auth + Realtime) with a single-file vanilla-JS
 * frontend. Manual counter entry only — the original's Gemini photo-OCR
 * pipeline was deliberately dropped, not lost; see DROPPED FROM ORIGINAL.
 *
 * FILES
 * -----
 * telemetry_schema.sql   - Full schema: tables, RPC functions, RLS, seed
 *                           data. Source of truth for the database. Run
 *                           once on a fresh project (see TELEMETRY_SETUP.md).
 * telemetry_index.html   - The entire frontend. Login/PIN, operator entry
 *                           view, control room dashboard. No build step —
 *                           open directly or host as a static file.
 * TELEMETRY_SETUP.md     - Step-by-step Supabase project setup, operator
 *                           account creation, deployment.
 * migration_fix_make_interval.sql - One-off fix already applied; safe to
 *                           ignore if working from telemetry_schema.sql
 *                           directly (the fix is already folded in there).
 *
 * PROJECT ISOLATION
 * ------------------
 * This is a SEPARATE Supabase project from the MDPL meter-readings app.
 * Separate URL, separate anon key, separate `operators` table — logins do
 * not carry over between the two. Deliberate choice: different operators
 * use each system, and isolation was preferred over one shared login. See
 * chat history if this decision needs revisiting.
 *
 * DATABASE OBJECTS
 * -----------------------------------------------------------
 * TABLES
 *   operators     - id (= auth.users.id), code, name, role
 *                   (operator|control_room), active
 *   lines         - line_id (text PK, e.g. 'L1'), line_name, active
 *   products      - sku_id (text PK), sku_name, rated_speed_bpm, active
 *   reason_codes  - reason_code_id (text PK), label, default_type
 *                   (Planned|Unplanned), active
 *   shifts        - shift_id, line_id, started_at, ended_at, status
 *                   (OPEN|CLOSED). THE canonical shift definition — every
 *                   calculation reads started_at/ended_at from here, there
 *                   is no second competing definition anywhere.
 *   runs          - run_id, shift_id, line_id, sku_id, started_at, ended_at,
 *                   status. One row per (shift, product) span. A product
 *                   changeover closes the current run and opens a new one
 *                   WITHOUT touching the shift.
 *   events        - event_id, line_id, shift_id, run_id, event_ts,
 *                   production_counter, notes, entry_type
 *                   (READING|COUNTER_RESET), source (MANUAL|SYSTEM),
 *                   recorded_by, logged_at. RAW FACTS ONLY — no stored
 *                   interval/BPM/output columns. See "RECOMPUTE, NEVER
 *                   FREEZE" below.
 *   downtime_log  - downtime_id, line_id, shift_id, run_id, start_ts,
 *                   end_ts, duration_min, type, reason_code_id,
 *                   reason_text, inferred, logged_by, logged_at
 *
 * RPC FUNCTIONS (call via supabase.rpc(name, params); all SECURITY DEFINER)
 *   log_reading(p_line_id, p_counter, p_notes, p_confirmed_checkpoint_ts)
 *       The only way an Events row gets written. Resolves/creates the
 *       current shift and run, computes the interval against the last
 *       event, detects counter resets and low-BPM downtime. Returns a
 *       jsonb result — see RETURN SHAPES below.
 *   log_downtime(p_line_id, p_start_ts, p_end_ts, p_type, p_reason_code_id,
 *                p_reason_text)
 *       Writes a DowntimeLog row. Called either after a downtime_prompt
 *       from log_reading, or manually.
 *   manual_start_new_shift(p_line_id)
 *       Explicit operator override — force-closes the open shift/run and
 *       opens a fresh one immediately, regardless of block boundaries.
 *   set_product(p_line_id, p_sku_id)
 *       Operator-driven changeover. Closes the open run, opens a new one
 *       on the same shift with the new sku_id.
 *   insert_backdated_reading(p_line_id, p_event_ts, p_counter, p_notes)
 *       Control-room only. Resolves the shift/run covering p_event_ts
 *       (not "now"), creating them if the block was never touched live —
 *       using the same 6/14/22 boundaries as the live rolling logic.
 *       Refuses if that would overlap an existing shift. Skips reset
 *       detection and downtime auto-inference — both assume the reading
 *       just happened.
 *   delete_reading(p_event_id)
 *       Control-room only. Hard-deletes a reading. No undo. The event
 *       that followed it recalculates its own interval against the new
 *       prior event automatically on next load.
 *
 * HELPER FUNCTIONS (not called from the frontend)
 *   is_control_room()          - RLS helper, checks operators.role
 *   current_block_start(ts)    - fixed 6/14/22 block boundary calc, in the
 *                                 'Asia/Kolkata' timezone (hardcoded — see
 *                                 GOTCHAS)
 *   _get_or_create_shift(...)  - auto-rolls a line's open shift across
 *                                 block boundaries, closing the old one and
 *                                 force-closing its open run
 *   _get_or_create_run(...)    - opens a run for the current shift, carrying
 *                                 the SKU forward from whatever was last
 *                                 running on the line (first-ever run on a
 *                                 line starts as 'UNSET')
 *
 * RETURN SHAPES from log_reading()
 *   Normal write:
 *     { event_id, shift_id, run_id, interval_output, interval_bpm,
 *       interval_duration_min, warning, downtime_prompt }
 *     downtime_prompt is null unless BPM fell below 90% of the product's
 *     rated_speed_bpm, in which case it's:
 *     { inferred_start_ts, inferred_end_ts, inferred_downtime_min,
 *       interval_bpm, rated_speed, shift_id, run_id }
 *   Needs reset confirmation (first reading of a run, counter dropped vs.
 *   the line's last reading, no confirmed checkpoint supplied yet):
 *     { needs_reset_confirmation: true, suggested_checkpoint_ts, shift_id,
 *       run_id }
 *     Frontend must re-call log_reading with the same p_counter/p_notes
 *     plus p_confirmed_checkpoint_ts set to complete the write.
 *
 * ARCHITECTURE DECISIONS AND WHY
 * -----------------------------------------------------------
 * 1. RECOMPUTE, NEVER FREEZE. `events` stores only event_ts and
 *    production_counter. Interval output, BPM, and duration are computed
 *    fresh every time — server-side in log_reading() for the just-written
 *    row, client-side in computeInterval()/enrichEvents() (in
 *    telemetry_index.html) for everything already on screen. A bug fix in
 *    either implementation corrects all historical numbers on next load,
 *    with zero backfill. IMPORTANT: the two implementations (SQL in
 *    log_reading, JS in computeInterval) must be kept in sync by hand —
 *    there's no shared source. If you change the interval math, change it
 *    in both places.
 *
 * 2. ONE SHIFT DEFINITION. shifts rows are the single source of truth for
 *    both the live dashboard and any historical/OEE-style calculation.
 *    Auto-rolls at fixed 6/14/22 block boundaries via
 *    _get_or_create_shift(); manual_start_new_shift() is an explicit
 *    override for real exceptions (early handover, correcting a mistake),
 *    not a cosmetic view toggle — it closes and reopens a real row.
 *
 * 3. WRITES GO THROUGH SECURITY DEFINER RPCs, NOT DIRECT TABLE INSERTS.
 *    Operators have no INSERT/UPDATE RLS policy on shifts/runs/events/
 *    downtime_log at all — the only way those tables get written is by
 *    calling log_reading/log_downtime/manual_start_new_shift/set_product,
 *    which run with elevated privileges and bypass RLS internally. This
 *    means the shift/run bookkeeping logic can never be accidentally
 *    bypassed by a client-side bug or a modified frontend — there is no
 *    other door in.
 *
 * 4. pg_advisory_xact_lock REPLACES withLineLock_(). The original Apps
 *    Script version used LockService.getScriptLock() to make the
 *    read-check-write sequence (get/create shift → get/create run → check
 *    for reset → write event) atomic per line. Postgres has no direct
 *    equivalent to a script-level mutex, so each RPC function opens with
 *    `perform pg_advisory_xact_lock(hashtext(p_line_id))` — this blocks
 *    any other transaction trying to acquire the same lock key until the
 *    current transaction commits or rolls back, and releases
 *    automatically at transaction end. Two operators submitting readings
 *    on the same line within milliseconds of each other cannot race.
 *
 * 5. RATED SPEED LIVES ON products, keyed off the run's sku_id — never a
 *    global constant. Multi-line, multi-product correctness depends on
 *    this exactly as it did in the original.
 * 6. BACKDATED READINGS use a separate RPC (insert_backdated_reading),
 *    not log_reading with an optional timestamp param. Reason: log_reading's
 *    shift/run resolution, reset detection, and downtime inference all
 *    implicitly assume "this reading just happened" (resolve against
 *    now(), diff against the last-written row). Backdating needs shift/run
 *    *lookup* against an arbitrary past timestamp instead, and must skip
 *    reset/downtime auto-detection entirely — neither makes sense for a
 *    reading inserted after the fact. Control-room role only.
 *    If no shift/run covers the given timestamp, one is created using the
 *    same 6/14/22 block boundaries as the live rolling logic
 *    (current_block_start()) — this handles a block that nobody logged a
 *    reading in at the time. Refuses (raises) rather than creating one if
 *    doing so would overlap an existing shift, so it can't be used to
 *    reshape real shift history. Inserted rows still flow through the
 *    RECOMPUTE, NEVER FREEZE model — enrichEvents() slots them in by
 *    event_ts and recalculates neighboring intervals automatically.
 *
 * 7. DELETING A READING (delete_reading RPC) is a hard delete, control-room
 *    only. No soft-delete/undo — the recompute-on-read model means the
 *    event that followed the deleted one will automatically pick up the
 *    correct prior event and recalculate its interval on next load, so no
 *    separate fixup step is needed after a delete.
 *
 * DROPPED FROM ORIGINAL (deliberate, not oversights)
 * -----------------------------------------------------------
 * - Photo upload + Gemini OCR (VisionService.gs equivalent). Manual
 *   counter entry only. Consequence: there's no OCR'd machine_status
 *   field, so log_reading can no longer tell "still stopped" from
 *   "recovered mid-interval" — every slow interval surfaces as ONE
 *   complete downtime_prompt immediately, rather than potentially
 *   silently extending across several readings while a machine sits
 *   stopped. If that turns out to matter in practice, the fix is adding a
 *   manual Running/Stopped toggle to the entry form and restoring the
 *   "silently open/extend downtime" branch in log_reading — ask whoever
 *   is modifying this to look at the original checkPossibleDowntime_ /
 *   writeReadingAndEvent_ logic in the GAS version for the exact shape.
 * - Photo-timestamp reconciliation (TIME_DISCREPANCY_TOLERANCE_MIN,
 *   confirmReadingWithTime). No longer needed — manual entries always use
 *   the server's now() as event_ts, there's no separate "time the photo
 *   claims" to reconcile against.
 *
 * DELIBERATELY NOT BUILT YET (fast-follow candidates, not forgotten)
 * -----------------------------------------------------------
 * - Rolling 24h SVG BPM trend graph (buildTrendGraphSvg_ in the original).
 *   NOTE: the today-overview TIMELINE BAR (buildTimelineBar_ equivalent —
 *   colored segments showing when CIP/changeover/downtime happened across
 *   the day) IS built as of the Aug 2026 entry-view rework below. Only the
 *   line-chart-style BPM trend graph is still missing.
 * - Hourly-breakdown grid on the History view (interpolateCounterAt_).
 * - Quality/scrap tracking — same extension point as the original design:
 *   add a `quality_log` table keyed by run_id (id, timestamp, category,
 *   qty_rejected) when ready. Quality-adjusted output for a run becomes
 *   (events output for that run) minus (sum of quality_log rejects for
 *   that run_id). Nothing in the current schema needs to change to
 *   support this later.
 *
 * KNOWN GOTCHAS
 * -----------------------------------------------------------
 * - `current_block_start()` hardcodes the 'Asia/Kolkata' timezone in two
 *   `at time zone` casts. If the plant's timezone ever changes, both need
 *   updating together — search the SQL file for 'Asia/Kolkata'.
 * - Postgres numeric vs. integer function signatures bit us once already:
 *   make_interval()'s named params are integer-typed and silently reject
 *   numeric arguments (see migration_fix_make_interval.sql). Watch for
 *   this pattern anywhere else fractional minutes/seconds get passed into
 *   a built-in date/time function.
 * - SHIFT_BLOCK_HOURS is duplicated: [6,14,22] as a Postgres calculation
 *   inside current_block_start(), and again as a plain JS array
 *   (SHIFT_BLOCK_HOURS) in telemetry_index.html for the client-side
 *   blockStart() helper. Keep both in sync if this ever changes.
 *
 * ENTRY-VIEW REWORK (Aug 2026)
 * -----------------------------------------------------------
 * Reordered for operator workflow — reading entry is the first thing
 * visible on load, no scrolling required:
 *   1. Compact line/product bar (line <select> auto-hides via
 *      loadLines() when state.lines.length <= 1 — currently true, only
 *      line L1 exists — and shows a plain text label instead; reappears
 *      automatically the moment a second active line is added, no code
 *      change needed)
 *   2. Counter input + Add Reading (was previously below the fold)
 *   3. Today's Total (all shifts) — refreshTodayTotal(), ports
 *      buildDayTotalHtml_. Pulls events from (today's midnight - 24h) for
 *      interval-calc context, sums interval_output for events >= today's
 *      midnight only.
 *   4. Today's Overview — refreshTodayOverview() / buildTimelineBar() in
 *      telemetry_index.html, ports buildTimelineBar_. Colored segments
 *      (CIP blue, changeover purple, label-change pink, planned amber,
 *      unplanned red) across the full 00:00–24:00 day for the current
 *      line, plus a grey "future" segment for time not yet elapsed and a
 *      dimmed "stale" segment if the last reading is >10 min old.
 *   5. This Shift (was "summary-panel", unchanged content, now demoted
 *      below Today's Total/Overview since day-level context ranks above
 *      shift-level for the operator's actual question — "how's today
 *      going")
 *   6. Manually Start New Shift / Shift Log (unchanged, still at bottom)
 * If more lines get added later, revisit whether Today's Total/Overview
 * should also gain a line-scoping affordance beyond the existing
 * line-select — right now both silently follow state.currentLine, same
 * as the shift summary always did.
 *
 * SETUP / MIGRATING DATA
 * -----------------------------------------------------------
 * See TELEMETRY_SETUP.md for project creation, operator account setup,
 * and deployment. There is no data to migrate from the GAS version — the
 * old ShiftLog rows have no line_id/sku_id/shift_id/run_id to attach to,
 * same limitation the original module map already noted for its own
 * predecessor.
 */