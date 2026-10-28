# ChainChit
Building the frontend for the new features now. First I'll look at the existing UI so the new pages match it.

Frontend in progress: the navbar with notifications, the KYC upload page and the member dashboard (pay into escrow, history, raise disputes) are done and typecheck. The foreman and oversight dashboards, groups, disputes and receipts pages, and the foreman desk patches are still to do.

Now the disputes pages, the receipt page, and the patches to the foreman desk and audit page.

All the frontend pages are written and the production build passes. I'm restarting the database and app now to smoke-test them in a browser before zipping.

All pages render without console errors. Let me glance at one screenshot, then package.

Looks right. Packaging now.

The frontend is done and zipped together with the backend in `chainchit-full.zip`. The type check and production build pass. A browser run against the production build loaded the foreman and oversight pages with no console errors. I only opened them. I did not click through the upload, dispute-ruling, or receipt flows, and the member view was not loaded.

What's new in the UI:
- **Member:** a KYC upload page, a notification bell with unread count, and a dashboard with payment (including a simulated-decline option), bids, payouts, disputes and receipt links.
- **Foreman:** a dashboard with a maintenance form and a sweep button, and a group page with schedule, members, ledger and settings. The foreman desk now shows submitted KYC documents and has the new group-creation fields.
- **Foreman, after payout:** settling a cycle opens its signed receipt.
- **Oversight:** an integrity banner, collusion review, and signed CSV/PDF exports on the dashboard and the audit page.
- **Shared:** dispute list and detail pages with evidence attached automatically and mandatory ruling comments. Receipt pages re-check the signature every time they open and can be printed.

Seeded logins are `foreman@chainchit.io` and `oversight@chainchit.io`, password `Password@123`.
Correction to my earlier answer: `SESSION_SECRET` isn't used by this app. The session and encryption keys are generated automatically and saved in a `.data/` folder the first time it runs. The only value you have to set is `DATABASE_URL`.

**Create the `.env.local` file**

In the `chainchit-main` folder, create a file named `.env.local` containing one line:

```
DATABASE_URL=postgresql://postgres:YOURPASSWORD@localhost:5432/chainchit
```

Replace the parts as follows:
- `postgres` is your Postgres username. `postgres` is the default.
- `YOURPASSWORD` is the password you chose when you installed Postgres.
- `chainchit` is the database name, which you create first.

**Create the database**

Pick one:
- **Command line:** `createdb -U postgres chainchit`
- **psql:** run `psql -U postgres`, then `CREATE DATABASE chainchit;`
- **pgAdmin:** right-click Databases, choose Create, then Database, and name it `chainchit`.

**Don't have Postgres?** The quickest route is Docker:
```bash
docker run -d --name chainchit-db -e POSTGRES_PASSWORD=pass -e POSTGRES_DB=chainchit -p 5432:5432 postgres:16
```
Then use `DATABASE_URL=postgresql://postgres:pass@localhost:5432/chainchit`. The database is already created in this case, so skip the createdb step.

**Then run**
```bash
npm run db:reset
npm run dev
```

If your password has special characters like `@` or `#`, URL-encode them (`@` becomes `%40`). On Windows, create the file in Notepad and save it as `.env.local` with "All files" selected, so it doesn't end up as `.env.local.txt`.
