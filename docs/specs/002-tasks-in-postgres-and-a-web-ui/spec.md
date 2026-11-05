---
spec_id: 002-tasks-in-postgres-and-a-web-ui
title: Tasks in Postgres and a web UI
status: done
priority: P1
shape: gitops-app
---

# 002 — Tasks in Postgres and a web UI

## Problem

Spec 001 gave the taskboard a task API that can create and list tasks, but the tasks live only in the task API's memory: every restart or redeploy throws away the whole board (AC-001.9), and the API has to run as a single instance (AC-001.12). The only way to use the board is to send raw HTTP requests, so nobody who is not a developer can use it. This feature keeps tasks in a Postgres database so they survive restarts and redeploys, and adds a web UI where people can see the board and add tasks to it. The task API contract from spec 001 (paths, fields, validation, `422` errors, oldest-first bare array) stays as it is. Only the in-memory consequences change. This spec supersedes AC-001.9 and the single-instance constraint of AC-001.12 and of spec 001's non-goals.

## Stories

As a taskboard user, I want the tasks I created to still be there after the system restarts or is updated, so that I can rely on the board as my record of what needs doing.
**Repos:** task-api, taskboard-gitops

As a taskboard user, I want to open the taskboard in my browser and see every task with its due date, so that I can check the board without sending API requests.
**Repos:** taskboard-web, task-api, taskboard-gitops

As a taskboard user, I want to add a task with a title and an optional due date from the web UI, and be told clearly when my input is wrong, so that I can record work without knowing the API.
**Repos:** taskboard-web, task-api

## Acceptance criteria

Persistence (task API)

- **AC-002.1** Given a task was created with `POST /tasks`, when the task API restarts, then `GET /tasks` still returns that task with the same `id`, `title` and `due_date` as its create response (supersedes AC-001.9).
- **AC-002.2** Given tasks A and B were created, the task API was then redeployed with a new version, and task C was created after that, when a caller sends `GET /tasks`, then the response is `200` with exactly A, B and C in creation order (A, B, C), and C's `id` differs from A's and B's.
- **AC-002.3** Given the database is unreachable, when a caller sends `POST /tasks` with a valid body or `GET /tasks`, then the response is `503` with a JSON error saying the task store is unavailable (never `201` and never an empty list `[]`), and once the database is reachable again no task from the failed `POST` appears in `GET /tasks`.
- **AC-002.4** Given the database is unreachable, when the task API's readiness endpoint is called, then it answers with a non-`200` status, and it answers `200` again within 30 seconds after the database is reachable again, without the task API being restarted.
- **AC-002.5** Given the task API runs as 2 instances and a task was created, when a caller sends `GET /tasks` 10 times in a row, then every response includes that task (one shared list regardless of the number of instances; supersedes the single-instance constraint of AC-001.12).
- **AC-002.6** Given spec 001's criteria AC-001.1 to AC-001.8, AC-001.10 and AC-001.11, when they are run against the task API backed by the database, then they all still pass (the API contract is unchanged; for AC-001.7 "since the task API started" now reads "in this environment's database").

Web UI

- **AC-002.7** Given tasks A, B and C were created in that order (through the API or the web UI), when a user opens the web app's root URL, then the page lists exactly A, B and C, oldest first, each showing its title and its due date in `YYYY-MM-DD` form, and a task without a due date shows "No due date".
- **AC-002.8** Given no task exists, when a user opens the web app's root URL, then the page shows the message "No tasks yet." and the form to add a task.
- **AC-002.9** Given the web app's root URL is open, when the user enters the title "Buy milk", picks the due date 2026-12-31 and submits the form, then "Buy milk" with "2026-12-31" appears at the end of the list without the user reloading the page, the form is emptied, and `GET /tasks` on the task API includes that task.
- **AC-002.10** Given the web app's root URL is open, when the user submits the form with a title but no due date, then the task appears at the end of the list with "No due date" and `GET /tasks` returns it with `"due_date": null`.
- **AC-002.11** Given the web app's root URL is open, when the user submits the form with an empty or whitespace-only title, then the message "Enter a title." is shown at the title field, no task is created (the list on the page and `GET /tasks` are unchanged), and the entered due date is kept.
- **AC-002.12** Given the web app's root URL is open, when the user submits a title longer than 200 characters, then the message "The title can have at most 200 characters." is shown at the title field, no task is created and the entered text is kept; a title of exactly 200 characters is accepted.
- **AC-002.13** Given a task with the title `<b>bold</b> & <script>alert(1)</script>` exists, when a user opens the web app's root URL, then the title is shown literally as those characters, nothing in it is rendered as markup and no script runs.
- **AC-002.14** Given the task API cannot be reached or answers with an error, when a user opens the web app's root URL, then the page shows "Tasks could not be loaded. Try again later." instead of a list or the "No tasks yet." message; and when the user submits a valid task in that state, the page shows "The task could not be saved. Try again." and keeps the entered title and due date.
- **AC-002.15** Given the web app's root URL is open, when the user submits the title `"  Buy milk  "` (with leading and trailing spaces), then `GET /tasks` returns the task with the title exactly `"  Buy milk  "`, untrimmed, as AC-001.11 stores it.

## Non-goals

- Moving tasks that exist in memory today into the database: they are lost once when the database-backed task API is deployed.
- Changing, completing or deleting tasks, in the API or the web UI; a detail page for one task.
- Users, sign-in or per-user task lists: everyone who reaches the web UI or the API sees and can add every task.
- Filtering, sorting options, searching or paging, in the API or the web UI.
- Live updates: tasks added by someone else appear in an open page only after it is reloaded.
- Changes to the task API's request and response contract beyond the `503` of AC-002.3.
- Database backups and restore; running the web UI outside `dev` (promotion is a separate step).
- Time of day or time zone on the due date (unchanged from spec 001).

## Open questions

- ~~Confirm: tasks held in memory by the current task API are not migrated and are lost once at the switch to the database.~~ Answered: yes, not migrated, see the first non-goal.
- ~~May the task API now run as more than one instance?~~ Answered: yes, the single-instance rule is lifted and the shared list must hold with 2 instances, see AC-002.5; how many instances run in each environment stays a deployment setting (1 in dev for now).
- ~~What does the task API answer when the database is unreachable?~~ Answered: `503` with a JSON error, and the readiness endpoint reports not ready, see AC-002.3 and AC-002.4.
- ~~How do people reach the web UI in the browser?~~ Answered: the web app is published through the gateway in dev under its own host, like the task API, and the page is at its root URL, see AC-002.7 to AC-002.15.
- ~~How does the web UI show due dates and missing due dates?~~ Answered: `YYYY-MM-DD` as stored, and "No due date" when there is none, see AC-002.7, AC-002.9 and AC-002.10.
- ~~In which order does the web UI list tasks?~~ Answered: oldest first, the same as `GET /tasks`, with new tasks added at the end, see AC-002.7, AC-002.9 and AC-002.10.
- ~~Confirm the messages users see.~~ Answered: as written, in English: "No tasks yet.", "Enter a title.", "The title can have at most 200 characters.", "Tasks could not be loaded. Try again later." and "The task could not be saved. Try again.", see AC-002.8, AC-002.11, AC-002.12 and AC-002.14.
- ~~Confirm: the web UI sends the title exactly as typed, without trimming.~~ Answered: yes, see AC-002.15.

## Changelog

- 2026-11-05 created
- 2026-11-05 amended: user answered all open questions, all as suggested; added AC-002.15 (web UI sends the title untrimmed); no other criteria changed
- 2026-11-05 approved
- 2026-11-05 building
- 2026-11-05 done
