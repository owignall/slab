---
name: slab
description: Slab is a framework for agent-maintained datasets, built from user-defined classes of objects (e.g. person) and sources of data (e.g. a Slack account). Use when helping a user design slab classes and sources, or when running the agent that builds and maintains a slab dataset.
---

# Slab

Slab is a framework for datasets that AI agents build and maintain. The user defines two things:

- **Classes**: kinds of object worth tracking, e.g. `person`, `company`, `project`.
- **Sources**: places data comes from, e.g. a Slack account, an email inbox, a folder of documents.

An agent then reads the sources and creates or updates **objects**, which are members of the classes (e.g. the `person` John Smith). Once the classes and sources are defined, the agent builds and maintains the dataset on its own, with no further human input.

A slab has two parts:

- **Definition**: the classes and sources, written by the user. These are instructions for the agent.
- **Build**: the dataset the agent produces from a definition. A build is itself a skill, so other agents can use the data. One definition can have any number of builds.

There are two roles. Read the reference for the role you're in:

1. **Design**: help the user write a good definition (done with the user). See [references/design.md](references/design.md).
2. **Maintain**: load the sources and keep a build up to date (done autonomously). See [references/maintain.md](references/maintain.md).

## Layout

The definition and the build are separate directories, and each can be anywhere. A build is often kept somewhere else entirely, but it can also sit inside its definition directory, e.g. as `data/`.

```
<definition>/
  classes/<class>.md          # user-defined
  sources/<source>.md         # user-defined

<build>/                      # agent-maintained
  .slab/
    slab.json                 # where the definition came from
    definition/               # copy of the definition used for the last run
      classes/<class>.md
      sources/<source>.md
    jobs/                     # systemd units for scheduled loads
    <source>.log              # run log, one per source
  SKILL.md                    # the build as a skill: what the dataset is and how it's laid out
  <class>.md                  # what the class is, for readers of the data
  <source>.md                 # where data comes from, and how to look up the original
  <class>/
    INDEX.md                  # one line per object
    <object>.md
```

The build is the dataset itself, and it should make sense on its own, without the definition.

## Classes

A class file describes what qualifies as a member, what's worth knowing about members, how significant each one is to the user, which sources inform the class, and how to tell when two mentions are the same object. It guides the agent's judgement; it is not a template for object files.

```markdown
---
name: person
---

A person the user interacts with: who they are, how the user knows them, and what
they've dealt with each other about.

## Membership
Someone the user has dealt with directly: exchanged messages with, met, or done
business with. Not people who only appear in newsletters, mailing lists or CC lines.

## Worth knowing
Their name and contact details. Their role and where they work. How the user knows
them and the history of their dealings. Anything notable about the relationship.

## Significance
- high: family, close friends, and people the user works with day to day
- medium: people the user has dealt with more than once, or recently
- low: one-off contacts

## Sources
- slack: messages and threads with the user
- email: correspondence, plus names and signatures in emails

## Identity
Match on email if present, otherwise full name plus company.
```

- **Membership**: what qualifies something as an object of this class, and what doesn't.
- **Worth knowing**: prose guidance on what to capture, including relations to other classes. Not a list of required fields.
- **Significance**: how to judge each object's significance to the user, as `high`, `medium` or `low`.
- **Sources**: the sources that inform this class, and what each one contributes. This is the only place where the link between classes and sources is recorded.
- **Identity**: the rule for matching a mention to an existing object.

## Sources

A source file describes where the data is, what in it is relevant, how to load it, and when to run loads.

```markdown
---
name: email
---

The user's Outlook inbox, read with `email_tool.py` from the my-data skill.

## Relevant data
Correspondence with real people and organisations the user deals with. Skip
marketing, newsletters, notifications and receipts unless they concern something
the user owns or rents.

## Backfill load
Steps to load every email in a given date range, or the full history when no range
is given.

## Incremental load
Steps to work out the date of the newest email already loaded, then load everything
from there up to now.

## Jobs
- incremental: `OnCalendar=*-*-* 07:00`. Daily: email changes often and is the
  main record of the user's dealings, but nothing in it needs to be in the build
  within the hour.
```

- **Relevant data**: what to keep and what to ignore, so the build isn't bloated with noise.
- **Backfill load**: a procedure for loading a given range of the data, or all of it. The first load of a build is a backfill of everything.
- **Incremental load**: a procedure that works out how far the data already loaded goes, then loads forward from there up to now.
- **Jobs**: scheduled loads, usually one incremental job. Each has a name, the load it runs, a systemd `OnCalendar` schedule, and why that schedule suits the source.

Source files don't record any state. How far the data has been loaded comes from the build's run log.
