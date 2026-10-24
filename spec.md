# SEMO Capstone Project Tracker · Feature F3: Add a capstone project

This is the spec for UC-2, Add Capstone Project, written in the "From Story to Spec" activity. It is the
input for the prompts in the "Prompt Engineering" lecture. Attach it in Copilot Chat with `#file:spec.md`.

The spec has two layers:

- **Behavior layer** (human-led): rules, examples, and acceptance criteria.
- **Technical layer** (AI-drafted, human-verified): the interface the code must follow.

## Traceability

| Source | ID | Text |
|---|---|---|
| User story | S3 | As the faculty coordinator, I want to add a finished capstone project to the tracker so that students can find it later and the department keeps a complete record of its capstone work. |
| SRS | REQ-7 | The system shall let a user signed in with the faculty role add a project record containing title, semester, team member names, faculty advisor, abstract, and technologies (all required), and a repository link and Confidential setting (both optional). |
| SRS | REQ-8 | The system shall refuse to save a project record that has an empty required field and shall name each missing field to the user. |
| SRS | REQ-9 | When a faculty user saves a new project whose title and semester match an existing record, the system shall display a duplicate warning and let the user either cancel or save anyway. |
| SRS | REQ-12 | If a user who is not signed in with the faculty role attempts to add, edit, archive, or export project records, the system shall deny the action and display the sign-in page. |
| SRS | NFR-3 | The system shall end a faculty user's session after 30 minutes without activity, after which add, edit, archive, and export actions require signing in again. |
| Use case | UC-2 | Add Capstone Project |

## Rules (decided by me)

- **R1.** A required field is missing if it is empty after leading and trailing spaces are removed. Team members are entered one per line, and at least 1 non-blank name is required.
- **R2.** A new project is a possible duplicate if an existing record (archived or not) has the same semester and the same title after trimming, squeezing double spaces, and ignoring capitals.
- **R3.** The semester is picked from a list: Fall, Spring, or Summer plus a year from 2015 through the current calendar year. A typed or out-of-range semester is rejected.
- **R4.** A faculty session expires when 30 minutes or more have passed since the last request to the server. Exactly 30:00 counts as expired. Any page load, search, or save resets the timer. The session check runs before the missing-field and duplicate checks.

## Acceptance criteria

| ID | Given | When | Then |
|----|-------|------|------|
| F3-AC1 | a signed-in coordinator has filled in all six required fields and no record has the same title and semester | they submit the form | "Project added" is shown with the new detail page, the record shows today's date and the coordinator's name, and a search for the title still finds it after a page reload |
| F3-AC2 | the form has Title "&nbsp;&nbsp;&nbsp;" (only spaces) and an empty Faculty advisor, and every other field is filled in | the coordinator submits the form | nothing is saved, "Missing: Title, Faculty advisor" is shown, and everything else they typed is still in the form |
| F3-AC3 | "Bookstore Inventory App" already exists for Fall 2025 | the coordinator submits "bookstore inventory app" for Fall 2025 | nothing is saved yet, and "A project with this title already exists for Fall 2025." is shown with **Cancel** and **Save anyway** buttons |
| F3-AC4 | the duplicate warning from F3-AC3 is showing | the coordinator clicks **Save anyway** | the second record is saved and a Fall 2025 search lists both projects |
| F3-AC5 | the coordinator's last request to the server was exactly 30:00 ago | they submit the form | nothing is saved and the sign-in page is shown |

Boundary examples for R1 and F3-AC2:

| Input | Result |
|---|---|
| `findMissingFields({ ...valid, title: "   " })` | `["title"]` |
| `parseTeamMembers("")` | `[]` (0 names, missing) |
| `parseTeamMembers("Ana Lopez")` | `["Ana Lopez"]` (1 name, accepted) |

Boundary examples for R2 and F3-AC3 (existing record: "Bookstore Inventory App", Fall 2025):

| Input | Result |
|---|---|
| "bookstore&nbsp;&nbsp;inventory app ", Fall 2025 | duplicate warning (differs only in case and spaces) |
| "Bookstore Inventory Apps", Fall 2025 | no warning (one extra letter) |
| "Bookstore Inventory App", Spring 2026 | no warning (different semester) |

Boundary examples for R3 (current year 2026):

| Input | Result |
|---|---|
| Spring 2015 | in the list |
| Fall 2014 | not in the list |
| Fall 2026 | in the list |
| Spring 2027 | not in the list |

Boundary examples for R4 and F3-AC5 (times in milliseconds):

| Input | Result |
|---|---|
| `isSessionExpired(0, 1799000)` | `false` (29:59, still signed in) |
| `isSessionExpired(0, 1800000)` | `true` (exactly 30:00) |
| `isSessionExpired(0, 1801000)` | `true` (30:01) |
| `isSessionExpired(0, 600000)` | `false` (10 minutes idle) |
| `decideSave(validProject, [], true, false)` | `"signin"` (nothing saved) |

## Interface (technical layer: AI-drafted, human-verified)

All functions live in `src/records.js`. They are pure: no DOM, no storage, no network, and they
never change their inputs. Tests call them directly. The current time and year are passed in, so
tests don't depend on the clock.

- `findMissingFields(project) → string[]`
  - Returns the names of the required fields that are missing under R1, for example `["title", "facultyAdvisor"]`. Returns `[]` if the project is complete.
- `parseTeamMembers(text) → string[]`
  - Splits the team box into one name per line, trims each name, and drops blank lines.
- `normalizeTitle(title) → string`
  - Trims, squeezes double spaces to one, and lower-cases the title (R2).
- `isDuplicate(project, existingProjects) → true | false`
  - `true` if any existing project, archived or not, has the same semester and the same normalized title (R2).
- `semesterOptions(currentYear) → string[]`
  - Every semester from `"Spring 2015"` through `"Fall " + currentYear` (R3).
- `isSessionExpired(lastRequestMs, nowMs) → true | false`
  - `true` if `nowMs - lastRequestMs` is 1,800,000 ms (30 minutes) or more (R4).
- `decideSave(project, existingProjects, expired, saveAnyway) → "signin" | "missing" | "duplicate" | "saved"`
  - Runs the checks in this order: session (R4), missing fields (R1), duplicate (R2). `saveAnyway: true` skips the duplicate check (F3-AC4).

### Data shapes

```js
// A project record
{
  id: "p-0042",
  title: "Bookstore Inventory App",
  semester: "Fall 2025",
  teamMembers: ["Ana Lopez", "Ben Carter"],
  facultyAdvisor: "Dr. Smith",
  abstract: "A web app that tracks textbook stock for the campus bookstore.",
  technologies: ["React", "Node.js"],
  repoLink: "https://github.com/example/bookstore",   // optional, "" if none
  confidential: false,                               // optional, default false
  archived: false,
  addedOn: "2026-10-24",
  addedBy: "Faculty Coordinator",
}

// The form, before it is saved: the same fields as text, with teamMembers as one string
// (one name per line) that parseTeamMembers() turns into an array.
```

### Input

- The semester is a dropdown filled from `semesterOptions(currentYear)`. Nothing else can be typed there.
- UI code lives in `src/app.js`. It calls the pure functions; it never re-implements their rules.

## Parked questions (answered by other features or later sprints)

- If the session expires while the coordinator is filling in the form, is what they typed kept? → Sprint 2 (needs a draft-saving design).
- What does a student or signed-out user see if they open Add Project (REQ-12)? → sign-in feature, as F3-AC6.
- Is there a longest allowed title? → not decided yet.

## Out of scope for F3

Form styling, the sign-in page itself, editing and archiving records, the CSV export, and search.
