# Arini Engineering Standard Operating Procedures

Source: abizer's Notion doc — [WIP] Standard Operating Procedures

This is the authoritative engineering culture document. The review principles
in SKILL.md are derived from this philosophy. When in doubt about reviewer
intent, refer back here.

## Philosophy (verbatim)

1. Code is a liability. The goal is to delete as much of it as we can.
2. Sloth is a cardinal sin. Laziness serves one person at the expense of everyone else.
3. Always understand what you are doing. DO NOT LAND CODE YOU DO NOT UNDERSTAND.
4. Think about interfaces and abstractions before writing the implementation.
5. Embrace simplicity, abhor complexity.

## Key operational principles not captured in review rules

- **Linear commit history**: Rebase, don't merge. Squash-and-merge or rebase-and-merge for PRs.
- **Incremental PRs**: Each PR does exactly one thing. Develop in parts that build on one another.
- **Design first**: Write out what you're achieving and how before writing code.
- **Minimize localdev/prod differences**: No special-casing for localdev.
- **Refactor as you go**: Fix/improve things in the process of implementing features, but separate into independent commits.
- **Edge case heuristic**: First time, special-case it. Second time, refactor. Don't let special cases pile up.
- **Don't scar on the first cut**: Don't overindex on something going wrong once.
- **It's OK to say no to the customer**: Don't introduce warts to ink a deal.
- **Recovering from errors**: Think carefully about revert vs fix-forward. Reverting is not always safe (e.g., dropped tables don't come back).

## On working with AI

> The models are smarter than you. That means they can handle more complexity
> than you, and their code reflects that. Don't blindly accept code you don't
> understand just because it looks like it works!

> The models DO NOT KNOW WHAT GOOD CODE LOOKS LIKE and will lie to you about
> it out of total ignorance. If you can't tell if some code is good or bad,
> read more code from other widely-used projects that are accepted as having
> good practices!

## On database performance

> Each database call is a network call. In localdev the DB is running on your
> laptop with microsecond latency. In prod it's AWS Aurora with 2-5ms per query.

> Database writes take write locks. If multiple writes in the same logical
> operation aren't wrapped in the same transaction, we get inconsistent DB
> state if a write fails.

> If we can do work in SQL, we should do it in SQL, not bring the data back to
> process it in Python. Python is slow for CPU-bound work, MySQL is written in
> C++ and has been optimized by wizards.

> The database is for long-lived information. If data is temporary or changes
> rapidly, it should live in a cache. If we're only going to use data once,
> put it in S3, not an OLTP store.

## On data design

> Don't overnormalize for the sake of normalization! If you're storing config
> data indexed by a primary key, just store it as a JSON column. Don't spread
> data that shares an index key across multiple tables.

> Design schemas from the perspective of how you're going to query the data.
