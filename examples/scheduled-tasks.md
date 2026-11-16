# Scheduling the autopilot

Create one scheduled task per site, three runs a week. Stagger the sites so two builds never run at once.

| Site | Cron | Meaning |
| --- | --- | --- |
| site-one | `30 7 * * 1,3,5` | Mon, Wed, Fri 07:30 |
| site-two | `30 9 * * 2,4,6` | Tue, Thu, Sat 09:30 |
| site-three | `5 8 * * 1,3,5` | Mon, Wed, Fri 08:05 |

Each task's prompt is short, because the method lives in the skill and the site file:

```
You are running the SEO and AI-search autopilot for site = site-one. Fresh session, no prior context.
Read and follow, in order:
1. skills/seo/SKILL.md
2. skills/seo/sites/site-one.md
Run with no mode argument. Apply fixes directly within the site file's guardrails. Follow the skill for the
run stamp, the log, the run record and closing every tab you opened, and end with a short report of what
you changed and anything that needs the owner.
```

Add the dead-man pings from [ai-agent-fleet-monitor](https://github.com/waltermitas/ai-agent-fleet-monitor)
as the first and last actions if you want to know when a run silently dies.
