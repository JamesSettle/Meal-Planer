# Firestore sync for Meal Planner

Convert localStorage-only app into a shared, real-time household plan synced via Firebase Firestore. Stay single-file, stay on GitHub Pages.

## Design

**One Firestore doc per household.** Path: `households/{householdId}`. Fields mirror current state:
- `week` (array, 7 entries of `{name, protein, done}`)
- `recent` (map of `mealName → timestampMillis`)
- `weeknum` (number)
- `shopChecked` (map of `ingredientNameLower → true`)
- `updatedAt` (server timestamp, for debugging)

One doc = 1 read/write per change, atomic updates (e.g. "New week" rewrites several fields in one go), well under the 1 MB doc limit.

**Identity:** hardcoded `householdId` constant in the file. Plus Firebase Anonymous Auth so security rules can require `request.auth != null`. That blocks anonymous bots without adding any friction for you/your wife — the SDK signs in silently on load.

**Sync model:**
- One `onSnapshot` listener on the household doc → re-renders all four tabs whenever state changes (yours or hers).
- Writes use `updateDoc` with field paths (e.g. `shopChecked.tomatoes: true`) so we send tiny diffs, not the whole state blob.
- Optimistic UI: tick a checkbox, paint it immediately, write in background. Listener echoes it back; idempotent.
- Last-write-wins on conflicts (fine — neither of you will tick the same item in the same millisecond).

**Offline:** enable Firestore offline persistence. Shopping in a supermarket with patchy signal: ticks queue locally and sync when reconnected.

**Migration from localStorage:** on first load, if the Firestore doc doesn't exist, seed it from current localStorage state (so you don't lose this week's plan). After that, localStorage stops being authoritative.

## Tasks

### Setup (you do this in Firebase console)
- [ ] Create a new Firebase project ("meal-planner" or similar)
- [ ] Enable **Firestore** (production mode, region `eur3` or `europe-west2`)
- [ ] Enable **Authentication → Anonymous** sign-in
- [ ] Copy the web app config object (apiKey, authDomain, projectId, etc.) and paste it into the chat
- [ ] Paste in these security rules (I'll give you the exact text):
  ```
  match /households/{id} {
    allow read, write: if request.auth != null;
  }
  ```

### Code (I do this)
- [ ] Add Firebase modular SDK imports (CDN, no build step) to `<head>`
- [ ] Add a `FIREBASE_CONFIG` and `HOUSEHOLD_ID` constant block at top of script
- [ ] Replace `S = { … load(...) }` with an in-memory `S` populated by the snapshot listener
- [ ] Wrap the existing `renderAll()` so it only fires after first snapshot arrives (show a tiny "Connecting…" state for the first ~500ms)
- [ ] Rewrite `save(key, value)` to call `updateDoc` with the matching field path
- [ ] Rewrite `markDone`, `toggleShop`, `generateWeek`, `doSwap`, `resetShop` to use granular `updateDoc` calls (not whole-doc rewrites)
- [ ] Add anonymous sign-in on load, then attach the snapshot listener
- [ ] Enable `enableIndexedDbPersistence` for offline support
- [ ] Add a one-shot migration: if doc missing on first connect, seed from current localStorage values
- [ ] Keep the existing iOS storage warning, but rewrite it to mean "can't reach server AND can't write locally"

### Verify
- [ ] Open on laptop + phone, generate a new week on one, see it on the other within ~1s
- [ ] Tick a shopping item on the phone, see it strike through on the laptop
- [ ] Turn on airplane mode on the phone, tick more items, turn it back on, watch them sync
- [ ] Check Firebase console "Usage" tab after a day to confirm we're nowhere near free-tier limits

### Out of scope (don't do unless asked)
- Per-user identity / "ticked by Jess" labels
- Multiple households
- Push notifications
- Cloud Functions
- Build step / bundler

## Open questions for you

1. **Region:** OK with EU (`eur3`)? Lower latency than US for you both, and keeps data in-region.
2. **Household ID:** I'll generate a random opaque string (e.g. `hh-7k4p9q2r`). Or do you want a memorable one?
3. **Existing data:** want me to seed the household with whatever's currently in your laptop's localStorage, or start blank and just hit "New week"?
