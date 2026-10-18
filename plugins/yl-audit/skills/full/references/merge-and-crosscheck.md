# Merge and cross-check

## Evidence tags
Tag every statement and finding with where it was seen: **Live** (browser), **Code** (source), **Config** (settings in files), **Docs** (stated, not checked), **User-run** (a check the user ran and pasted). A finding seen in two or more tags is **corroborated**.

## Cross-check rules
- **Live and Code agree:** mark corroborated, confidence raised.
- **Live only** (for example data over-sent): find the cause in code if code exists; otherwise "plausible" with a "To verify: share the code for X".
- **Code only** (for example a route with no gate): check whether the browser traffic shows it; if the route is never called from the app, say so. Do not call routes by hand unless the code proves they are pure reads.
- **Contradictions** (docs say gated, code or traffic say not; code says protected, browser reaches it): list them in a "Contradictions" table and rate on the weaker evidence of safety.
- **Supabase findings** tie back to the endpoint map: each browser-direct table, function or bucket gets its policy type and the anon result.
- **Docs-only claims** are never treated as confirmed.

## Merged outputs
1. **Architecture overview:** one diagram, with each connection tagged Live, Code or Docs.
2. **Authentication mechanism:** from code, with the browser's view (what token is held, where, what the app sends).
3. **Endpoint map:** union of code and traffic, bucketed by auth mechanism, ranked by data risk, with a column for evidence tags. Add rows the other source missed.
4. **Supabase section** (if run): level, delivery checks, anon exposure.
5. **Dependencies and pinning**, **secrets and history**, **scanner results**.
6. **Findings:** one table with a **Category** column (the eight categories in `../../code/references/report-template.md`), unique IDs with the agreed prefix, High to Low, severity per `../../code/references/severity-guide.md`. Stage trade-offs are Low and labelled as pragmatic suggestions (see `rules.md`), with a note above the table.
7. **Recommendations** ordered by work with effort tags; open with real good practice.
7a. **Architecture suggestions:** the section described in `../../code/references/report-template.md`, drawing on the Supabase level, the CI level and the observability level, scaled to the team and product stage.
8. **What is working well**.
9. **To verify (one consolidated list).** Deduplicate across audits. Group by who acts: project owner or client dashboard, the user running SQL or commands, the client (settings or access), a code drop. Each item: **To verify:** do this (where). **Expect:** result. **If not:** meaning and next step. Order by value (what would change a rating first).
10. **Coverage and limits:** audits run and skipped with reasons, tools run and skipped, what the user must still check, the browser audit trail, and an explicit statement that live settings were not checked unless results were supplied.

## Deduplication and rating
- One finding per cause. Merge duplicates from different audits and list every evidence tag on the merged row.
- Rate on the strongest confirmed evidence, never on a single unverified claim.
- Re-read every High and Medium in the source before it goes in the table.
- If a reviewer or scanner claim is wrong, correct or drop it and say so in the coverage notes.
- Add a short count of findings by category and by severity to the overview, so patterns are visible at a glance.
