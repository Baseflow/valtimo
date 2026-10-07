# 13.49.0

Release date: 07-10-2026

---

## New Features

### New feature title

New feature explanation.

---

## Enhancements

### New enhancement title

New enhancement explanation.

---

## Bugfixes

| Area | Fix |
|------|-----|
| IKO | A widget or search result list that could not retrieve its data now says so and offers a retry |
| Building blocks | A final building block can be deployed to an environment that does not allow drafts, such as production |
| Form flows | A form flow that has been used can now be deleted from a draft case definition or building block, as can the draft case definition itself; its form flow instances are deleted along with it |
| Cases | A form opened in the side panel of a case stays open and keeps its contents when switching between the case's tabs |
| Cases | A start form configured to open in the side panel now opens there on every case tab, including tabs without a task list, instead of opening in a modal |
| Platform | Memory no longer grows when browsers disconnect from live updates (SSE). Disconnected subscriptions are dropped after 2 minutes instead of 3 hours, their event backlog is capped, connected clients get a heartbeat, and events are sent from a background thread so a slow or vanished browser can no longer stall case or task processing. The new `valtimo.sse.*` settings control the connection timeout, heartbeat interval, grace period and backlog size |
| Platform | A browser that disconnects mid-request (broken pipe) is no longer logged as an error |
