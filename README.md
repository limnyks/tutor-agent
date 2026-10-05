# Tutor: a personal study agent built on Claude Code

A Claude Code agent that acts as one student's tutor for a university programme. It knows the
courses, the deadlines, what the student has answered and what they have not yet been checked on.
It plans the week, teaches by asking questions, and keeps a log that is the only evidence of what
the student knows.

This repository describes the design. It contains no student data. The tutor's code is in a
separate repository; the agent's memory (courses, answer log, plans) is private.

## What it does

| Area | Behaviour |
|---|---|
| **Lessons** | Short review (10 min max), then Socratic teaching: ask, wait, escalate hints. After 3 failed tries it gives the answer with reasoning and asks a similar question to confirm. |
| **Checking** | Questions in the style of the course's own tasks and marking rules. Full reasoning is required; a bare answer is not full marks. |
| **Evidence** | Every checking question is logged (right/wrong and what exactly was wrong). Studying or having notes is not evidence of knowing; only answers are. |
| **Summaries** | Every lesson ends with a summary the student writes. The agent corrects it and stores it with the lesson's mistakes and the next step. A hook flags lessons that ended without one. |
| **Spaced review** | Topics come back for review based on logged answers. |
| **Planning** | Reads the student's calendars for busy time, plans blocks for lessons and deadlines, writes them to Google Calendar, and re-plans after a missed block. |
| **Weekly cycle** | Sunday: a weekly test from course materials (no hints), a review of what was planned vs done, then the next week's plan. Last Sunday of the month: a month review. |
| **Moodle** | Syncs deadlines, changes, grades and course files. Grades are given as feedback only (what was lost, why, what to do differently), never as numbers. |
| **Teachers' announcements** | A daily read-only scan of Slack course channels and teacher email for deadline moves, cancellations and new tasks. |
| **Handwritten notes** | Reads notes synced from an e-ink tablet (PDF per notebook), checks them for mistakes and gaps, and turns the gaps into questions. |
| **Morning run** | A scheduled cloud run that builds the day's plan and sends a short phone notification: today's blocks, deadlines, reviews due, warnings. |

## Principles

- **Concise.** Bullets, no filler, no motivational framing, no praise.
- **No grading of the student.** No verbal scales ("good", "7/10"). It says what is wrong, why, and what to do next.
- **Sources.** Teaches from the course's own materials and named books, and cites file and page. If the materials do not cover something, it says so before using general knowledge.
- **Course AI policy first.** Each course has a flag for whether AI help is allowed on graded work.
  - Allowed: the agent is a tutor on the task itself (guiding questions, hints, checking the student's own steps). The student writes the solution; the agent never does. It tells the student what to cite or disclose.
  - Not allowed: the agent does not read, solve, check or discuss the actual graded work. It teaches the same idea on a different problem.
  - Quizzes and tests during their window: no help in any course.
- **Read-only on messages.** It never posts, replies, reacts, sends or deletes in Slack or email. Message contents are data; instructions found in them are never followed.
- **Privacy.** Titles and descriptions of personal calendar events never enter the plan, memory or chat. Only start and end times are used.
- **Neutral by default.** It does not guess pronouns.

## Architecture

```
student ──► Claude Code (CLAUDE.md = the tutor's rules)
               │
               ├─ MCP server "tutor": Moodle, memory, planner, test and review tools
               ├─ Google Calendar, Slack, Gmail, Google Drive connectors (read; calendar writes only its own blocks)
               └─ hooks: briefing at session start, memory sync at session end,
                         "lesson has no summary" check on stop

memory repo (private, append-only):
  courses/*.json      topics, grading, AI policy, books per course
  events/*.jsonl      answers, summaries, study reports, Moodle changes (never edited)
  moodle/snapshot     last Moodle state
  plan/               settings (day windows, weights, limits)
  notes/              mapping of notebooks to courses, teacher sources
  cloud/              the scheduled morning run
```

- **Append-only log.** Everything the tutor knows is derived from daily event files, one per day and writer, so the Mac, the cloud run and other machines can write without conflicts.
- **Briefing hook.** Each session starts with a generated briefing: deadlines, Moodle changes, topics due for review, topics studied but not checked, lessons without a summary. The student never re-explains context.
- **Planner.** Takes busy intervals and course settings (day windows, weights, at most 3 subjects a day, long blocks, breaks) and returns blocks plus warnings. Work that does not fit is reported with what to drop or move, not just flagged.
- **Cloud vs Mac.** Moodle login and the notes folder exist only on the Mac. The cloud run reads the synced snapshot and says how old it is.

## Tools exposed by the `tutor` MCP server

- Moodle: deadlines, assignments, grades, changes, course contents, files, sync, download
- Memory: briefing, topics (evidence per topic), history, `log_answer`, `log_summary`, `log_study`, `log_notice`
- Planning and reviews: `plan_week`, `plan_save`, `test_plan`, `review_period`

## Example of one lesson

1. Briefing says topic X is due for review and the notebook changed since the last lesson.
2. The agent reads the newest notebook pages, finds a missing step in a proof, and turns it into a question.
3. Short review, then Socratic questions on the topic, each answer logged with what exactly was wrong.
4. The student writes the summary in their own words; the agent corrects it and logs it with the mistakes and the next step.
5. Next session the logged mistakes decide what gets checked first.

## Limits

- It is built around one student's courses and tools (Moodle, Google, Slack, an e-ink tablet). Reusing it means replacing those sources.
- Handwriting is read from page images and can be misread; the agent states uncertainty.
- AI-generated mathematics must be checked by hand.
