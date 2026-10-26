---
layout: default
title: "Kronova asset-intel-orchestration-engine: an RLS policy with USING (true) let the anon role read every profile"
description: "Notes on GHSA-8h52-3cfj-cmrw in Kronova-Intelligent-Systems/asset-intel-orchestration-engine. A permissive SELECT policy for anon on public.profiles had a true predicate, and Postgres ORs permissive policies, so a narrower sibling policy did not limit it."
date: 2026-10-27
---

# Kronova asset-intel-orchestration-engine: an RLS policy with USING (true) let the anon role read every profile

**Advisory:** [GHSA-8h52-3cfj-cmrw](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine/security/advisories/GHSA-8h52-3cfj-cmrw) (published 2026-09-17)
**Affected:** `asset-intel-orchestration-engine`, all versions per the advisory, no patched version listed
**Weakness:** CWE-1188, CVSS 3.1 `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` (High, 7.5)

[asset-intel-orchestration-engine](https://github.com/Kronova-Intelligent-Systems/asset-intel-orchestration-engine) is a Next.js application on Supabase. As in any Supabase app, the browser client is built with the anonymous key (`NEXT_PUBLIC_SUPABASE_ANON_KEY` in `lib/supabase/client.ts`), which Next.js inlines into the client bundle by design. That key is safe to publish only because row level security decides what it can read.

## Root cause

`public.profiles` has RLS enabled. The migration `supabase/migrations/20240327_add_profile_public_id.sql` creates this policy:

```sql
CREATE POLICY "Allow public access to profiles"
ON public.profiles
FOR SELECT
TO anon, authenticated
USING (true);
```

`USING (true)` admits every row, and `TO anon` applies it to callers with no session. A narrower policy created the same day in `20240327_add_public_id_to_profiles.sql` uses `USING (public_access = true)`, which looks like the intended rule. It does not limit the first one, because Postgres combines permissive policies with OR. Walking all 108 migrations in order, no later one drops the broad policy.

The table defines `email`, `full_name`, `username`, `company`, `website`, `waddress` and `xhandle`, and `app/actions/profile.ts` fills `email` from the account's auth email. So an unauthenticated caller could read every registered user's email and wallet address through PostgREST. The policy is `FOR SELECT`, so there is no write path.

My reproduction ran only against a local Postgres container with the policies applied from the migrations. As a control, dropping that one policy made the same `anon` read return zero rows, which shows RLS was active and this policy was what opened the table. I did not test or look for any deployed instance.

## Fix

The advisory lists no patched version. The suggested change is to drop the broad policy and keep an `anon` read only where the owner opted in (`USING (public_access = true)`). Two related points from the report: `public_access` defaults to true, so existing rows stay readable unless the default is flipped or the column backfilled, and `email` is safer outside any table that has an anonymous read path, for example behind a view that exposes only display fields. A policy can also be changed directly on a deployed database, so the migrations in the repository do not show the state of any deployment.

## What I took from it

- In Postgres RLS, a permissive policy can only widen access. Adding a narrow policy next to a broad one changes nothing. Look for the broadest permissive policy per role and command.
- For idempotent migrations that drop and recreate policies, the only reliable way to know the final policy set is to replay them in order.
- `TO anon` with `USING (true)` is worth a lint rule on any table that holds personal data.

Thanks to the maintainers for reviewing the report and publishing the advisory.
