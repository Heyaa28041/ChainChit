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
Signed, independently verifiable receipts and exports (#7)
Fraud quarantine with ruling override (#8)
