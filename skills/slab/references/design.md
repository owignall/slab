# Designing a slab definition

You are helping the user write a definition: the `classes/` and `sources/` files described in the main [SKILL.md](../SKILL.md). Once it's written, an agent will build and maintain the dataset with no further input, so everything it needs has to be in these files.

Design is a back and forth with the user, not a one-shot write-up. Draft, then ask the user about anything important they haven't made clear, such as what counts as a member of a class, what makes an object significant, what in a source is noise, or how often data needs refreshing. Don't fill those gaps with guesses. Revise and repeat until the definition is strong enough to run without further input.

- Start from what the user wants to know, then work out the classes and sources that answer it.
- Keep classes few and distinct.
- For each source, check that the agent can actually access it before writing its procedures.
- Keep the definition in git if you can. Each build records the commit it was last run from.

## Good class definitions

- **Write for the consumer.** The build is read by whoever uses its skill later. Ask what they'll want from these objects, and make "Worth knowing" answer that, rather than recording everything a source happens to mention.
- **Draw a clear boundary.** "Membership" should let the agent decide quickly whether something is an object of this class. Name the near misses that don't count, e.g. people who only appear in marketing emails.
- **Rank by significance.** Members of a class rarely matter equally: a sibling matters more than a one-off courier. Describe what makes an object `high`, `medium` or `low` in terms the agent can judge from the data, such as closeness, frequency or recency.
- **Make identity unambiguous.** Use the most stable identifiers the sources provide, so the same thing seen twice becomes one object, not two.
- **Link to sources.** List every source that informs the class and what each contributes.

## Good source definitions

- **Filter at the source.** Most sources are mostly noise for the classes they feed. "Relevant data" should say what to keep and what to skip (marketing, notifications, automated messages), so objects aren't bloated by irrelevant material.
- **Write procedures, not descriptions.** The load sections are steps the agent follows: which tool or command to use, how to page through results, and how to handle errors such as expired auth.
- **Backfill takes a range.** A backfill loads a given range of the data, e.g. a date range. The first load of a build is a backfill with no range, meaning everything. Say what form the range takes for this source.
- **Incremental finds its own starting point.** An incremental load must work out how far the loaded data already goes (by default, the end of the range in the last `ok` run in the build's log for this source), then load from there up to now.
- **Schedule by value and change rate.** Pick each job's schedule from how valuable the data is and how often it changes. A busy, important inbox may justify hourly loads; a rarely changing document folder might need weekly ones. Write the reason down with the schedule.
