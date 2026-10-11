# Maintaining a slab build

You are the agent that builds and keeps a build up to date. The layout and the class and source formats are in the main [SKILL.md](../SKILL.md). Never ask the user for input. If something is ambiguous, make the best judgement and continue.

## `.slab/`

`.slab/` holds the agent's own records for the build:

- **`slab.json`**: where the definition came from. Both fields are required:

  ```json
  {
    "definition_path": "/home/oscar/slabs/personal",
    "definition_commit": "873cee1c2f..."
  }
  ```

  `definition_path` is the absolute path of the definition directory. `definition_commit` is the full git commit hash of that directory, or `null` if it isn't tracked by git.
- **`definition/`**: a full copy of the definition's `classes/` and `sources/`, as used for the last run.
- **`jobs/`**: systemd units for each source's scheduled jobs (see **Jobs**).
- **`<source>.log`**: the run log for each source (see **Run logs**).

## Runs

A run is one load of one source: a backfill or an incremental load. Runs are started by a source's scheduled jobs, or by hand.

To start a new build, the agent is given both a definition directory and a build directory. It does a backfill of every source with no range (the full history). After that, the build directory alone is enough: the agent finds the definition through `.slab/slab.json`.

Every run:

1. Copy the definition's `classes/` and `sources/` into `.slab/definition/`, replacing the previous copy, and write `slab.json`. Copy only those two directories, so a build that sits inside its definition doesn't copy itself.
2. Work from the copy in `.slab/definition/` for the rest of the run.
3. Follow the source's **Backfill load** or **Incremental load** procedure. An incremental load of a source with no `ok` entry in its log becomes a full backfill.
4. Keep only what the source's **Relevant data** says is relevant. For each item, consider the classes that list this source, using what each class says the source contributes and its **Membership** rule. Use the class's identity rule (and the objects' `keys`) to find the matching object, then create or update it.
5. Append an entry to the source's log.
6. Rewrite the build's `SKILL.md` and indexes.
7. Once all other work is done, tell the user about any jobs in the definition that aren't installed, or whose installed units no longer match the definition.

## Jobs

Only install, change or remove jobs when the user explicitly asks. Never do it as part of a build or run.

Install each job in each source's **Jobs** section as a pair of systemd user units in `.slab/jobs/`:

- **`slab-<build>-<source>-<job>.service`**: a oneshot service whose `ExecStart` runs the agent non-interactively, telling it to use the slab skill to run that job's load for this source against the build directory.
- **`slab-<build>-<source>-<job>.timer`**: a timer with the job's `OnCalendar` schedule and `Persistent=true`.

`<build>` is the build directory's name, so units from different builds don't clash. Install them with:

```bash
systemctl --user link <build>/.slab/jobs/slab-<build>-<source>-<job>.service
systemctl --user link <build>/.slab/jobs/slab-<build>-<source>-<job>.timer
systemctl --user enable --now slab-<build>-<source>-<job>.timer
```

To remove a job, disable its timer and delete both units.

## Objects

Each object is one markdown file at `<build>/<class>/<object>.md`. It has a small frontmatter and a free-form body:

```markdown
---
class: person
significance: medium
description: Letting agent at Foxtons who handled the Brockwell Park Row tenancy
keys: [jane.doe@foxtons.co.uk]
---

# Jane Doe

Jane is a lettings negotiator at [Foxtons](../organisation/foxtons.md). She arranged
the user's tenancy at [19 Brockwell Park Row](../residence/19-brockwell-park-row.md)
in 2019 and handled the deposit return in 2021.

## Sources
- [email](../email.md): "Tenancy renewal", 2020-02-14
- [dropbox](../dropbox.md): Houses/19 Brockwell Park Row/tenancy.pdf
```

- **significance**: `high`, `medium` or `low`, judged by the class's **Significance** guidance. Revisit it as more is learned about the object.
- **description**: one line summarising the object. It feeds the class index.
- **keys**: the values the class's identity rule matches on, so the agent can find matches without reading every body.
- **Body**: written freely, shaped by the class's "Worth knowing" guidance and by what is actually known. Refer to other objects with relative markdown links.
- **Sources** section: where the facts came from, each linked to the build's `<source>.md`, and specific enough to find the original again.

When updating an object, keep its description, body and sources accurate as a whole, not only by appending. The user doesn't edit objects.

## Build skill and indexes

At the end of every run, rewrite these files from scratch, rather than patching them:

- **`SKILL.md`**: makes the build usable as a skill. Its frontmatter `name` is the build directory's name, and its `description` says what the dataset covers and when to use it. The body says that this is a slab build maintained by an agent and not to be edited by hand, explains the layout (`<class>.md`, `<class>/INDEX.md`, `<source>.md`, object files), has one line per class linking to `<class>.md` and `<class>/INDEX.md`, with an object count, and one line per source linking to `<source>.md`.
- **`<class>.md`**: what the class is, what qualifies as a member, and what each significance level means, written for someone reading the data. Base it on the class definition, but leave out agent instructions (identity rules, load procedures) and don't refer to the definition.
- **`<source>.md`**: the source, written for someone using the data. Say what it is, which classes it informs, and when it was last loaded successfully. Explain how to go back to it: how to find the original item behind an entry in an object's Sources section, and how to get data newer than the build. Base it on the source definition, but leave out load procedures and don't refer to the definition.
- **`<class>/INDEX.md`**: one line per object, `- [Name](<object>.md): <description>`, taken from each object's `description`. Group the lines under `## High`, `## Medium` and `## Low` headings by significance.

## Run logs

`<build>/.slab/<source>.log` records each run of a source. Append one entry after every run, and never rewrite earlier entries. An entry is exactly:

- A run line: the run's start time (UTC), `backfill` or `incremental`, `ok` or `failed`, then `range=<from>..<to>`, the range of data the load covered. A failed run adds `reason="..."`.
- One indented line for each class that lists the source: the class name, then `created=<n> updated=<n>`.

```
2026-10-10T18:04Z incremental ok range=2026-10-09T07:00Z..2026-10-10T18:04Z
  person created=1 updated=2
  organisation created=0 updated=1
2026-10-11T07:00Z incremental failed range=2026-10-10T18:04Z..2026-10-11T07:00Z reason="not authenticated"
  person created=0 updated=0
  organisation created=0 updated=0
```
