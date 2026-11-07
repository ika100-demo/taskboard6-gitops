---
spec_id: 001-persistent-tasks-and-task-ui
title: Persistent tasks and task UI
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and task UI

## Problem

The taskboard6 system can create and list tasks only through its API, and the API keeps them in memory: every restart or redeploy loses every task, so nobody can rely on the board. Using it also requires sending HTTP requests by hand, because the web app is still the starter page. Now that both components run in dev, tasks should survive restarts and people should be able to see and add them in the browser.

## Stories

As a team member, I want the tasks I record to still be there after the system restarts or is redeployed, so that I can rely on the board as the record of our work.
**Repos:** taskboard6-api, taskboard6-gitops

As a team member, I want to open the web app and see all tasks with their due dates, so that I know what is to be done without using the API by hand.
**Repos:** taskboard6-web, taskboard6-api, taskboard6-gitops

As a team member, I want to create a task with a title and an optional due date from the web app, so that I can record work in the browser.
**Repos:** taskboard6-web, taskboard6-api

## Acceptance criteria

- **AC-001.1** Given tasks A, B and C were created through `POST /tasks` in that order, when the API is restarted or redeployed and a client then sends `GET /tasks`, then the response is `200` with exactly those three tasks in creation order, each with the same `id`, `title`, `due_date` and `created_at` returned when it was created.
- **AC-001.2** Given the API cannot reach its database, when a client sends `POST /tasks` with a valid body or `GET /tasks`, then the response is `503` with a JSON error body, no task is created, and the API's readiness endpoint answers with a non-`200` status until the database is reachable again.
- **AC-001.3** Given tasks exist, when a user opens the web app's root URL, then the page lists every task returned by `GET /tasks` in the same order, showing for each its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.4** Given no task exists, when a user opens the web app's root URL, then the page shows the text "No tasks yet" and the create form.
- **AC-001.5** Given the web app's root page is open, when the user enters the title "Buy milk" and the due date 2026-12-31 and submits the form, then within 5 seconds and without a manual page reload the task appears at the end of the list with that title and due date, the form fields are cleared, and `GET /tasks` on the API returns the task.
- **AC-001.6** Given the web app's root page is open, when the user submits the form with no due date, then the created task is listed with "No due date".
- **AC-001.7** Given the web app's root page is open, when the user submits the form with a title that is empty, only whitespace, or longer than 200 characters after trimming, then the page shows an error message next to the title field, the entered values stay in the form, and no task is created (`GET /tasks` is unchanged).
- **AC-001.8** Given the API is unreachable or answers with a `5xx` status, when a user opens the web app's root URL, then the page loads and shows "Tasks could not be loaded. Please try again later." instead of the list; and when a user submits the create form in that state, the page shows "The task could not be saved. Please try again." and keeps the entered values in the form.

## Non-goals

- Updating, completing or deleting tasks, in the API or in the web app.
- Authentication, users or per-user task lists in the web app or the API (consistent with taskboard6-api spec 001: anyone who can reach the system may create and list tasks).
- Filtering, sorting options, search or pagination of the task list.
- Live updates of the list when another person creates a task (a page reload shows them).
- Carrying over tasks held in the API's memory at the moment persistent storage is switched on.
- Database backups, restore and running in staging or prod; this spec covers dev, the only environment both components run in today.
- Any task field beyond those in taskboard6-api spec 001 (`id`, `title`, `due_date`, `created_at`).

## Assumptions

- **A1** The API contract of taskboard6-api spec 001 stays unchanged (paths, fields, validation, `422` errors, creation order); persistence changes only where tasks live (other option: extend the contract in the same spec; affects AC-001.1, AC-001.3, AC-001.5).
- **A2** Tasks kept in memory before the switch are not migrated: they are lost on every restart already and exist only in dev (other option: a one-off copy into the database; affects AC-001.1).
- **A3** When the database is unreachable the API answers `503` and reports not ready, rather than falling back to memory (other option: `500`, or buffer writes in memory; affects AC-001.2).
- **A4** The web app is a single page at its root URL: the list with the create form above it (other option: separate list and create pages; affects AC-001.3 to AC-001.8).
- **A5** The list shows tasks in the API's creation order, oldest first, and a new task appears at the end (other option: newest first; affects AC-001.3, AC-001.5).
- **A6** Due dates are shown as `YYYY-MM-DD`, and a task without one shows "No due date" (other option: a localized date format; affects AC-001.3, AC-001.5, AC-001.6).
- **A7** The web app checks the title limits (1 to 200 characters after trimming) before sending, and also shows the API's `422` field error if one comes back (other option: rely on the API's validation only; affects AC-001.7).
- **A8** UI texts are in English, with the wording given in the criteria (other option: other wording or language; affects AC-001.4, AC-001.6, AC-001.8).
- **A9** The database is the system's own in-cluster Postgres declared in this repo, not a paid managed service, so it adds no external cost (other option: a managed cloud database; affects AC-001.1).

## Open questions

## Changelog

- 2026-11-07 created
- 2026-11-07 approved
