Potential material for a case write up.

This is a case (one task) from a job I worked at.
This task involved cleaning up stale artefacts in the databases.
Product had two distinct databases for different kinds of workloads.
There were two deployments - dev environment (staging like), and production.
In production mode, one database was also implemented as two, switching every day.
At that point in time, there were a lot of stale artefacts - some obvious, like "*_tmp", some are not.

My overall approach was the following:
1. Make a script that gets the list of tables and "pokes" them all. This gives me a list to work with and surfaces artefacts that are completely broken, like "Table doesn't exist in engine";
2. Make a script that will try to parse database schema control artefacts (migrations) in the repository to see if the tables are version controlled properly;
also parse them from VCS history of schema control. This surfaced tables that were not a part of version control - some were created by hand for some reasons,
sometimes in dev deployment for WIP, some are results of code that creates temporary artifacts but fails to clean them;
3. Make a script that will parse table names from other code and its VCS history. This surfaced tables that were artefacts of deleted scripts.
This also gave me a chunk of info per table to analyze its usage.
4. Add detection of time related columns into the table script. This surfaced tables that were technically operational, but not used anymore for anything. This information was added to the analysis.
5. Make a schema for structurally saving the results of my analysis of all that information and verdicts - delete tables via migrations, fix scripts, or delete manually by DBA.
6. Made a tool that compiles all of the data into a big google sheet. Present the results.
7. Afterward, made fixes to scripts and later committed version controlled changes to appropriate artefacts, and passed instructions to DBA.
8. Worked to create a set of conventions for easier analysis in the future, like unified naming schemes etc.

Potential questions to cover:
"Did you ever hit ambiguity you couldn’t resolve?" - Certainly as expected.
In all cases I had to either consult with others or leave my structured verdict as "ambiguous";
this was reflected in the compiled results for leads and other engineers to be able to comment upon.
Tables that were not in VCS but recent were all either: intermediate WIP artefacts in dev env;
or created by scripts that made temporary artefacts (like a temporary switch variable).
All of the work, scripts, and results were open for code review and analysis by the team as a regular part of the working process.

"Migration history" - they were plain SQL, which is already structured, but for my purposes, a regex and git regex search sufficed.
Of course, there were ambiguities - this is why I had a step where I manually worked through all gathered data on each table to analyze.
Some were trivially in migrations, some were e.g. mentions, some were in migrations at some point but then deleted - e.g. a dev worked on a feature,
added related table manually to dev db to code something ad hoc, made a migration, but then changed his mind but forgot to delete the table,
and the commit was left since there is no squashing or cleaning up before master merge. This is why a naming scheme for any intermediary work was later decided.
Some tables were worked on ad hoc live in the dev db, which is why surfacing everything for team observability was important.

"Safety net" - absolutely nothing was relied on the structured "verdict" alone, which you would not ask if you read my text carefully.
But there were extra measures to avoid incidents - all artefacts scheduled for removal were first simply renamed with a clear scheme, and a task for removal after some time was added.
This was done for both version controlled artefacts and the ones DBA had to deal with manually.

"Manual artifacts in dev" - policies were declared, naming schemes were added.
This cannot be avoided fully since the entire context of the project makes it infeasible (a giant DB that is not allowed to be replicated for local isolated dev).
This all took about a week for hundreds of tables. I did all of the analysis and automation alone,
but I made people analyze and react to the results especially in cases of greater ambiguity (I had a checklist of points that had to be reviewed to proceed).
Of course, project lead and DBA are absolutely required to review this. And code reviews were mandatory in that company.

Extra points:
1) The project has mechanisms for surfacing errors so if the tables were ever referenced, a bug alert would surface. Metrics were only available to DBA/CTO level.
2) There was zero risk, since analysis only used zero cost db tools - trivial schema introspections and trivial selects.
That was extra important, because analysis had to run against both dev and prod deployments - to cleanup both environments.
That also surfaced many discrepancies between them, as well as some errors in overall workflows.
3) I did CI tests for version controlled database schema, though that is unrelated since final artefacts should end up with clean names;
but it is not quite possible to do for dev env where multiple concurrent work is done by many developers.
It was also not entirely feasible at the time to make tooling for detection of improper use of temporary tables in scripts, but it was added to the plans for later extensions of the validations.
The tooling for this task was committed for later reuse.
4) The same way I dealt with every other table. There were no issues, since if e.g. data shown that a table is not used for a long time,
I investigated related Jira data, poked proper people for info, and in cases of ambiguity the process is as stated - mark ambiguity,
request for comments - then either do nothing if it is really just a rarely used, but active useful artefact, or mark it for later removal.
Removals were abstracted as separate tasks with the entire process from jira to review to merge to deploy. If I had to do it again,
I would do the same since there is no issues in the approach. What is more important, is that I would avoid ending up in a situation where something like this has to be done.
