# Copilot instructions

## Build, test, and run

- This repository is a static site and has no package manifest, build command, lint command, or automated test runner.
- To run it locally, configure Supabase as described in [SUPABASE_SETUP.md](../SUPABASE_SETUP.md), then run `python3 -m http.server 8000` from the repository root and open `http://localhost:8000`. Do not open the site with `file://`.
- There is currently no automated single-test command. Validate UI changes in the browser; database-backed flows require a configured Supabase project.

## Architecture

- The app is a single-page site in `index.html`: markup, styles, and application logic are kept together. Hash routes switch between the public catalog and the MTO management view.
- Supabase JS v2 is loaded from a CDN. `supabase-config.js` provides `window.SUPABASE_CONFIG`; the browser client reads and writes `public.inventory_items`, with database column mapping handled by `fromDatabase` and `toDatabase` in `index.html`.
- A database row represents a top-level inventory item. Its variants are stored as a nested JSONB array and are edited through the parent item. Catalog search and filters include both parent items and variants.
- Photos are uploaded to the public `inventory-photos` Supabase Storage bucket; records store the resulting public URLs. Supabase Auth provides employee sign-in. Authorization must remain enforced by Supabase grants and RLS policies, not just by hiding management controls in the UI; see [SUPABASE_SETUP.md](../SUPABASE_SETUP.md).
- Older browser data under the `kulisy_inventory_v1` localStorage key can be imported only when the remote table is empty. Import also uploads legacy data-URL images; it does not delete the original local data.

## Repository-specific conventions

- Keep user-facing text and inventory statuses in Russian. The supported status labels are `В наличии`, `Занят`, and `На ремонте`.
- The page uses inline event attributes as well as JavaScript event listeners. Functions called from inline attributes must remain globally accessible; preserve the existing interaction pattern unless deliberately refactoring those call sites.
- When generating HTML with template strings, escape dynamic text and attribute values with the existing `esc` helper. Prefer `textContent` for plain text.
- Preserve the database schema and JSON property mapping documented in `SUPABASE_SETUP.md` when changing inventory fields. Keep Supabase credentials in the separate browser config; only a publishable/legacy anon key belongs in client code, never a `service_role` key.
- Keep asynchronous database and upload failures visible to users through the existing connection status or error alerts rather than silently treating them as success.
