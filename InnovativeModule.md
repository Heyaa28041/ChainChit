Big picture

ChainChit runs a chit fund (ROSCA) online. A group of members each pay a fixed amount every cycle, and one member takes the pot each cycle. In a real chit fund an auctioneer decides who wins, which is where cheating happens. Here, software does the job:

Members bid in a reverse auction: the lowest bid wins.
Fraud checks run on every bid.
Every important action is written to a tamper-proof log.
Architecture
Frontend and backend: Next.js 16 (App Router) with React 19. The pages are in app/(app), and the API routes are in app/api.
Database: PostgreSQL through Drizzle ORM. The schema is in lib/db/schema.ts and the triggers are in lib/db/triggers.sql.
Real-time: Server-Sent Events (SSE).
Crypto: Node’s built-in crypto module (SHA-256, Ed25519, AES-256-GCM).
Gatekeeper: proxy.ts returns 401 for API calls with no session cookie and redirects pages to /login. Role checks happen inside each route, using requireRole.
How a chit actually runs, step by step
1. Registration and KYC
A member registers. The mobile number and email are validated, and the member gets an ID like M1001.
The member uploads an Aadhaar or PAN document. The server:
sniffs the file’s magic bytes, so a renamed .exe is rejected,
enforces the 5 MB limit,
encrypts the file with AES-256-GCM before saving.
The foreman views the document, which is decrypted on the fly and logged, then approves it or rejects it with a mandatory reason.
Only VERIFIED members can be enrolled or bid.
2. Group creation
The foreman creates a group with a pool value (for example ₹1,00,000) and a number of members. The group gets a code like G001.
buildSchedule() generates every cycle’s start and end dates up front.
Each member’s contribution is chitValue / numberOfMembers.
Enrolment is allowed only before the first cycle starts. The group row is locked so two people can’t take the last seat.
3. Contributions go to escrow
Each member pays for the cycle (payment is simulated). The amount must match exactly, and duplicates are rejected.
A confirmed payment is stored with escrowStatus = LOCKED, and a ledger entry is written.
A DB trigger stops anyone changing or deleting locked rows.
4. Bidding (the reverse auction)

The foreman opens the cycle with a timer (default 180s). When a member bids, the server (app/api/cycles/[id]/bid) does this in order:

Checks the window is still open, the member is KYC-verified and enrolled, and the bid is no higher than the pool.
Blocks the bid if the member already holds the lowest accepted bid. This is the self-outbid rule.
Runs evaluateBidFraud (see section 5).
Saves the bid as ACCEPTED or FLAGGED_SUSPICIOUS, with its Z-score.
If flagged, creates a fraud alert and notifies the foreman and oversight.
Writes a ledger entry (BID_SUBMITTED or BID_FLAGGED_FRAUD).
Broadcasts NEW_BID over SSE, so every screen updates instantly.
5. How fraud detection works

For each bid, in order:

Rule A, already won: if the group doesn’t allow repeat winners and this member already won, the bid is flagged ALREADY_WON.
Rule B, win frequency: if repeat winners are allowed, it checks how many of the recent cycles this member won. Too many wins means COLLUSION_WIN_FREQUENCY.
Rule C, statistical underbid: it computes Z = (mean − bid) / σ.
With 4 or more clean historical bids, the mean and σ are calculated live. Otherwise the group’s baseline values are used.
If Z is above the threshold (default 2.5), the bid is flagged UNDERBID_DEVIATION.

Why it works: honest bids cluster around the group’s usual discount, so a bid far below that is a statistical outlier. That usually means a rigged deal or collusion.

6. Closing the auction
When the countdown hits zero, every connected browser calls /close. This is safe because of an atomic claim: UPDATE ... SET status='BIDDING_CLOSED' WHERE status='BIDDING_OPEN'. Only the first call gets a row back and does the work.
The server clock decides expiry, and the browser countdown is corrected for clock skew.
The winner is the lowest bid with status ACCEPTED, and WINNER_DETERMINED is logged.
If the lowest overall bid was flagged, escrow becomes HELD_FOR_REVIEW.
If every bid was flagged, the cycle goes to DISPUTED.
7. Foreman resolves fraud alerts

The foreman clears or confirms each alert (fraud/override). If the ruling changes who rightfully won, the code demotes the old winning bid and promotes the new one, then releases the hold. Every ruling is logged.

8. Settlement and payout

settleCycle runs in one DB transaction and:

Locks the cycle row (FOR UPDATE).
Refuses if a payout already exists, if there is no winner, if the cycle isn’t closed, if the winning bid is flagged, if alerts are unresolved, or if escrow is on hold.
Checks that locked escrow covers the payout.
Sets the transaction flag chainchit.payout = on, which is the only thing that lets the trigger allow escrow release.
Creates the payout, with a dividend per member: the discount is shared equally among members.
Builds the receipt, canonicalises it to JSON, signs it with Ed25519, and stores both.
Writes PAYOUT_RELEASED to the ledger in the same transaction.
Marks the group COMPLETED if this was the last cycle.

After the commit, it updates trust scores, notifies the winner and broadcasts over SSE.

The tamper-proof ledger, in detail

Each entry (logAuditEvent) stores previousHash, entryHash and a signature. The hash is computed as:

entryHash = SHA256( previousHash + {sequence, actor, role, action, entity, group, payload} )
signature = Ed25519-sign( entryHash )
Fork prevention: pg_advisory_xact_lock makes writers take turns, so two simultaneous writes can’t both claim the same “previous” entry.
No rewriting: DB triggers raise an error on UPDATE, DELETE and TRUNCATE.
Verification: verifyChain() walks every entry and recomputes it. It fails with BROKEN_CHAIN, HASH_MISMATCH or BAD_SIGNATURE, and names the exact entry.

Changing any old entry changes its hash, which breaks every entry after it. The signature also shows that only the server’s key could have created the entries.

Real-time implementation
sse-broker.ts is a small in-memory publish/subscribe map keyed by channel (cycle:<id>).
/api/cycles/[id]/stream subscribes the browser to that channel and sends a ping every 15 seconds to keep the connection alive.
On the client, useCycleRoom loads the full state and then applies SSE events. After any reconnect it reloads the full state, so a missed event can’t leave the screen wrong.
Other supporting pieces
Reminders and delinquency: dueState() classifies each unpaid contribution as REMINDER_DUE, DUE, LATE (inside the grace period) or DELINQUENT. runSweep sends notices and flags members.
Disputes: raising one snapshots the bids, contributions, payout and fraud logs at that moment, so the evidence can’t change later.
Exports: the CSV or PDF is generated, then digested and signed. /api/exports/verify lets anyone check it against the public key.
Honest limitations (good to state before sir asks)
Payments are simulated.
The SSE broker lives in one server process, so scaling to several servers would need something like Redis.
In the bid route, the bid insert, the alert insert and the ledger write are separate steps, not one transaction. Settlement is fully transactional, but bidding isn’t. If asked, say that’s a known trade-off for speed.
There’s no real blockchain, only the signed hash chain.

Innovative Modulea:
1. Z-score fraud detector (lib/fraud-detector.ts)
Problem: collusive or fake deep underbids rig the auction.
How: on every bid it computes Z = (mean − bid) / σ. With 4 or more clean past bids it uses live statistics, and otherwise the group’s baseline. A bid with Z above the threshold (default 2.5) is saved as FLAGGED_SUSPICIOUS, and a fraud alert is created.
Say: “Honest bids cluster around the usual discount, so a bid more than 2.5σ below the mean is a statistical outlier.”
2. Collusion rules and trust scores
Problem: the same member keeps winning, or wins twice.
How: ALREADY_WON blocks a second prize. getWinFrequency looks at the last N settled cycles and flags a member who has won too many. updateTrustScores gives members +5 per clean bid and −25 per flagged one, and gives the foreman +3 or −5 depending on whether alerts are resolved.
Say: “Even the foreman is scored, so the supervisor is accountable too.”
3. Hash-chained, signed audit ledger (lib/audit-logger.ts)
Problem: logs can be edited after the fact.
How: entryHash = SHA256(previousHash + entry data), then Ed25519-signed. An advisory lock serialises writers so the chain can’t fork. verifyChain recomputes everything and names the corrupted entry.
Say: “Changing one old record breaks every hash after it, and the signature proves only our server wrote it.”
4. Database-enforced immutability (triggers.sql)
Problem: a bug or a rogue admin could still run an UPDATE on the log.
How: Postgres triggers raise an error on UPDATE, DELETE and TRUNCATE of audit_logs, and block any change to a payout.
Say: “The rule lives in the database, not just in app code, so even direct SQL can’t rewrite history.”
5. Escrow released only through the payout contract
Problem: the foreman could take the pooled money.
How: contributions are stored as LOCKED. The trigger permits release only if the transaction sets chainchit.payout = on, and only settleCycle does that. The withdraw route always refuses and logs the attempt.
Say: “There is no code path, and no SQL path, that moves escrow outside the payout function.”
6. Atomic payout (lib/server/settlement.ts)
Problem: double payouts or paying a fraudulent winner.
How: one DB transaction takes a row lock (FOR UPDATE) on the cycle. It refuses if the winner is flagged, alerts are open, escrow is on hold or escrow is insufficient. unique(cycle_id) on payouts is the backstop. It then writes the receipt and the ledger entry in the same transaction.
Say: “It either all happens or none of it happens, and two simultaneous calls can’t both pay.”
7. Signed receipts and exports (crypto.ts, exports.ts, pdf.ts)
Problem: how does a member or auditor trust a receipt or report?
How: the data is turned into canonical JSON, signed with Ed25519, and checked against a public key at /api/ledger/public-key. Receipts are re-verified every time they load. /api/exports/verify checks a CSV from outside. The PDF is built by a custom writer with no library.
Say: “A signed file that is modified by even one character fails verification.”
8. Fraud quarantine and ruling override
Problem: a flagged bid shouldn’t win, but it shouldn’t be silently deleted either.
How: a flagged bid is excluded from winning. If it was the lowest, escrow goes to HELD_FOR_REVIEW. After the foreman’s ruling, the code can demote the old winner, promote the correct one and release the hold. Everything is logged.
Say: “Fraud handling is a reviewed workflow with a record, not a silent block.”
9. Race-safe auction close
Problem: every browser calls “close” at zero, so there could be several winners.
How: UPDATE ... SET status='BIDDING_CLOSED' WHERE status='BIDDING_OPEN' is an atomic claim, and only one caller gets a row back. The server clock decides expiry, and the client countdown is skew-corrected.
Say: “It’s idempotent: any number of callers, exactly one winner determination.”
10. Real-time engine (SSE)
Problem: members need to see bids and timer changes instantly.
How: sse-broker.ts is a publish/subscribe map with one channel per cycle. The stream route adds a 15-second heartbeat. The client hook re-syncs state after any reconnect, so no event is lost.
Say: “SSE is lighter than WebSockets for one-way updates.”
11. Secure KYC pipeline
Problem: fake documents, malware uploads, and leaked identity data.
How: it checks magic bytes, so a renamed .exe is rejected, and enforces 5 MB. Files are AES-256-GCM encrypted at rest. The government ID is encrypted as a field and shown masked. Every foreman view is logged.
Say: “Sensitive data is encrypted at rest and every access is on the ledger.”
12. Dispute system with frozen evidence (evidence.ts)
Problem: evidence can change after a complaint.
How: raising a dispute snapshots the bids, contributions, payout and fraud logs at that moment. It is allowed only inside the dispute window. Resolutions need mandatory comments.
Say: “The evidence is captured when the dispute is raised, so it can’t be altered later.”
13. Due-state machine and lazy sweep (schedule.ts, reminders.ts)
Problem: handling late payers without a separate job server.
How: dueState() returns NOT_DUE → REMINDER_DUE → DUE → LATE → DELINQUENT. runSweep sends notices and flags members. It runs from cron or lazily, throttled via maybeSweep.
Say: “The rules are a pure function with no DB access, so they’re easy to unit test.”
14. Concurrency-safe IDs and enrolment
How: advisory locks allocate M1001 and G001 without collisions, and the group row is locked so two people can’t take the last seat.
15. Live bell-curve UI (bell-chart.tsx)
How: it draws the group’s normal distribution and the flag line. Each bid is a pin on the curve, so members can see why a bid was flagged.
Say: “The UI shows the same threshold the server enforces.”
Top 5 if sir asks “what’s most novel?”
Z-score fraud detection (#1)
Hash-chained signed ledger with DB triggers (#3, #4)
Escrow and atomic payout contract (#5, #6)

(or)

A. Fraud and bidding
Z-score underbid detector. Z = (μ − bid) / σ, flagged above the group’s threshold (default 2.5). The mean and σ switch from baseline to live history after 4 clean bids. The threshold is configurable per group and takes effect immediately with no redeploy. (lib/fraud-detector.ts)
Collusion rules. There is a win-frequency check (FR22), an already-won check, and a trust-score engine:
Members get +5 for a clean bid and −25 for a bid that stays flagged.
The foreman gets +3 when all alerts are resolved by cycle end and −5 when alerts are still open.
Fraud quarantine and adjudication.
A flagged bid can’t win, and escrow goes to HELD_FOR_REVIEW.
The foreman’s ruling (fraud/override) can re-pick the rightful winner after close: it demotes the old winner, promotes the new one, and releases the hold.
Reverse-auction engine.
The lowest valid, non-flagged bid wins.
Self-outbid is blocked.
Bids are checked for KYC, enrolment and the chit-value cap.
Idempotent, race-safe cycle close.
Every client calls close when its countdown hits zero, but an atomic claim means only the first call does the work.
Members can only trigger it after the server clock says the window expired.
Foreman controls. Extend the timer live (+60s, +120s) or force-stop. The force-stop resolves the lowest valid bid, or logs FORCE_STOPPED_NO_VALID_BID when there is none.
B. Integrity and cryptography
Hash-chained, append-only audit ledger.
Each entry’s SHA-256 includes the previous hash and its sequence number, so entries can’t be re-ordered.
Each entry is also Ed25519-signed.
An advisory lock prevents the chain from forking.
Entries commit atomically with the business change they record.
Database-enforced immutability (triggers.sql). Triggers block UPDATE, DELETE and TRUNCATE on audit_logs. Payouts are immutable once issued.
Chain verifier. It re-checks links, hashes and signatures, and pinpoints the tampered entry.
Signed, verifiable outputs.
Receipts, CSV exports and PDF exports are Ed25519-signed with canonical JSON.
A public-key endpoint (/api/ledger/public-key) lets anyone verify.
An external auditor can POST a CSV to /api/exports/verify.
Receipts are re-verified every time they’re read.
Dependency-free PDF writer (pdf.ts). It builds valid A4 PDFs with an xref table, for the signed exports.
C. Money safety
Escrow released only via the payout contract.
A DB trigger rejects any release unless the transaction sets chainchit.payout = on.
Locked contributions can’t be deleted or changed.
settleCycle payout contract.
It locks the cycle row (FOR UPDATE) and has unique(cycle_id) as a backstop, so no double payout.
It blocks payout on a flagged winner, an open alert, or an escrow hold.
It verifies that escrow actually covers the payout.
It issues the signed receipt, writes the ledger entry, and auto-completes the group when all cycles settle.
Withdraw endpoint that always refuses and logs. The DB trigger blocks a raw SQL UPDATE as well.
Contribution flow. It rejects an amount mismatch, rejects duplicate payment, and handles a simulated declined payment with no lock and no ledger entry.
D. Security and compliance
KYC pipeline.
Magic-byte file sniffing, so a renamed .exe is rejected, plus a 5 MB limit.
Files are AES-256-GCM encrypted at rest.
A foreman viewing a document is itself written to the ledger.
Rejection requires a reason.
Field-level encryption and masking of the government ID.
Custom signed session tokens with an expiry, plus a proxy.ts gate that returns 401 for API calls and redirects pages to login.
Role-based access control with 3 roles. Public registration always creates a MEMBER. The foreman onboards other roles. The dashboard endpoint returns role-specific shapes, so a member can never receive foreman data.
Concurrency control.
A group row lock stops two enrolments taking the last seat.
Advisory locks keep the human-readable IDs (M1001, G001) collision-free.
E. Operations
Due-state machine (schedule.ts): NOT_DUE → REMINDER_DUE → DUE → LATE → DELINQUENT. The scheduling logic is pure, so it’s easy to unit test.
Reminder and delinquency sweep.
It can be triggered by cron (x-cron-secret) or lazily and throttled (maybeSweep).
It has an asOf time-travel parameter for testing, refused in production.
Maintenance windows that are blocked if they overlap a live bidding session, and bidding can’t open inside a window.
Notification system with dedupe keys, plus fraud alerts that go to the foreman and oversight viewers.
F. Disputes
Dispute workflow.
A dispute can be raised only inside the group’s dispute window.
Evidence is auto-snapshotted at that moment (bids, contributions, payout, fraud logs).
The foreman or oversight can resolve, reject or escalate, with mandatory comments.
G. Real-time and UI
SSE broker with one channel per cycle, a 15-second heartbeat, and a global singleton that survives hot-reload.
Live bell-curve chart showing the flag zone, with each bid as a pin on the curve.
Multi-room bidding with a navbar live-count, a lot board, bid slip, countdown and rolling-number animation.
Demo tooling. A simulate endpoint (dev only) with a reset mode and a scripted 3-bidder scenario.“Good morning sir. My project is ChainChit, a bidding and fraud-detection system for chit funds. In a chit fund, a group of people each pay a fixed amount every month, and one member takes the pot each cycle. The winner is decided by auction. The problem is that the auction is usually run by one foreman, so it can be opaque. Bids can be rigged, members can collude, and the records can be changed afterwards. ChainChit automates the auction and adds three protections: statistical fraud detection, protected escrow, and a tamper-proof audit log.”

[0:40 – Architecture, 30s]
“It’s a full-stack Next.js application with TypeScript. The database is PostgreSQL through Drizzle ORM. Live updates use Server-Sent Events, and all the cryptography uses Node’s built-in crypto module. There are three roles: Member, Foreman and Oversight Viewer. Access is enforced on every route.”

[1:10 – Onboarding and escrow, 40s] [Demo: register, KYC, approve]
“A member registers and uploads an Aadhaar or PAN document. The server checks the file’s actual bytes, so a renamed .exe is rejected, and it encrypts the file with AES-256 before storing it. The foreman reviews it, and that view is logged. Once approved, the member is enrolled in a group, and each cycle they pay their share. That money goes into escrow. A database trigger makes sure it can only be released inside the payout function, so nobody, including the foreman, can withdraw it.”

[1:50 – Auction, 45s] [Demo: open cycle, place bid, try second bid]
“The foreman opens a bidding window with a timer. It’s a reverse auction: members bid the amount they’re willing to accept, and the lowest valid bid wins. The difference is shared among the others as dividend. Every bid is validated for KYC, enrolment and the pool limit. A member who already holds the lowest bid can’t bid against themselves. Every bid appears instantly on all screens through Server-Sent Events.”

[2:35 – Fraud detection, 50s] [Demo: place the low ₹60,000 bid, show the bell curve]
“This is the core innovation. Every bid is scored with a Z-score: the group’s mean bid minus this bid, divided by the standard deviation. Honest bids cluster near the mean, so a bid more than 2.5 standard deviations below it is an outlier and is flagged. There are also rules for a member who has already won, and for a member who wins too often. A flagged bid can’t win. The foreman and oversight are alerted, and if it would have been the winner, the payout is held for review. The bell-curve chart shows members exactly where the flag line is.”

[3:25 – Closing and payout, 40s] [Demo: stop cycle, show receipt]
“When the timer ends, every browser calls close, but an atomic database update means only one call determines the winner. Settlement runs in one transaction. It locks the cycle, refuses if the winner is flagged or alerts are unresolved, checks escrow covers the payout, releases the money, and issues a receipt signed with Ed25519. A second payout for the same cycle is impossible.”

[4:05 – Audit ledger, 45s] [Demo: audit page, Verify Integrity]
“Every action goes into a hash-chained ledger. Each entry contains the SHA-256 hash of the previous entry and is digitally signed. Changing any old record breaks every hash after it. On top of that, database triggers reject any update or delete on the log. The oversight role can run ‘Verify Integrity’ and, if anything was tampered with, the system points to the exact entry. Receipts and exports are signed too, so an outside auditor can verify them with the public key.”

[4:50 – Close, 10s]
“To summarise: fair bidding, mathematically detected fraud, locked escrow and a verifiable audit trail. Thank you, sir. I’m happy to take questions.”

Quick Q&A backup
Why no blockchain? A signed, append-only hash chain gives the same tamper-evidence without the cost, and the database triggers add a second layer.
Why 2.5σ? It’s about the 99th percentile of a normal distribution, and it’s configurable per group.
Limitations? Payments are simulated, and the SSE broker is in-memory, so scaling to several servers would need Redis. Bid recording isn’t one transaction, but settlement is.
Signed, independently verifiable receipts and exports (#7)
Fraud quarantine with ruling override (#8)
