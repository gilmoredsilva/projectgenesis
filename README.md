# Project Genesis — Web (Stage 7)

This repo holds the browser dashboards and the real-time game server for **Project Genesis**, a physical puzzle game for 4–5 players.

| Team | Operator stations | Executors |
| --- | --- | --- |
| Five players (3 operators) | 1 Puzzle Solver · 2 Clue Analyst · 3 Mission Communicator | 2 |
| Four players (2 operators) | 4 Puzzle Solver · 5 Communicator | 2 |
---

## 1. Startup

You need **Node.js 22.12 or newer**.

```powershell
cd path\to\project-genesis-web
npm install          # only needed once; no new dependencies were added in Stage 3
npm run build
npm start
```

The server prints the address to use from other devices, plus the cinematic's duration, resolution, bit rate, codec and **version** (a content hash).

| Who | URL |
| --- | --- |
| Supervisor (phone or laptop) | `http://<laptop-ip>:3001/supervisor` |
| Operator station *n* | `http://<laptop-ip>:3001/operator?station=n` (n = 1 … 5) |
| Console list | `http://<laptop-ip>:3001/` |
| Connection check | `http://<laptop-ip>:3001/diag` |

**Development mode:** `npm run dev`, then use port **5173** instead. Use production mode (port 3001) for game sessions and phones.

**Rehearsals:** for a short mission clock, run `$env:GENESIS_MISSION_DURATION_MS="60000"; npm start` in PowerShell.

**Other settings (Stage 7):** `GENESIS_FINAL_WORD` changes the final cipher's word (A–Z, default `RESCUED`). `GENESIS_DATA_DIR` moves the session records folder (default `data` in this folder — for example, set it outside OneDrive).

---

## 2. Stations and roles

| URL | Team | Role | What it shows |
| --- | --- | --- | --- |
| `/operator?station=1` | 3 operators | **Puzzle Solver** | Five minigame cards, clue-packet status, executor feed; the final-puzzle screen in FINAL |
| `/operator?station=2` | 3 operators | **Clue Analyst** | TERM CLASSIFICATION board, packet log, unlocked clues; moderate feed |
| `/operator?station=3` | 3 operators | **Mission Communicator** | Large executor feed; clues to relay, clue status board and activity beside it |
| `/operator?station=4` | 2 operators | **Puzzle Solver** | Same as station 1 |
| `/operator?station=5` | 2 operators | **Communicator** | One page, two panels: **A · Clue Analysis** (classification) and **B · Communication** (feed, clues to relay, status) |

- **The supervisor's team selector decides which stations are active:**
  - 3 operators → stations 1, 2, 3
  - 2 operators → stations 4, 5
- **Inactive stations** show a standby screen. They aren't required for the start, and the server rejects their workflow actions. They can still download the cinematic in advance.
- **Every active dashboard** shows its station and role, the shared countdown, the mission phase, the connection status, the executor-feed placeholder and the decorative telemetry.
- **Station identity comes from the connection, not from the request.** A station can't act as another station, and if the same station is opened twice, the newer one takes over and the older one shows "Displaced".
- **There is no login.** Anyone on the Wi-Fi can open any URL. That's acceptable for a local game network, but it isn't real security.

---

## 3. Local cutscene cache

### Why

On another laptop, playing the 101 MB MP4 over Wi-Fi buffered even after it seemed loaded. Browsers only keep part of a streamed video in memory, and the bit rate is high for Wi-Fi: about 101 MB over about 2 minutes, roughly 6–7 Mbit/s. Each station now plays a **complete local copy**.

### How it works

1. **The server reads your original `CUTSCENE.mp4` and never modifies it.** It computes the file's SHA-256 and a SHA-256 for each 4 MB chunk. The first 16 hex characters of the file hash are the **version**.
2. **Each station shows "Download / Prepare Cutscene"** until it has a valid copy. Downloading fetches the chunks from the game server, three at a time (HTTP Range requests), with a progress bar, MB/s and the time left. **Load from file…** does the same from a copy of the MP4 already on that computer (section 3a).
3. **Each chunk is checked** for length and SHA-256 as it arrives, and is then stored in the browser's **IndexedDB**. Only a few 4 MB chunks are held in memory at a time.
4. **The station verifies the result before marking itself ready.** It checks that every chunk is present, the total size is right, the browser can open the file, and the duration matches the server's.
5. **Playback uses a `blob:` URL** built from the stored chunks. The blob refers to them rather than copying the video into memory. There is one URL at a time, and it is revoked when it's replaced or when the page closes.
6. **There is no network fallback.** If a station has no prepared copy, the cinematic does **not** stream. During the cinematic that station shows "Cinematic unavailable on this station" and reports it to the supervisor.

### Station states

| State | Meaning |
| --- | --- |
| Not downloaded | No copy yet. If part of a download was saved, it shows "x% saved" and the button reads **Resume download**. |
| Downloading | Shows progress and speed. |
| Verifying | The browser is opening the stored copy. |
| Ready — stored on this computer | The copy is valid and is the current version. |
| Outdated | The server's video changed. **Update cutscene** deletes the old copy and downloads the new one. |
| Failed | Shows the reason, for example a lost connection, a storage-quota problem, a hash mismatch, a wrong file or an unsupported codec. It offers **Resume download**, **Try again** or **Check again**, plus **Load from file…**. |
| Not supported | This browser offers no usable storage, for example some private windows. |

### Handled cases

- **Refresh mid-download:** the next download resumes from the stored chunks.
- **Network loss:** the station shows "Download interrupted… Press Resume".
- **The server's file changes:** the server checks every 10 s, and stations switch to Outdated.
- **Storage full:** the station shows a clear message.
- **A copy removed by the browser:** the station is back to "Not downloaded" after a reload.

### 3a. Any video per station: Load video from file

**What it does:** on any station, **Load video from file…** turns **any video** on that computer into its cinematic. MP4 (H.264) plays everywhere; WebM works in Chrome, Edge and Firefox.

- **Checked before it's stored:** the browser must be able to play the file, otherwise you get a clear error.
- **Stored and played locally:** the video is kept in this browser's storage and played from it. It is **never uploaded to the game server**, and the server sends no video data. Only the start time comes from the server.
- **Survives reloads:** **Use a different video…** replaces it. **Use the official cinematic instead** downloads the server's file.
- **Per station:** each station can have its own video.

**Timing stays with the server.** The cinematic phase always lasts the **planned length**: the server's `CUTSCENE.mp4` duration (1:59), or `GENESIS_CUTSCENE_DURATION_MS` if the server has no file. Then the mission clock starts for everyone.

- **Shorter video:** when it ends, the station shows a **"TRANSMISSION COMPLETE — Stand by"** screen with the time until the mission starts.
- **Longer video:** it's cut when the mission clock starts.
- **What the station shows:** its video's length compared with the planned length.
- **What the supervisor sees:** "Own video · m:ss", and "Video ended · stand-by" during the cinematic.

**Running without a cinematic file on the server** works too. Every station then needs its own video:

```powershell
$env:GENESIS_CUTSCENE_DURATION_MS="119000"   # 1:59
npm start
```

**Slow Wi-Fi:** downloading the official file over a slow link can take a very long time. Loading a video from a file avoids the network entirely. `/diag` has a **Download speed from the game server** test.

### Why IndexedDB, not the Cache API

Stations open the game over plain `http://<laptop-ip>`. Browsers only provide the Cache API, `crypto.subtle` and `navigator.storage.persist()` on HTTPS or localhost. IndexedDB works on plain HTTP and stores large blobs on disk. That's also why hashing uses a small built-in SHA-256, which is tested against Node's crypto.

### Limitations (honest version)

- **It's browser storage, not an installation.** Clearing site data, using a private/incognito window, a different browser or browser profile, or the browser freeing space can remove it. Each browser and profile on each computer keeps its own copy.
- **Persistence can't be requested** on plain HTTP, so the storage is "best effort". Safari may also delete data for sites you haven't used for about 7 days. Open the station page and check it shows **Ready** before each game day.
- **A partial download from before this update** (8 MB pieces) restarts from 0%; a **complete** copy keeps working.
- **Storage space:** each station needs about 105 MB free in browser storage. Normally that's fine on laptops.
- **One tab per station:** the same browser and profile shares one copy across its tabs.
- **Readiness:** the supervisor can't start until every **active** station reports a Ready copy of the **current** version. The server checks the version. **Start anyway…** still exists for deliberate override; unprepared stations then show the error card, and nothing is streamed.

If browser storage proves unreliable on your machines, the smallest alternative is to run the game server on every operator laptop just to serve its own local copy. I have **not** built that, because it adds a process per laptop. IndexedDB should be tried first.

---

## 4. Mission lifecycle, tutorials and the ready wait

```
LOBBY → CUTSCENE → TUTORIAL ──(last required station presses Finish Tutorial)──→ ACTIVE_MISSION → TIME_EXPIRED
                      │  each station: tour → Finish Tutorial → ready wait
```

- **CUTSCENE** works exactly as before. When the server's cinematic timer ends (or the supervisor skips it) the server switches to **TUTORIAL**. No clock is started.
- **TUTORIAL.** Each active station gets its role's tour. Its current step is saved on the server, so a refresh or reconnect resumes on the same step. The cinematic is not replayed.
- **Finish Tutorial** sends `operator:tutorialComplete` with the session id. The server records `completedAt` once; repeats are accepted and change nothing. A request from an older session (a stale tab after a reset) is refused.
- **Ready wait.** A finished station shows *Tutorial complete — you're ready*, a list of the active stations (Ready / In tutorial · step 3 of 8 / Not connected), and **Review the tutorial again**. Reviewing is local only — it sends nothing and can't un-ready the station.
- **Start of the clock.** In the same server call that records the last completion, the server checks that every required (active) station has finished **and is connected**. If so it sets `phase = ACTIVE_MISSION`, `missionStartedAt` and `missionDeadline = now + 8 min` and broadcasts one state update. Node runs one handler at a time, so two stations finishing in the same millisecond still produce exactly one start. No grace period is added.
- **Disconnects before the start.** A station that finished but is disconnected holds the start. When it reconnects (and everyone else is ready) the clock starts then. A station that never connected also holds it. The supervisor sees why.
- **Disconnects after the start** don't affect the mission; reconnecting restores the current state and the same deadline.
- **Reset** creates a new session with fresh tutorial state.
- **Supervisor override:** *Start the clock now…* (with confirmation) starts the mission during the tutorial phase without waiting. It's for emergencies such as a crashed station.
- With `GENESIS_ALLOW_PARTIAL_START=1`, only connected active stations are required.

### The tutorials

| Role | Steps |
| --- | --- |
| Puzzle Solver (1, 4) | 1 Welcome · 2 Mission timer (8 min, not running yet) · 3 Executor video preview (placeholder — says so) · 4 The five puzzle cards (minigame earns clues, doesn't solve the physical puzzle) · 5 Clue handoff to the analyst → **Finish Tutorial** |
| Clue Analyst (2) | 1 Welcome · 2 Incoming clues · 3 Sort into a box · 4 Each box is a group · 5 Review groups (unsorted clues aren't sent) · 6 Send to Communicator · 7 The communicator relays it; the supervisor verifies, not you → **Finish** |
| Mission Communicator (3) | 1 Welcome (you don't solve or sort) · 2 Executor video preview · 3 Incoming clue list · 4 Ready-to-relay briefings · 5 Tell the executors, then *Mark as relayed* · 6 The supervisor verifies physical puzzles → **Finish** |
| Analyst & Communicator (5) | 1 Welcome (you do both jobs because the team has two operators) · 2 The five-step bar · 3 Incoming clues · 4 Sort and group · 5 Review · 6 Send / prepare · 7 Communicate tab (the tour switches tabs itself) · 8 Executor view and verification → **Finish** |

During the tutorial, empty areas show clearly labelled **example** cards (EX-1…EX-3, an example briefing) so there is something to point at. They are never sent to the server.

The overlay (`client/src/components/tutorial/TutorialTour.tsx`) is reusable. It finds its target by a `data-tour` attribute and follows it every animation frame (scrolling, resizing, tab switches). The popover goes below, above, right or left — whichever fits — and is always kept inside the window. Keys: → / ← change step, Tab stays inside the popover. During the mission, **? Tutorial** in the header re-opens it as a review.

## 4a. Operator workflow (Stage 5)

All of this state lives on the server; refreshing or reconnecting restores it. Workflow actions are accepted only during ACTIVE_MISSION, and everything is read-only after TIME_EXPIRED.

```
Solver minigame ──complete──▶ clue packet (3 terms) ──analyst submits──▶ correct: clue unlocked ──▶ Communicator relays (Relayed: Yes)
                                                                      └─▶ wrong: packet LOST ──▶ solver replays the SAME minigame ──▶ new packet (attempt 2…)
Physical puzzle: only the supervisor's Verify marks it solved (never the minigame, the clue or the relay).
```

**Binary Switches (real minigame).**

- Each session the server secretly shuffles the weights 1, 2, 4 … 128 across the eight shapes of the physical board, and picks a physical target from the curated list `37, 53, 86, 109, 146, 173, 197, 218, 229, 245`.
- The solver sees a decimal target, eight ON/OFF switches labelled 128 … 1, the binary and decimal readout, and **Check answer**. The server checks the switches; a wrong answer is shown (“your switches make 87, the target is 86”) and does not count. Round targets come from the same list, never the physical target.
- **Three correct rounds** release one clue packet. Retried requests are counted once; round progress survives refresh and reconnect.
- The unlocked clue (BS-MAP) lists every shape with its value, in board order, plus the physical target. It is built from the same hidden mapping the supervisor sees.
- The shapes and their left-to-right order on the physical board are set in `shared/src/binary.ts` → `BINARY_BOARD` (default: triangle, circle, square, diamond, star, hexagon, arrow, wave). **Change it if the real board differs.**

**TERM CLASSIFICATION (analyst, and panel A on station 5).**

- A packet holds three terms from three different boxes: **Hardware** (components inside the computer: CPU, RAM, GPU, Motherboard, Power Supply), **Input Devices** (Mouse, Keyboard, Microphone, Scanner, Webcam), **Output Devices** (Monitor, Printer, Speakers, Projector, Headphones), **Computer Fundamentals** (Binary, Bit, Byte, Algorithm, Kilobyte). The list is in `server/src/game/puzzleSource.ts`; the correct boxes never reach a browser.
- Drag a term onto a box, or tap the term and then the box (works on touchscreens). Placements are saved on the server immediately.
- **Submit classification** needs all three placed. Correct → *✓ Correct — clue unlocked*. Wrong → *✗ Packet lost — 2 of 3 terms were right* (never which ones), and the solver's card shows **Replay needed**. The regenerated packet has new terms and is marked *Regenerated · attempt 2*.

**Communicator (station 3, and panel B on station 5):** unlocked clues with **Relayed: No / Yes** and **Mark as relayed**; a status row per puzzle (locked / awaiting classification / lost / regenerated / unlocked / relayed).

**Supervisor:** the *Binary Switches · answer* panel and the *Clue pipeline* panel (section 5). Verification is unchanged.

**Security note:** the server sends the hidden answers only to sockets that joined as supervisor; every operator snapshot has `answerKey: null`. There is still no login, so anyone who opens `/supervisor` sees the answers.

## 4b. Stage 6 puzzles

All three use the Stage 5 pipeline unchanged: minigame → packet of 3 terms → classification (wrong = packet lost, replay **that** minigame with a fresh challenge) → clue → relay → supervisor **Verify**. Each puzzle has its own four boxes:

| Puzzle | Boxes |
| --- | --- |
| Binary Switches | Hardware · Input Devices · Output Devices · Computer Fundamentals |
| Algorithm Sequencer (Stage 7) | Sorting Algorithms · Searching Algorithms · Programming Constructs · Programming Languages |
| Shortest Path | Graph Parts · Algorithms · Data Structures · Networking |
| Logic Gates | Logic Gates · Number Systems · Storage Units · Electronic Components |
| Parity Matrix | Error Detection · Storage Devices · Software · Graphics |

The terms and their correct boxes are in `server/src/game/puzzleSource.ts` → `TERM_POOL` (server only).

**Shortest Path.**
- *Digital:* a new 6-node graph (A–F) each session and each replay: 7–8 edges chosen from a non-crossing layout, weights 1–9, connected, endpoints never neighbours. Tap nodes in order (tap the last node again, **Undo** or **Clear** to change), **Submit path**. The server checks the route follows edges, starts/ends at the endpoints and repeats no node, adds the weights, and compares with its own optimum (Dijkstra for the shortest; exhaustive search of simple paths for the longest). Any route tied for the optimum is accepted; a valid but worse route shows its cost. Phase 2 (longest **simple** path) uses the same graph. Both phases are needed for the packet.
- *Physical:* the classroom graph is fixed in `shared/src/puzzles.ts` → `PHYSICAL_GRAPH`: nodes **A, B, C, D, E, F** and 8 edges **A–B, A–C, B–C, B–D, C–E, D–E, D–F, E–F**. Each session the server gives every edge a random weight 1–9 and computes the shortest **A → F** route(s) and cost. The clue (SP-MAP) lists every edge with its weight.

**Logic Gates.**
- *Digital:* five server-chosen questions over inputs A, B, C (0/1): 1 = one AND/OR gate, 2 = a different single gate, 3 = two gates, 4 = three gates with brackets, 5 = AND, OR and NOT together. Every compound operand is bracketed, so no question depends on precedence. AND / OR / NOT buttons slide their truth tables open and shut. Answer 0 or 1; wrong answers don't count; retries are counted once.
- *Physical:* the lamp target (never all on or all off) is chosen at session start. The clue (LG-LAMPS) reads “Lamp A ON, Lamp B OFF, Lamp C ON”. The five questions are unrelated to the lamps.

**Parity Matrix.**
- *Digital:* a 5 × 5 grid of red tiles with one yellow. **Start** turns it red and plays a server-generated shuffle (10 steps of 3 simultaneous swaps between neighbouring tiles; the tracked tile moves in 7 of 10 steps; it starts slow — 1 s per step — and speeds up to 0.5 s). Tiles step aside and pass at half size, so they never overlap. Tap the tile that was yellow. The server knows where it ended up and judges the pick. A wrong pick says so without revealing the right tile and deals a new yellow tile and a new shuffle.
- *Physical:* the rule (**even** or **odd**: every row and column must contain an even/odd number of filled cells) is chosen at session start; the supervisor also gets a valid example grid with row/column counts. The clue (PM-RULE) states the rule.

**When answers are created:** all physical answers are generated when a session is created (server start, and every **Reset session**), so the supervisor sees them in the lobby. Refreshing a page or reconnecting never changes them.

**Security note:** the tracking game must send the shuffle to the solver's browser so it can be animated; someone reading the network traffic could work out the answer. The server still makes the decision. All physical answers and digital optimal answers stay on the server (supervisor gets the physical ones).

## 4c. Stage 7 — Algorithm Sequencer, final cipher, Mission Success, records

```
ACTIVE_MISSION ──(supervisor verifies the 5th physical puzzle)──▶ FINAL ──(supervisor: Verify final cipher)──▶ SUCCESS
        └──────────────── deadline ────────────────▶ TIME_EXPIRED ◀──────────── deadline ─────┘
```

**Algorithm Sequencer.**
- *Session answer:* when a session is created the server picks one of Selection, Insertion, Merge or Bubble Sort, and generates precedence rules for the five step cards. The supervisor sees the algorithm, the five cards in order and the rules from the lobby onwards.
- *Digital:* the solver sees the algorithm's name, a short explanation, its five steps (the same text as the physical cards), and a list of six different numbers (shuffled, 6–10 pairs out of order). The playground offers only that algorithm's moves, and the server applies each move to its own copy of the list:
  - Selection — tap the smallest number of the unsorted part, then *Swap into place*.
  - Insertion — the key is highlighted; *Move key left* while the number on its left is bigger, then *Insert here*.
  - Bubble — the current neighbour pair is highlighted; *Swap* only if the left one is bigger, otherwise *In order*. Passes shrink; the run ends with a pass that makes no swaps.
  - Merge — *Split* every group until each holds one number; then merge partner groups by tapping the smaller front number (the rest follows automatically when one side is empty).

  A move that is not the algorithm's next step is refused with an explanation and changes nothing. *Reset List* restores the starting list (same algorithm) and restarts the hint timer. After 30 s a hint names the next step for the current list; it stays until dismissed.
- *Clue:* three terms (Sorting / Searching / Constructs / Languages). Correct → **AS-ORDER**: rules such as “Find the smallest” comes before “Place it first”. The rules always allow exactly one order (checked by trying all 120 orders). Wrong → packet lost; the replay deals a new list with the same algorithm.

**The five physical cards** (title — text). Print the title large on each card; the clue refers to cards by title. The texts live in `shared/src/sequencer.ts` → `ALGORITHMS`.

| Algorithm | 1 | 2 | 3 | 4 | 5 |
| --- | --- | --- | --- | --- | --- |
| Selection Sort | Unsorted portion — Begin with the unsorted portion of the list. | Find the smallest — Find the smallest element in that portion. | Place it first — Place it at the first unsorted position. | Mark and repeat — Mark that position as sorted and repeat. | All sorted: stop — Stop when every position is sorted. |
| Insertion Sort | First is sorted — Treat the first element as a sorted portion. | Take the key — Take the next element as the key. | Compare the key — Compare the key with elements in the sorted portion. | Shift and insert — Shift larger elements and insert the key in its correct position. | Repeat to the end — Repeat until every element is sorted. |
| Bubble Sort | Start at the beginning — Start at the beginning of the list. | Compare neighbours — Compare adjacent elements. | Swap if left is larger — Swap them when the left element is larger. | Continue the pass — Continue through the list and repeat passes as necessary. | Stop: no swaps — Stop when the list is sorted and a pass makes no swaps. |
| Merge Sort | Divide in half — Divide the list into smaller halves. | Divide to single items — Continue dividing until each sublist has one element. | Compare front items — Compare the first available elements of two sorted sublists. | Merge in order — Merge the sublists in ascending order. | Merge into one list — Continue merging until one fully sorted list remains. |

**Final cipher.**
- *Session answer:* plaintext `RESCUED` (or `GENESIS_FINAL_WORD`), shift 1–25, ciphertext = every letter moved forward by the shift (Z wraps to A). On the supervisor phone from the lobby: the ciphertext to put on the board, the plaintext, the shift and the full position-by-position mapping.
- *Start:* the moment the supervisor verifies the fifth physical puzzle, the server switches to FINAL (one update to every device). The mission clock keeps running; the five puzzle modules close.
- *Solver:* FINAL PUZZLE — DECODE THE MESSAGE, the letter slots, “What letter is X positions before C?” for each ciphertext letter left to right, an A–Z table (letters only) and a one-letter input. Wrong letters are counted and do not advance; repeated letters (RESCUED has two E's) are separate positions.
- *Analyst / station 5 panel A:* the mappings decoded so far (“Letter 2 · Cipher S → E”), nothing for unsolved positions. *Communicator / station 5 panel B:* each mapping with what to say and **Mark as relayed**. Station 5 shows both side by side — no tabs.
- *Executors:* write each relayed letter under the matching ciphertext letter on the board.
- *Supervisor:* compares the board with the plaintext and presses **Verify final cipher…** (two-step confirm). Answering every digital question never ends the game by itself.

**Mission Success.** SUCCESS is terminal until a reset: workflow actions are refused, the clock stops at the completion time, puzzle states and the decoded word are kept, and a refreshed device returns to the same screen. Stations show MISSION SUCCESS, the decoded word, the mission time and each puzzle's split time; the supervisor keeps every panel plus a success banner.

**Timing and records.**
- All timestamps come from the server. In production the server clock is the wall clock at start-up plus a monotonic timer, so a Windows clock adjustment mid-game cannot distort durations.
- *Mission time* = ACTIVE_MISSION start → final verification (cinematic and tutorial excluded). *Final cipher time* = FINAL start → final verification.
- Per puzzle: first time the solver opened the minigame (never restarted by Reset List or a lost packet), digital completion (the play-through whose packet unlocked the clue), physical verification, digital and end-to-end durations, split time after the mission start, play-throughs, packets lost (each replay is listed) and wrong answers.
- Session status: *success*, *time-expired*, *aborted* (reset mid-mission), *interrupted* (server stopped mid-mission), *in-progress*. Puzzle status: *solved*, *incomplete*, *aborted*.
- The record is written to `data/session-records.json` whenever a timing changes (temporary file + rename). Records are never deleted by the game. A session is recorded once its mission clock starts.
- Supervisor phone: **Session timing** (live) and **Records → Session history** (tap a row for details; **Export CSV** downloads a spreadsheet file — durations in seconds, one row per session).

### Stage 4 version of this section (replaced)

- **Puzzle Solver:** five large cards. A card opens its placeholder minigame; **Solve Puzzle** releases two deterministic `[DEMO]` clues. The new **Clues earned** panel lists them; they go straight to the analyst.
- **Clue Analyst — CLUE SORTING:**
  - **Incoming clues**: cards with a symbol (triangle, circle, square…), a code such as BS-1, a short label, the text and a hint (“Hint: Code Clues”).
  - **Four boxes**: Shape Clues, Code Clues, Sequence Clues, Path Clues. These are placeholder categories for the interface, not the final puzzle rules.
  - **Click a clue, then click a box** — or drag it. Click a sorted clue and another box to move it; *Put back in incoming* un-sorts it. Esc drops a picked-up clue.
  - A card shows *Saving…* until the server confirms, then the box flashes green with *✓ BS-1 saved here*.
  - **Each box is a group.** *Review groups* lists what's in each box, the counts (clues / to sort / sorted) and which clues still need sorting.
  - **Send to Communicator** (optional one-line note). The server builds the briefing from **its own saved sorting**, so the client can't send a clue that doesn't exist or wasn't saved. Unsorted clues aren't sent. Success is shown only after the server confirms (*✓ Briefing #2 received…*). A retried request is ignored, and sending the same sorting again is refused (*Nothing new to send*).
  - Each briefing contains everything sorted so far; clues new or moved since the previous one are marked **NEW** for the communicator.
  - *Sent to communicator* lists briefings with *Waiting* / *Relayed*.
- **Mission Communicator:** the largest video preview on the left; **Ready to relay** (newest briefing in large text, grouped by box, with **Mark as relayed**), the clue list with each clue's box, and the activity feed on the right. Relaying the newest briefing also covers older ones still waiting, since it contains them (they show *Included in #2*).
- **Station 5:** the same sorting game. A five-step bar (*Incoming → Sort → Review → Send / prepare → Communicate*) and two tabs: **Sort clues** and **Communicate**. After sending, *Go to Communicate →* switches tabs. Nothing is typed twice.
- **Supervisor:** physical verification is unchanged and remains the only way to mark a physical puzzle solved.

## 5. Supervisor additions

**Stage 7:**

- **Algorithm Sequencer · answer** — algorithm, the five cards in order, the AS-ORDER rules, and the usual minigame/clue/relayed/physical status.
- **Final cipher · answer** — ciphertext to place on the board, plaintext, shift, the position table (cipher, plain, and whether the team has it: • released, ✓ relayed), and **Verify final cipher…** during FINAL. It moves to the top of the page during FINAL and SUCCESS.
- **Session timing** (live) and **Records** (session history, details, CSV export). The *Final puzzle — reserved* placeholder is gone.

**Stage 6:** three more answer panels, each with the minigame / clue / relayed / physical status:

- **Shortest Path · answer** — the shortest A → F route and its cost (ties listed), the classroom graph drawn with this session's weights (route highlighted), and the weight of every edge.
- **Logic Gates · answer** — Lamp A / B / C with ON/OFF.
- **Parity Matrix · answer** — EVEN or ODD, how to check (“every count must be 0, 2 or 4”), and a valid example grid with its row and column counts.

**Stage 5:**

- **Binary Switches · answer** (supervisor only): the physical target in decimal and binary; the eight board positions left to right, each with its shape, its secret value and **ON/OFF**; the sum (e.g. `32 + 4 + 1 = 37`); and the minigame, clue, relayed and physical-verified state. Compare it with the board, then press **Verify** below.
- **Clue pipeline**: per puzzle — minigame progress, latest packet (awaiting / lost / regenerated / unlocked), relayed, physical.
- Mission facts now show clues unlocked, clues relayed and packets lost.

**Stage 4:**

- **Mission state** during the tutorials: *x of N operators ready*, each active station's status, and *Not running — starts when every operator is ready*. After the start: the authoritative countdown (same deadline as every station).
- **Station table**: the last column becomes **Tutorial** during the tutorial phase (*In tutorial · step 3 of 5*, *Tutorial done · waiting*, *Ready · disconnected*).
- **Launch panel**: *Skip cinematic…* now goes to the tutorials. During the tutorials, *Start the clock now…* is an emergency override. You don't need to press anything normally.
- Unchanged: team selector, Start Mission, verification controls, activity, cinematic info, reset.

---

## 6. Network, firewall and phone access

These are unchanged from Stage 2.

- **Firewall:** allow Node on **Private** networks, and set your Wi-Fi profile to **Private**. If needed, open PowerShell as Administrator and run:
  `New-NetFirewallRule -DisplayName "Project Genesis" -Direction Inbound -Protocol TCP -LocalPort 3001,5173 -Action Allow -Profile Private`
- **If a device can't connect:** open `/diag` on it and check every line.
- **OneDrive:** the project lives in OneDrive. If `npm install` is slow, pause syncing.

---

## 6a. HTTPS for the Communicator microphone (executor M4)

The Communicator talks to both executor phones by voice. Browsers only allow the microphone on a **secure page**: `https://…` or `localhost`. On plain `http://<laptop-ip>:3001`, `window.isSecureContext` is `false` and `navigator.mediaDevices` is `undefined`, so the microphone cannot even be requested. The server can therefore also listen on HTTPS **in addition to** HTTP. Everything else (other stations, supervisor, phones) keeps using HTTP. No Chrome flags, no public hosting, no internet.

**How it fits together.** One Node process, two listeners: HTTP on `PORT` (3001) and HTTPS on `GENESIS_HTTPS_PORT` (3443). They share the same Express app, the same Socket.IO/Engine.IO server and the same game state. The dashboards connect to their own origin, so an `https://` page automatically uses `wss://` on 3443 (no mixed content), while the phones and other stations stay on `ws://` 3001. Voice and video signalling is relayed between both listeners' sockets as before. HTTPS needs **production mode** (`npm run build` + `npm start`): in `npm run dev`, Vite serves the dashboards over HTTP only, so 3443 has no pages (the start-up log says so). In dev, open the Communicator on the laptop itself at `http://localhost:5173` instead (localhost counts as secure).

**Certificates.** `npm run https:setup` makes its own small local CA ("Project Genesis Local CA", ECDSA P-256, 10 years) and a server certificate signed by it (800 days) with Node's built-in crypto. It plays the same role as mkcert but needs nothing to download. The CA is trusted once per Communicator computer. When the laptop's address changes, only the server certificate is remade, so nothing needs re-trusting. (A certificate from mkcert also works: save it as `certs\server.crt` / `certs\server.key`.)

**1. On the game laptop (PowerShell, in the project folder)**

```powershell
npm run https:setup          # creates certs\ (git-ignored): local CA + server certificate
npm run build
npm start                    # also prints: HTTPS (Communicator microphone) on port 3443, and the https:// URLs
```

The server certificate covers `localhost`, `127.0.0.1`, the computer name (`<name>` and `<name>.local`) and **every current IPv4 address**. When the laptop gets a new address (another Wi-Fi or a phone hotspot), run `npm run https:setup` again and restart the server. The start-up log warns when the current Wi-Fi address is missing from the certificate. Extra names or addresses: `npm run https:setup -- 192.168.43.1`. Check the current address with `ipconfig` (Wi-Fi adapter, IPv4 Address).

**2. Once per Communicator computer: trust the CA.** Copy `certs\genesis-local-ca.crt` (**never** the `.key`) to that computer, e.g. on a USB stick.

- Windows, PowerShell (no admin needed; Windows asks you to confirm "Project Genesis Local CA", click **Yes**):
  `certutil -user -addstore Root genesis-local-ca.crt`
  Or double-click the file → *Install Certificate* → *Current User* → *Place all certificates in the following store* → **Trusted Root Certification Authorities**.
  Chrome and Edge use the Windows store. **Close every browser window** and reopen it afterwards.
- macOS: double-click → keychain *login* → open the certificate → *Trust* → **Always Trust**. Restart the browser.
- Firefox uses its own store unless `security.enterprise_roots.enabled` is on (the default on Windows in current versions). Chrome or Edge is recommended for the Communicator.

If the laptop itself is the Communicator computer, run the `certutil` line there with `certs\genesis-local-ca.crt`.

**3. Open the Communicator station** at `https://<laptop-ip>:3443/operator?station=3` (five players) or `station=5` (four players), e.g. `https://192.168.0.101:3443/operator?station=5`. Use the IP address from other computers: the computer name only works where the LAN resolves it. On the laptop itself, `https://localhost:3443/…` works too. The HTTPS address is its own browser origin, so the station downloads its local copy of the cinematic once more (*Download / Prepare Cutscene*). The browser asks for the microphone when the executor tutorial starts: click **Allow**. If the Communicator opens the usual HTTP address, the voice bar says the microphone needs the secure address and links to it.

**4. Check it.** The address bar shows a padlock with no warning. In DevTools (F12) → Console:
`window.isSecureContext` → `true`, `typeof navigator.mediaDevices?.getUserMedia` → `"function"`. During the tutorial the voice bar shows the microphone level and `EX-01 · connected`, `EX-02 · connected`. The Supervisor's executor table shows the same. If the voice bar says *Mic blocked*, allow the microphone with the padlock / site-settings icon in the address bar, then reload. *No microphone found* means Windows has no input device (Settings → System → Sound → Input).

A command-line check from a computer that trusts the CA: `curl.exe --ssl-revoke-best-effort https://<laptop-ip>:3443/api/health`. Windows' curl hard-fails on revocation checks by default, and local certificates (also mkcert's) have no revocation server. Browsers do not need this.

**Firewall (game laptop).** Inbound TCP **3443** must be allowed, as 3001 already is. If Node.js was allowed for the current network type when Windows first asked, all its ports are already open. Otherwise, in an Administrator PowerShell:
`New-NetFirewallRule -DisplayName "Project Genesis HTTPS" -Direction Inbound -Protocol TCP -LocalPort 3443 -Action Allow -Profile Private`
(and set the Wi-Fi profile to **Private**, or use `-Profile Private,Public` for a network you trust). Check with `Get-NetConnectionProfile` and, from the Communicator computer, `Test-NetConnection <laptop-ip> -Port 3443`.

Settings: `GENESIS_HTTPS_PORT` (default 3443), `GENESIS_CERT_DIR` (default `certs` in this folder), `GENESIS_HTTPS=0` turns the HTTPS listener off. If 3443 is taken, the server keeps running on HTTP and logs that HTTPS is off. `/api/health` reports `httpsPort` only while HTTPS really serves the dashboards.

**Security note:** `genesis-local-ca.key` can sign certificates that the trusting computer accepts for *any* site. Keep it on the game laptop only, never share or commit it (`certs/` is git-ignored). If it leaks, or when the game is retired, delete `certs\` and remove the CA from every trusting computer: `certutil -user -delstore Root "Project Genesis Local CA"`.

---

## 7. Files changed in Stage 4

**New**

- `client/src/components/tutorial/TutorialTour.tsx` — reusable spotlight + popover tour
- `client/src/components/tutorial/tutorialSteps.tsx` — the four role tutorials
- `client/src/components/tutorial/ReadyWait.tsx` — ready-wait card and shared status labels
- `client/src/components/tutorial/stepHelpers.ts`
- `client/src/styles/stage4.css` — tutorial, sorting game, briefings, readability

**Changed**

- `shared/src/game.ts` — TUTORIAL phase, `StationTutorial`, `TUTORIAL_STEP_COUNTS`, `tutorialStartedAt`
- `shared/src/events.ts` — `operator:tutorialProgress`, `operator:tutorialComplete`, `supervisor:startClockNow`
- `shared/src/workflow.ts` — four sorting categories, clue symbol + hint, briefings with groups; old link/note/draft actions removed
- `server/src/game/GameStore.ts` — tutorial phase, completion, atomic clock start, override
- `server/src/game/workflow.ts` — sorting, server-built briefings, relay covers older briefings
- `server/src/validation.ts`, `server/src/realtime.ts` — new events
- `server/src/game/GameStore.test.ts`, `workflow.test.ts`, `realtime.test.ts`
- `client/src/components/operator/RoleDashboards.tsx` — tutorial orchestration, new layouts
- `client/src/components/operator/analysis.tsx` — rewritten as CLUE SORTING
- `client/src/components/operator/communication.tsx`, `shared.tsx`, `SolverDashboard.tsx`, `OperatorShell.tsx`
- `client/src/components/supervisor/MissionClock.tsx`, `OperatorPresence.tsx`, `client/src/pages/SupervisorDashboard.tsx`
- `client/src/components/CountdownDisplay.tsx`, `client/src/game/selectors.ts`, `client/src/realtime/useGameConnection.ts`, `client/src/main.tsx`, `client/src/pages/OperatorDashboard.tsx`
- Wording only: `CutsceneOverlay.tsx` (“Operator training starts in…”), `LocalCutscenePanel.tsx`
- `README.md`, `TEST_CHECKLIST.md`

The cinematic file, its cache code, playback sync and networking are not modified.

---

## 8. Test results

### Automated and browser checks (Stage 7, 2026-10-12)

- `npm run check` passes: typecheck, **96 server tests**, production build. New: `server/src/game/stage7.test.ts` (algorithm rules for all four sorts, random-move fuzzing, unique card order over 300 sessions, cipher generation and wraparound, FINAL/SUCCESS transitions, role checks, timing records, reset/abort/expiry, JSON file persistence across a restart, CSV export) and a Socket.IO final-phase test in `realtime.test.ts`.
- Real Chrome on the Windows laptop (headless, production build): supervisor at 390 px plus stations 4 and 5 played the full flow through the UI — Parity grid keeps 400 px and stays centred during the shuffle; sequencer wrong move, 30 s hint, completion; classification → AS-ORDER; five verifications → FINAL; wrong and right cipher letters; relay; Verify final cipher → MISSION SUCCESS on every screen (also after a refresh); history and no horizontal scroll at 390 px; records kept after reset and server restart. **23/23 checks, no page errors.** All four algorithms were also completed through the UI.
- **Not yet tested on your devices** (MacBook, phone, second laptop) — see `TEST_CHECKLIST.md` section 2c.

### Automated (Stage 4)

`npm run check` passes: typecheck, **38 server tests**, production build.

New or changed server tests:

- Lifecycle CUTSCENE → TUTORIAL (no deadline, no time limit) → ACTIVE_MISSION only after all tutorials → TIME_EXPIRED.
- One station finishing doesn't start; the last one starts **once**, with deadline = now + 8 min, in **one** broadcast; duplicate and late completions change nothing.
- Completion refused for another session, an inactive station, or outside TUTORIAL.
- Progress is stored, clamped to the role's step count, and survives disconnect/reconnect; reports after finishing are ignored.
- A finished-but-disconnected station holds the start until it reconnects; a never-connected station holds it too.
- Supervisor override only during TUTORIAL. Three-operator team ignores stations 4–5.
- Sorting: role checks, unreleased clues refused, move between boxes, unsorted clues not sent, idempotent retry, *Nothing new* refusal, NEW markers, relaying #2 covers #1.
- Socket.IO flow: tutorial progress/completion over sockets, stale session refused, one deadline for all, sort → send → relay → verify.

**Browser runs, Stage 4** (headless Chromium, production build, accessed by network IP, one browser profile per station, 20 s VP9 test cinematic, real 8-minute clock)

| Requirement | Two operators (4, 5) | Three operators (1, 2, 3) |
| --- | --- | --- |
| Cinematic ends by itself → tutorial on every station | ✔ (station 5 used its own 8 s video → stand-by, then tutorial) | ✔ via Skip cinematic |
| Timer shows 08:00, *Not running* during the tutorial | ✔ | ✔ |
| Correct tutorial per role (5 / 8 and 5 / 7 / 6 steps) | ✔ | ✔ |
| Spotlight sits on the right element; popover fully on screen (1280×720, 1366×768, 1440×900) | ✔ every step | ✔ every step |
| Next, Back, ← / → keys, Finish Tutorial | ✔ | ✔ |
| Refresh mid-tutorial resumes on the same step; cinematic not replayed | ✔ | — |
| One station finishing early → ready wait, clock still stopped | ✔ | ✔ |
| Review again → *Close review* → still ready; refresh after finishing shows ready wait, not the tour | ✔ | — |
| Supervisor sees *x of N ready*, steps, *Tutorial done · waiting* | ✔ | ✔ |
| Last finisher starts ACTIVE_MISSION on every screen; exactly one start entry | ✔ | ✔ with stations 2 and 3 finishing within milliseconds |
| Supervisor and all stations show the same remaining time | ✔ 07:59 ×3 | ✔ ×4 |
| Sort by click, by drag-and-drop, move between boxes; saved feedback; persists across refresh | ✔ | ✔ |
| Send → server-confirmed success; communicator receives groups + text; unsorted clue not sent; NEW marker on the 2nd briefing | ✔ | ✔ |
| Mark as relayed → analyst sees *Relayed* | ✔ (same station) | ✔ |
| Supervisor verification still works and appears for the stations | ✔ | ✔ |
| Station closed and reopened mid-mission: same deadline, work kept, no tutorial | ✔ | ✔ (analyst refresh) |
| *? Tutorial* review during the mission doesn't affect the phase | ✔ | — |
| Reset → new mission starts tutorials fresh at step 1; *Start the clock now…* override | — | ✔ |
| Station 5 at 390 px wide: no horizontal scroll | ✔ | — |
| No page errors | ✔ | ✔ |

### Stage 3 browser runs (unchanged code paths, still valid)

The rows about the old analyst tools (classify, link, note, draft, Submitted tab) are replaced by the Stage 4 sorting rows above. After Stage 4 the cinematic ends in the tutorial, not straight in the mission.

**Stage 3 browser runs** (headless Chromium, production build, through a network IP; each station a separate browser profile; a 30 MB / 20 s / 4-chunk VP9 test clip, because the test browser can't decode H.264)

| Requirement | Three operators (1, 2, 3) | Two operators (4, 5) |
| --- | --- | --- |
| Start blocked until stations cached | ✔ hint names OP-01–03 | ✔ |
| Download → verified → Ready, played from `blob:` | ✔ 4 range requests | ✔ |
| Refresh mid-download → "28% saved" → Resume → Ready | ✔ | — |
| Network loss mid-download → "Download interrupted" → Resume | ✔ | — |
| Reload a ready station → Ready with **0** new video requests | ✔ | — |
| Server file changed → **Outdated**, Start blocked → Update → Ready | — | ✔ |
| Cinematic synchronized from local copies; no network video requests during playback | ✔ spread 31–33 ms | ✔ 12 ms |
| Correct role per station after the cinematic | ✔ Solver / Analyst / Communicator | ✔ Solver / Analyst & Communicator |
| Inactive station shows standby | ✔ (4, 5 "Not in team") | ✔ station 1 open on standby |
| Solve → 2 DEMO clues in the analyst inbox | ✔ | ✔ (Incoming-clues badge 2) |
| Classify, link, note, draft; **refresh keeps them** | ✔ | ✔ tab switch keeps text |
| Submit → communicator receives → Mark as relayed → analyst sees Acknowledged | ✔ | ✔ same station |
| Supervisor verification reaches every station and the activity feed; minigame status stays separate | ✔ | ✔ |
| Refresh during the mission keeps the clock and the workflow | ✔ | ✔ |
| Station 5 at 390 px wide: no horizontal scroll | — | ✔ |
| Slow link: warning + time left shown | ✔ | — |
| **Own video per station** (8 s WebM on station 4, 30 s MP4 on station 5): short one → stand-by screen, long one cut at the planned end, **0 video requests to the server** | — | ✔ with a 20 s server file and with **no** server file (15 s planned) |
| Non-video file rejected with a clear message; own video survives a reload | — | ✔ |
| Parallel network download (3 at once) | ✔ | — |
| Old 8 MB-piece copies: a complete one stays Ready; a partial one restarts cleanly | ✔ | — |
| `/diag` speed test | ✔ | — |

### Needs manual testing on your devices

None of this was tested on your hardware.

- **The tutorial on the real laptops** — Windows laptop and MacBook, in the browser used on game day: spotlight position, readability from where players sit, Next/Back/keys.
- **Drag-and-drop on the MacBook trackpad / Safari.** Click-then-click always works; drag is the extra.
- **The supervisor page on your phone** during the tutorial phase.
- **Your real `CUTSCENE.mp4`**, caching on each laptop, sound/autoplay — as in Stage 3.

### Known remaining issues

1. **No victory trigger.** The lifecycle ends in TIME_EXPIRED; the SUCCESS phase is still reserved (there was no victory handling before). Reproduce: verify all five puzzles — the clock keeps running.
2. **The armed status isn't remembered after a refresh** (from Stage 3). A refreshed station plays muted with “Tap for sound” unless re-armed.
3. **A station stuck in the tutorial blocks everyone.** That is the intended rule; use *Start the clock now…* on the supervisor in an emergency. Reproduce: close one station's browser during the tutorial — the others wait and the supervisor shows *Not connected*.
4. **During the ready wait, the dashboard behind the card can't be used** (by design, so nothing happens before the clock starts).
5. **Categories are placeholders.** Shape / Code / Sequence / Path and the DEMO clue texts are for the interface only.
