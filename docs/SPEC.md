# Pet Afterlife — build spec

Goal: a virtual memorial world where pet owners give a pet who has died a place of its own. People visit, remember them, and connect with others who feel the same loss. Reference: the Korean site "Starkkuk" (a 3D pastel meadow of rounded headstones in rows, halls like "Hall 2 · Sunshine Hill", counters for Hearts / Shining stars / Places left, a "Starlight Portal", a plaza button and an SNS feed). It went viral after 6,000 pets "moved in" within 3 days. Build the same idea but clearly better: warmer, more alive, more personal, and more social.

## Deliverable
ONE self-contained file: /home/claude/pet-afterlife/index.html (begin with <!DOCTYPE html>, <title>Pet Afterlife</title>).
It will be published as a claude.ai artifact (sandboxed iframe). Do NOT publish it yourself.

## Hard platform constraints (CSP / sandbox)
- Scripts: inline, or from cdnjs.cloudflare.com (preferred), cdn.jsdelivr.net/npm/, unpkg.com. Use Three.js from cdnjs, e.g. https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js (no OrbitControls on cdnjs for r128: write your own touch/mouse orbit + pan + pinch-zoom).
- Fonts: Google Fonts stylesheet only. Images: data: and blob: URIs only — no remote images. No fetch to other hosts.
- alert/confirm/prompt, window.open, downloads and window.print are blocked: build in-page dialogs. Forms: preventDefault.
- localStorage may throw: wrap in try/catch, use only for small conveniences (language, sound on/off).
- Routing: only plain #name hashes (letters, digits, hyphens), e.g. #m-abc123 to open a memorial directly. Handle on load and on hashchange.
- If you use viewport-fit=cover, pad with env(safe-area-inset-*).
- The owner mostly opens it on an iPhone: mobile-first, touch-first, 60fps target, keep draw calls low (InstancedMesh or merged geometry for plots, a few hundred plots per hall max, cap devicePixelRatio at 2).

## Shared data: the `db` capability (realtime document store, shared by all viewers)
Declared later as capabilities {db:{}, user:{scopes:["profile"]}}. Access it like this:
```js
// window.claude has ONLY `use`. Resolves later, maybe null (preview / signed out). Never read window.claude.db.
const db = await window.claude?.use?.("db");   // null => run in local demo mode
const user = await window.claude?.use?.("user"); // may be null
const uid = user ? await user.id() : null;       // opaque id; store ids, never names
const ps = await user.profiles([id1,id2]);       // ps[id].name || 'Someone' — resolve at render time
user.isOwner(); user.canEdit(); await user.can("data.write") // true/false/null
```
Guard: `typeof window.claude === 'undefined'` → demo mode immediately. Also treat no answer within 10 s as null.
API (Firestore-like): db.doc("a/b") (EVEN segments = doc), db.collection("a") (ODD = collection), docRef.collection("sub"), coll.doc(id?) / coll.add(data), ref.get() → {exists, id, data()}, ref.set(obj) (full replace), ref.update(obj) (merge, must exist), ref.delete(), query.where(f,op,v).orderBy(f,'desc').limit(n), .get() → {docs,size,empty}, .onSnapshot(next, err) → unsubscribe (deliver docChanges(); first snapshot may be fromCache). ref.acquire({holder, ttlMs}) → {acquired} for a short cooperative lease (use it to claim a plot so two people never take the same spot: acquire plots/<hall>-<n>, get(), set claim only if empty).
Rules: segments use letters, digits, _ - . ~ : @ + only. Docs ≤ 256 KiB plain JSON objects. ≤ 25,000 docs total. One write at a time per doc, only on real user actions. Subscribe ONCE per query (never in render). Errors have .code: invalid_argument, resource_exhausted, quota_exceeded (tell the user), unavailable (retry once), revoked (go read-only). Start a 10 s timer on subscribe; if no data, show a gentle "can't reach the meadow right now" note; never reload.
Write permission: viewers without write access get rejected writes (invalid_argument) — then make the UI read-only for that visit ("You're visiting as a guest").
NO counters via read-modify-write. Count interactions as one doc per person: e.g. memorials/<id>/candles/<uid> {at}, memorials/<id>/hearts/<uid> {at}; counts = snapshot size. Guestbook messages: memorials/<id>/messages/<auto> {authorId, text, at}. Keep aggregate counts for the top bar computed from what you've loaded (fine to approximate from loaded data).

Suggested model:
- memorials/<id>: {name, species ('dog'|'cat'|'rabbit'|'bird'|'hamster'|'fish'|'other'), breed, born, passed (YYYY-MM-DD), story (≤2000 chars), favorites, nicknames, hall, plot (int), ownerId, createdAt (ms), color (fur colour for the little figure), thumb (data URL ~240px JPEG, ≤40 KB)}
- memorials/<id>/photos/<n>: {data: data-URL JPEG, max 1024px long side, quality tuned so each ≤ 180 KB}, up to 6 photos. Compress client-side with canvas (handle HEIC failure gracefully: tell them to pick a JPEG/PNG).
- memorials/<id>/messages, /candles, /hearts, /flowers/<uid> {kind:'rose'|'daisy'|'sunflower'|'treat'|'toy', at}
- letters/<auto>: {memorialId, authorId, text, at, public:true} — "Letters to the stars" feed in the Plaza.
- plots/<hall>-<n>: {memorialId, ownerId}
Do NOT hardcode seed data into the db. If db is null (preview), run a clearly labelled local DEMO with ~12 sample pets kept in memory only. If db works but is empty, show the beautiful empty meadow and a warm invitation to bring the first pet home.

Owner/moderation: the memorial's ownerId (or user.isOwner()) can edit/delete their memorial and delete messages on it. Everyone can report nothing — keep it simple.

## Experience (make it clearly better than Starkkuk)
1. World: a soft, painterly 3D meadow (pastel, warm, hopeful — not a cemetery). Each memorial is a small rounded stone/house with the pet's photo framed on it, a tiny glowing lantern, and visible offerings (flowers, candles flickering, treats) left by visitors. Empty plots are faint outlines with a "+" when you look closely. Gentle grass sway, fireflies, drifting petals, clouds.
2. Sky follows the viewer's local time: dawn, day, golden hour, night. At night each candle lit becomes a star in the sky ("Shining stars"), and pets' lanterns glow. A rainbow bridge on the horizon.
3. Halls (each its own meadow, switch via a bottom sheet): Sunshine Hill, Moonlit Garden, Rainbow Bridge Shore, Starfall Lake, Cherry Blossom Path. Each shows Places left.
4. Top bar like Starkkuk: brand, "Here now" (approx: viewers active recently, or just memorial count if unavailable), Hearts, Shining stars, Places left; bell for anniversaries.
5. Tap a memorial → camera glides to it, then a bottom-sheet profile: photo carousel, name, dates, age, species/breed, nicknames, favourites, story, and actions: ❤️ heart, 🕯 light a candle, 🌼 leave a flower/treat, ✉️ write in guestbook. Guestbook shows messages with names (resolve via user.profiles at render).
6. "Bring them home" flow (the + button / empty plot): multi-step, gentle wording, photo upload (from file input, compressed), choose hall + plot, fur colour, then a short arrival animation (a little glowing figure walks/floats to the plot, lantern lights up). Shareable link #m-<id> with a Copy link button (navigator.clipboard in the click handler; catch refusal and show the text to copy).
7. Plaza (광장 / Plaza button): a calm community feed — newest arrivals, recent "Letters to the stars", and "Remembered today" (birthdays & passing anniversaries). Tap an item → fly to that memorial.
8. Search by pet name; "Random visit" button that floats you to someone's pet who has had few visitors (so every pet gets remembered).
9. Bilingual UI: English and 简体中文 toggle (default from navigator.language). All UI strings in one dictionary.
10. Soft optional ambient sound generated with WebAudio (off by default, start only from a click).
11. Accessible: real buttons, labels, focus rings, reduced-motion support, readable contrast on every sky. A list view fallback if WebGL fails.
12. Tone: compassionate. Never say "dead"; use "crossed the rainbow bridge", "remembered". A small footer line: "Grieving a pet is real grief. If it feels heavy, talking to someone helps." — no hotline numbers.
Do not depict any known copyrighted characters or logos; do not use the name Starkkuk anywhere in the page.

## Quality bar
Distinctive, polished visual design (pick a cohesive palette and type pairing, e.g. a rounded display face + clean sans; CJK fallback). Smooth camera, no jank. Works in a 390×844 portrait viewport and on desktop.

## Verification you must do before finishing
- Extract the inline script(s) and run `node --check` on them.
- If a headless browser is available (try `python3 -c "import playwright"` or `npx playwright`), load the file with a 390×844 viewport, wait 3 s, screenshot to /home/claude/pet-afterlife/shot-mobile.png and also 1280×800 to shot-desktop.png, check console for errors, and fix them. (Three.js from cdnjs may be unreachable from the sandbox network: if so, temporarily serve a locally pip/npm-installed three.min.js just for the test, but keep the cdnjs URL in the final file.)
- Report: file size, features done, anything not done, and any known issues.
