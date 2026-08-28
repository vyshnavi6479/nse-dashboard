# NSE Hourly Dashboard

A shareable webpage version of the Excel logger — same logic (correct
NSE API, one row per hour, only updates on real volume/delivery
change), but as a live link instead of a file, with a date picker to
browse previous days.

## How it works

- A background thread checks NSE every 5 minutes for RELIANCE, TCS,
  INFY, HDFCBANK, SBIN — regardless of how many people are viewing
  the page.
- One row per hour per stock. A row only updates when Total Traded
  Volume, Quantity Traded, or Deliverable Quantity actually changes.
- Data is stored in a local file (`nse_data.db`), so past days stay
  browsable via the date dropdown/picker at the top of the page —
  they don't disappear when the app restarts.
- The page auto-refreshes every 30 seconds, but ONLY while viewing
  today — picking a past date shows a fixed snapshot that doesn't
  need to keep re-checking, since history doesn't change.

## Run it locally first

```bash
pip install -r requirements.txt --break-system-packages
python app.py
```
Open http://localhost:5000 — confirm real data appears before
deploying anywhere.

## Deploying (free tier)

1. Create a free account at render.com
2. Push this folder to a GitHub repo (or use Render's manual deploy)
3. Render -> New -> Web Service -> connect your repo
4. Leave build/start commands as default (Render reads `Procfile`
   and `requirements.txt` automatically)
5. Deploy. You'll get a URL like `https://your-app.onrender.com` —
   that's the link to share.

## IMPORTANT — what "free tier" actually means here (read before relying on this)

You chose the free tier, so here's exactly what that trades off:

**The app sleeps after ~15 minutes with no visitors.** The background
5-minute checker ONLY runs while the app is awake — so if nobody
opens the link for a while (e.g. overnight, or during a quiet part of
the day), you'll get a **gap** in that period's hourly data. When
someone next opens the link, the app wakes up (takes ~30-60 seconds
the first time) and resumes checking from that point forward — but
the gap itself isn't backfilled, since NSE doesn't offer that data
retroactively either.

In practice: the more often someone actually opens the link during
the day, the more complete the data will be, since each visit keeps
it awake a while longer. If you need guaranteed, gap-free continuous
logging regardless of visits, that's what the ~$7/month always-on
tier is for — but you've decided the free tier's trade-off is
acceptable for now, and this is built accordingly.

## Second known risk (separate from the sleep issue)

Everything has only been tested from a home network so far. Hosting
providers use datacenter IP ranges, and we don't know for certain
whether NSE treats those differently until this is actually deployed
and tested live. If it works locally but the hosted version gets
blocked, that's the likely reason.

## Why only 1 worker in the Procfile

Each gunicorn worker would start its own background poller thread —
multiple workers means multiple simultaneous pollers hitting NSE
independently. Don't increase worker count without also restructuring
how polling works.
