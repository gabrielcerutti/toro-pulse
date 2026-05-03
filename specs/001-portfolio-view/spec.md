# Specify: Public Portfolio View

**Spec ID:** 001-portfolio-view
**Status:** Draft
**Author:** Gabriel Cerutti
**Date:** 2026-04-30

---

## 1. Problem

Retail and prospective copy-traders on eToro inspect other users' portfolios to evaluate strategy, risk, and consistency before they copy or compare. eToro's native UI surfaces this information, but third-party views (e.g., bullaware.com/etoro/gabipins) attract sustained traffic by presenting the same data with sharper analytics, faster navigation, and a public-link sharing model.

We want a public, read-only portfolio view for any eToro user that:
- Loads faster than eToro's own profile pages.
- Surfaces analytics not available natively (concentration, sector exposure, drawdown, normalized returns).
- Is link-shareable without login.
- Lays the groundwork for authenticated features (watchlists, alerts, copy comparison) without re-architecting.

## 2. Why Now

This spec exists primarily to give the project's tech stack a real surface to harden against. It is intentionally narrow. Once the stack is validated end-to-end against this spec, broader product specs follow.

## 3. Scope

### In scope (v1)

- **Search by username.** Type or paste any public eToro username; resolve and route to that user's portfolio page.
- **Profile summary card.** Display name, avatar, country, copier count, year-to-date gain, all-time gain, risk score, win ratio, last active date.
- **Current holdings table.** Each row: instrument symbol, instrument name, instrument type (stock, ETF, crypto, currency, commodity, index), portfolio weight %, average open price, current price, P&L %, direction (long/short).
  - Sortable by weight, P&L %, name.
  - Filterable by instrument type.
- **Performance chart.** Historical equity curve with selectable ranges: 1M, 6M, 1Y, All. Hover reveals point-in-time equity and daily change.
- **Top 10 holdings panel.** Visual breakdown (donut or stacked bar) of top 10 by weight, with the rest collapsed to "Other."
- **Sector and asset-class breakdown.** Two pie/donut charts: by sector (for equities), by asset class (across the full portfolio).
- **Public, link-shareable URL.** `/u/{username}` resolves directly. No auth wall.
- **Cached eToro responses.** Stale-while-revalidate. 15-minute TTL on portfolio composition, 5-minute on prices, 1-hour on profile metadata.

### Out of scope (v1)

- Trading actions (open/close, copy, watchlist add).
- Authenticated features (saved searches, alerts, comparison) — auth is scaffolded but not surfaced.
- Mobile-native apps (responsive web only).
- Internationalization (English-only v1).
- Historical holdings (v1 shows current snapshot only).
- Risk decomposition models (Sharpe, max drawdown beyond what eToro exposes natively).
- User-to-user comparison views.
- Comments, social, or any UGC.

## 4. Users and Use Cases

**Primary user:** a retail trader evaluating an eToro user before copying.
- *Job:* "I want to check this person's holdings and performance trend without logging in to eToro."

**Secondary user:** an eToro user sharing their own portfolio externally.
- *Job:* "I want a clean public link to my portfolio that I can put in my LinkedIn profile or X bio."

**Tertiary user:** a researcher or journalist sampling eToro user portfolios.
- *Job:* "I want to look up several users quickly without a login flow."

## 5. User Stories

1. As a visitor, I land on the home page, type an eToro username into a search box, and arrive at that user's portfolio page within 2 seconds.
2. As a visitor, I see the profile summary at the top, a performance chart, and a holdings table without scrolling more than once on a 14" laptop.
3. As a visitor, I sort the holdings table by P&L % and the table re-orders without a full page reload.
4. As a visitor, I switch the performance chart from "1M" to "1Y" and the chart re-renders in under 500ms (warm cache) or shows a skeleton state otherwise.
5. As a visitor, I share the URL with a colleague; they open it and see the same data without logging in.
6. As a visitor on a screen reader, I navigate the holdings table with arrow keys and hear instrument name, weight, and P&L for each row.
7. As an eToro power user, I bookmark `/u/myname` and use it as my public profile link.

## 6. Acceptance Criteria

The spec is satisfied when all of the following hold:

| ID  | Criterion                                                                                                  | How verified                                |
|-----|------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| A1  | Searching a valid public eToro username navigates to `/u/{username}` and renders portfolio data.           | Playwright E2E.                             |
| A2  | Searching an unknown username shows a clear "no such public user" empty state, not a 500.                  | Playwright E2E.                             |
| A3  | Holdings table is sortable by weight, P&L%, name, and filterable by instrument type, all client-side.      | Playwright + axe.                           |
| A4  | Performance chart range selector switches data without a hard navigation.                                  | Playwright.                                 |
| A5  | Top 10 holdings donut, sector donut, asset-class donut all render with legible labels at 1366×768 desktop. | Visual regression (Playwright screenshot).  |
| A6  | All public routes score Lighthouse Performance ≥ 90, Accessibility = 100.                                  | CI Lighthouse run.                          |
| A7  | P95 server response for `/u/{username}` portfolio fetch is < 1500ms cold, < 300ms warm.                    | Synthetic monitor.                          |
| A8  | Holdings table is fully operable via keyboard and announces sort state to screen readers.                  | axe + manual NVDA pass.                     |
| A9  | Empty / loading / error states exist for every data panel.                                                 | Code review + Storybook (if added).         |
| A10 | OpenTelemetry traces exist for every API endpoint backing the page.                                        | Tempo / Honeycomb dashboard check.          |

## 7. Open Questions (for /clarify)

1. Does eToro's public API permit unauthenticated portfolio reads, or do we need an authenticated service account? (Affects rate-limit strategy and ToS posture.)
2. What is the legal posture on caching and republishing eToro data? (May require an attribution footer and a delete-on-request mechanism.)
3. Do we want SEO-friendly server rendering for `/u/{username}` (yes likely, but confirm — affects Server Component vs Client Component split)?
4. Currency display: USD only, or follow the user's eToro account currency? (Affects FX layer.)
5. What's the upper bound for holdings count we render before we paginate or virtualize? (Affects table strategy.)

## 8. Non-Goals (explicit)

- We are not building a trading platform.
- We are not building a copy-trading recommender.
- We are not collecting or storing any personal data about *visitors* in v1 beyond standard request logs.
- We are not negotiating a commercial data agreement with eToro for v1; we work within their public terms.

## 9. Risks

- **eToro API access changes or rate-limits.** Mitigation: cache aggressively, build a circuit breaker, monitor upstream.
- **Legal/ToS challenge.** Mitigation: attribution, delete-on-request, no scraping beyond documented public surfaces.
- **Cold cache latency.** Mitigation: pre-warm popular usernames via a background job once we have traffic data.
- **Currency/FX correctness.** Mitigation: clearly label currency at every numeric display; never silently convert.

## 10. Success Metrics (post-launch, informational only — not gating)

- Time-to-first-portfolio-view from landing page < 30 seconds (median).
- Bounce rate on `/u/{username}` < 50%.
- Cache hit rate on portfolio fetch > 70% within first month.
- Zero P0 incidents related to eToro upstream coupling in first 30 days.

---

## Appendix: Reference UI

The bullaware.com/etoro/gabipins layout is the closest existing reference. We are not copying it; we are matching the *information density* and *load profile* of that page as a baseline for the v1 surface.
