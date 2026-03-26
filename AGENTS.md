# This project
This project is a personal (https://github.com/arlegotin/OpenRA) fork of https://github.com/OpenRA/OpenRA.
It's for fixes & updates to push into main repo.

## Branches
You consider branch "agentic" as a your main branch: it forked from the original main branch named "bleed".
Keep "agentic" synced with "bleed" from the original repo.

This "agentic" branch is enriched with agentic tools, which must never be visible outside of "agentic" branch.
Specifically (please autoupdate this list when necessary):
- AGENTS.md
- directory ./planning/*

So whenever something goes to the main repo – you must never allow this tools and their artifacts "leak" into the main project,
but only changes which are naturally belong to the update itself.

Any projects/fixes/update must never be commited to "agentic" itself – but rather fork a working branch.

Whenever you preparing a fix/update for the original repo, follow all the ruls of this repo + make this PR generalle nice and elegant.