# PQRS System — Transportes RG

This repo showcases my experience developing a complaints-and-requests management system I built for a Colombian logistics
company. While i cannot share the code, i can and will share my developement process. 
"PQRS" stands for Peticiones, Quejas, Reclamos y Sugerencias — the formal channel companies here use to track customer requests and complaints.

The source code is in a private repo since it's an internal tool for a real company, so this README is a walkthrough of what I built and the decisions
behind it rather than the code itself. Happy to go through it in an interview.

## What it does

- Each employee logs in with their own account
- Register a PQRS with customer data, type, channel and file attachments
- Assign it: owner, area, priority and a response deadline
- Move it through states (New → Under review → In progress → … → Closed / Overdue)
- Full audit trail — every change records who, when and what changed
- In-app notifications (new ticket, assignment, resolved/closed)
- Reports: tickets this month, open, overdue, average response time, plus
  breakdowns by type / status / owner
- Filters by year and month, type, status and owner

## Tech stack

- Frontend: JavaScript + Vite (no framework since its a small project and i wanted to do understand the DOM better)
- Database / Auth / Storage: Supabase (PostgreSQL) since i wanted a cloud architecture. This system is meant for being used by
  less than 10 people so i used Supabase free plan instead of buying a server and complicating everything more.
- Hosting: Vercel, auto-deploying from GitHub so i could easily deploy and keep the page alive (also for free)
- Automated backups: I used GitHub Actions to run a scheduled `pg_dump`

## Some decisions I'm happy with

- Audit history at the database level. The client told me the change history was the single most important feature.
- Instead of logging changes from the frontend (which can be skipped or bypassed), I used PostgreSQL triggers, so any
change to a ticket gets recorded no matter where it comes from.

- Row Level Security. Nothing can be read or written without a valid session, and notifications are scoped so each user only ever sees their own.

- A generated column for claim math. For claims, "company loss" is the claim value minus what each party covers. I made it a Postgres generated column so it stays correct automatically and can't drift away from the inputs.

## Problems I ran into (where I learned the most)

- Backups were failing with a version mismatch. My scheduled backup kept giving me errors with
`pg_dump: server version mismatch`. Supabase had moved to PostgreSQL 17 but the GitHub runner shipped pg_dump 16.
I installed the v17 client… and it still used 16, because the runner's PATH pointed at the old binary. The fix was to
call pg_dump by its full path (`/usr/lib/postgresql/17/bin/pg_dump`). Took a few iterations to actually land.

- The app breaking when switching browser tabs. Coming back to the tab threw `invalid input syntax for type uuid: "undefined"`.
Supabase fires an auth event on token refresh, and my code was re-rendering the whole app but dropping the ID
of the record being viewed. I fixed it by holding the current view and its params in state and only re-rendering on a real sign-in/out,
not on every token refresh.

- One long word wrecking the layout. A test comment with a few hundred characters and no spaces pushed an entire panel off screen.
This one looks stupid, but i was just seeing my panel disappear becasuse that one long comment was at the bottom of the page were i wasnt looking and i didnt even consider it. Classic CSS grid `min-width` trap — fixed with `min-width: 0` on the grid children plus `overflow-wrap`.

## What I'd do next

- Email alerts for tickets about to hit their deadline
- Exporting reports to Excel
- A custom domain

## Context

This was my first time using Supabase and deploying something end to end. When i was asked to do this i genuinely thought there was
no way i was going to be able to do it, but after learning about supabase and vercel it was fairly easy with all the online information 
(also,  probably thanks to this project being a system that only a few people will use and not a hundred-thousand user project).
