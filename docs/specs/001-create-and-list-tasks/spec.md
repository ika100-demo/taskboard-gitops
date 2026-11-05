---
spec_id: 001-create-and-list-tasks
title: Create and list tasks
status: approved
priority: P1
shape: gitops-app
---

# 001 — Create and list tasks

## Problem

The taskboard system has no tasks yet: the task API only answers health, readiness, metrics and documentation requests. A user can't write down something to do or look at what is already on the board. This first feature adds the core of the system, creating a task with a title and an optional due date and listing the tasks, so later features have something to build on. Tasks are kept only in memory for now.

## Stories

As a taskboard user, I want to create a task with a title and, if I want, a due date, so that I can record what needs doing.
**Repos:** task-api

As a taskboard user, I want to list all the tasks I created, so that I can see what is on the board.
**Repos:** task-api

## Acceptance criteria

- **AC-001.1** Given the task API is running, when a caller sends `POST /tasks` with `{"title": "Buy milk"}`, then the response is `201` with a JSON object holding a non-empty `id`, `"title": "Buy milk"` and `"due_date": null`.
- **AC-001.2** Given the task API is running, when a caller sends `POST /tasks` with `{"title": "File taxes", "due_date": "2026-12-31"}`, then the response is `201` and the returned task has `"due_date": "2026-12-31"`.
- **AC-001.3** Given two tasks created one after the other, when each create response is read, then the two `id` values are different.
- **AC-001.4** Given a request body with no `title`, a `title` that is not a string, or a `title` that is empty or only whitespace, when a caller sends `POST /tasks`, then the response is `422` with an error that names the `title` field, and no task is created (the list is unchanged).
- **AC-001.5** Given a `due_date` that is not a valid calendar date in `YYYY-MM-DD` form (for example `"31.12.2026"`, `"2026-02-30"`, `"tomorrow"` or `42`), when a caller sends `POST /tasks`, then the response is `422` with an error that names the `due_date` field, and no task is created.
- **AC-001.6** Given a `title` longer than 200 characters, when a caller sends `POST /tasks`, then the response is `422` with an error that names the `title` field, and no task is created; a title of exactly 200 characters is accepted.
- **AC-001.7** Given no task has been created since the task API started, when a caller sends `GET /tasks`, then the response is `200` with an empty JSON array `[]`.
- **AC-001.8** Given tasks A, B and C were created in that order, when a caller sends `GET /tasks`, then the response is `200` with a JSON array of exactly those three tasks in creation order (A, B, C), each with the same `id`, `title` and `due_date` as its create response.
- **AC-001.9** Given a task was created, when the task API restarts, then `GET /tasks` no longer returns it (tasks are held in memory only).
- **AC-001.10** Given a `due_date` in the past, for example `"2020-01-15"`, when a caller sends `POST /tasks` with `{"title": "Old errand", "due_date": "2020-01-15"}`, then the response is `201` and the returned task has `"due_date": "2020-01-15"`.
- **AC-001.11** Given a `title` with leading and trailing whitespace, for example `"  Buy milk  "`, when a caller sends `POST /tasks`, then the response is `201` and the title is returned exactly as sent (`"  Buy milk  "`) both in the create response and in `GET /tasks`.
- **AC-001.12** Given a task was created, when a caller sends `GET /tasks` 10 times in a row, then every response includes that task (all callers see one shared list; the task API runs as a single instance in every environment).

## Non-goals

- Persistence: tasks live in memory and are lost on every restart or redeploy. No database is added.
- Running more than one task API instance: while tasks are in memory it runs as one instance in every environment (see AC-001.12).
- Changing, completing or deleting tasks; fetching a single task by id.
- Filtering, sorting options, searching or paging the list.
- Users, sign-in or per-user task lists: every caller sees every task.
- A web user interface; this feature is the API only.
- Time of day or time zone on the due date.

## Open questions

- ~~Confirm: the API paths are `POST /tasks` and `GET /tasks`, and the fields are named `id`, `title` and `due_date`.~~ Answered: yes, see AC-001.1 to AC-001.8.
- ~~Confirm: the due date is a calendar date in `YYYY-MM-DD` form, with no time or time zone.~~ Answered: yes, see AC-001.2, AC-001.5 and the non-goals.
- ~~May a due date be in the past?~~ Answered: yes, any valid calendar date is accepted, see AC-001.10.
- ~~What is the maximum title length?~~ Answered: 200 characters, see AC-001.6.
- ~~Is a title trimmed of leading and trailing whitespace before it is stored?~~ Answered: no, it is stored exactly as sent (a blank title is still rejected), see AC-001.4 and AC-001.11.
- ~~Confirm: the list is returned as a bare JSON array in creation order, oldest first.~~ Answered: yes, see AC-001.7 and AC-001.8.
- ~~Confirm: invalid input is answered with `422` and an error naming the field.~~ Answered: yes, see AC-001.4 to AC-001.6.
- ~~Confirm: the task API runs as a single replica while tasks are in memory, so every caller sees the same list.~~ Answered: yes, one instance in every environment, see AC-001.12 and the non-goals.

## Changelog

- 2026-11-05 created
- 2026-11-05 amended: user answered all open questions; added AC-001.10 (past due date accepted), AC-001.11 (title stored untrimmed), AC-001.12 (one shared list, single instance); replica non-goal reworded
- 2026-11-05 approved
