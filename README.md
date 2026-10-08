# School System Daily Learning

A mobile-friendly daily learning briefing for Singapore school management system developers.

## Publish on GitHub Pages

1. Create a public repository named `school-system-daily-learning` under `WingFei`. Initialise it with a README so that a `main` branch exists.
2. Upload the contents of this folder to the repository root, rather than placing the folder itself inside the repository.
3. Open **Settings → Pages**.
4. Under **Build and deployment**, select **Deploy from a branch**.
5. Select **main** and **/(root)**, then save.
6. Use the live URL shown by GitHub Pages. The expected address after successful publishing is `https://wingfei.github.io/school-system-daily-learning/`.

Official instructions: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site

## Files

- `index.html`: latest briefing.
- `briefing-2026-10-06.html`: fixed dated edition.
- `archive.html`: edition index.
- `.nojekyll`: serves the plain static HTML without Jekyll processing.

No build, database, API key or external scripts are needed. Print / Save as PDF opens the browser print dialogue; it does not download a pre-generated PDF.

## Add a daily edition

1. Research and verify source facts and dates. Separate confirmed facts from recommendations.
2. Preserve the old dated edition.
3. Create `briefing-YYYY-MM-DD.html` with the new edition.
4. Copy the new edition to `index.html`.
5. Add its date, title and relative link to `archive.html`, newest first.
6. Commit the files. Once Pages is configured, it publishes from the configured branch.

No automatic content-generation schedule has been configured. Publishing a page does not generate tomorrow's briefing.

## School Management System development

- [Phase 1 — ASP.NET Core, Blazor Interactive Server, EF Core, MariaDB and DevExpress Blazor](guides/new-school-management-system-phase-1.md): architecture, setup, service and persistence examples, migrations and verification checklist.
