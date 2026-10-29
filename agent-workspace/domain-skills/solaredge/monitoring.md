# SolarEdge Monitoring portal — site data, alerts, production history

`monitoring.solaredge.com` is a React/MUI micro-frontend SPA ("one" portal). The classic
`/solaredge-web/p/site/<id>/#/layout` URLs now redirect to the new dashboard, and the old
`/solaredge-apigw/api/...` endpoints return **410** with a marketing link — don't bother.

**Everything the dashboard shows is available as session-cookie JSON under `/services/`.**
Log in once in the browser, then `fetch()` with `credentials:"include"` via `js(...)`.
No CSRF token needed for GETs (verified Oct 2026).

## URL patterns

- Login: `https://monitoring.solaredge.com/` → `/mfe/auth/` (Cloudflare challenge in front; let the human log in).
- Site dashboard: `https://monitoring.solaredge.com/one#/residential/dashboard?siteId=<siteId>`
- Site list: `/cni#/site-list`
- Other hash routes (`layout`, `alerts`, `admin`, `charts`) silently bounce back to `dashboard`
  unless the account has those tabs. Check `/services/installer-infra/sites/<id>/tabs` first —
  a homeowner login returns `["DASHBOARD"]` only. Error codes and the physical layout are NOT
  exposed to homeowner logins; they only exist on the inverter's local page (`http://172.16.0.1`
  over the inverter's `SEDG-<serial>` Wi-Fi AP) or in SetApp.

## Private API (same-origin GETs, all return JSON)

```python
import json
def se(path):
    return js("""(async()=>{const r=await fetch(%s,{credentials:"include"});return await r.text()})()""" % json.dumps(path))

SITE = 715722   # from the dashboard URL
se(f"/services/dashboard/alerts/sites/{SITE}")                 # {"totalAlertsCount":2,"topAlerts":[{alertType,creationDate,impact,deviceName}]}
se(f"/services/dashboard/v2/site-details/{SITE}/equipment")    # inverter model + serials, optimizer model + count
se(f"/services/dashboard/site-details/{SITE}/communication")   # {"connectivityStatus":"ONLINE"}
se(f"/services/dashboard/site-details/{SITE}/details")         # address, installer accountName, peakPower, cellularPlanExpirationDate
se(f"/services/dashboard/live-power/sites/{SITE}")             # {"currentAcPower":0.0,"maxAcPower":6000.0}
se(f"/services/dashboard/comparative-energy/{SITE}/monthly")   # per-month Wh, every year since install → fastest way to find WHEN a system died
se(f"/services/dashboard/comparative-energy/{SITE}/yearly")
se(f"/services/dashboard/energy/sites/{SITE}?chart-time-unit=days&start-date=2025-04-01&end-date=2025-04-30&measurement-types=production")
se(f"/services/dashboard/power/sites/{SITE}?chart-time-unit=quarter-hours&start-date=2025-04-09&end-date=2025-04-10&measurement-types=production")
se(f"/services/dashboard/inverters/power/sites/{SITE}?start-date=2025-04-08&end-date=2025-04-11&normalized=false&chart-time-unit=hours&page-number=0")
se("/services/cni/ui-api/user-info")                            # who is logged in; accountId null = homeowner
se("/services/user-management/user")                            # trustedInstaller flag
```

- Energy values are **Wh**, power values **W**. Dates are site-local with offset.
- `chart-time-unit` accepts `quarter-hours`, `hours`, `days`, `months`.
- In the inverter power series, `null` = no telemetry received for that hour (inverter dropped
  off), `0.0` = inverter reported but produced nothing. The null→0.0 transition pins the
  failure hour.
- Alert types seen: `INVERTER_BELOW_THRESHOLD_GENERAL`, `INVERTER_BELOW_THRESHOLD_LOW_VOLTAGE`.
  Clicking an alert card in the dashboard does nothing for homeowner logins.

## Discovering more endpoints

`performance.getEntriesByType("resource")` filtered to `xmlhttprequest|fetch` lists every call
the dashboard made — that's how the list above was found. Repeat after opening any new panel.

## Traps

- The dashboard is taller than the viewport and scrolls inside its own container; `scroll()`
  on the window does nothing visible. Use the API instead of scrolling for data.
- `Emulation.setDeviceMetricsOverride` through the harness `cdp()` helper fails with
  "Message may have string 'sessionId' property" — don't try to widen the viewport that way.
- The left rail is a single "Site Overview" link for homeowner accounts; there is no hidden menu.
- Third-party beacons (harksys pulse, pendo) show up in the resource list — ignore them.
