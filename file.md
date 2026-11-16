AIDLC Docs Insights: Concept
Nov 16, 2026 · @Bikashranjan 
Summary
AIDLC Docs Insights is a read-only dashboard that shows how many Work Plans, SRS, SAD, Test Cases and Pull Requests our Co-work AIDLC skills have produced, per repository.
It needs no change to Co-work or to the skills. The skills already save every document into the docs folder of each project's Bitbucket repository, so the dashboard simply reads what is there.
A clickable mock with sample data is ready for review: AIDLC Docs Insights mock.
The problem today
Teams use Co-work skills to generate planning, design and test documents, but nobody can see the overall picture.
• There is no single place that shows how many documents were created, for which project, and when.
• To check a feature's status, someone has to open the repository and look through folders by hand.
• Management cannot easily see adoption of the AIDLC process across projects.
How it works
Information flows one way: Co-work → Bitbucket → collector → dashboard.
1.  A user works in Co-work. They ask the AIDLC skills to plan a feature, write the SRS and SAD, create test cases or raise a pull request. This is unchanged.
2.  The skills save the output to Bitbucket. Each document goes into the repository's docs folder, in a folder per feature, and is pushed. This also happens already.
3.  A collector reads Bitbucket on a schedule. A small background service lists each repository's docs folder and its pull requests, using read-only access.
4.  The collector records what it finds. For each document it stores the type, feature, author, date and link in a small database.
5.  The dashboard shows the result. Users choose a repository and see the counts, trends and details in a web page.
Steps 1 and 2 exist today. Steps 3 to 5 are what we build.
What the dashboard shows
View
What the user sees
All repositories
Totals across every repository, side by side, to compare adoption
Repository summary
Counts of Work Plans, SRS, SAD, Test Cases and Pull Requests for the chosen period
Trend
Documents created per week, by type
Documents
Every file in docs, with filters by type, person, week and search
Features
Which documents each feature has and which are still missing
Pull requests
Pull requests raised by the AIDLC flow, open or merged
What we need
Three things to build:
• Collector: a small background service that reads Bitbucket through its API.
• Database: one small store for the document records.
• Dashboard: the web page, based on the approved mock.
Three things from other teams:
• Bitbucket access: a read-only service account for the repositories in scope.
• Hosting: somewhere internal to run the collector, database and dashboard.
• Repository list: which projects are included in the first release.
Nothing is needed from the Co-work platform or the owners of the skills.
Rollout in phases
Phase
What happens
Result
1. Mock (done)
Clickable design with sample data
Agreement on what the dashboard should show
2. Pilot
Collector and dashboard connected to one or two real repositories
Real numbers, checked against the repositories by hand
3. Rollout
All repositories in scope added, with login for users
Dashboard available to teams and management
4. Later options
Document preview, alerts for features with missing documents, export of reports
Decided after feedback from phase 3
Effort and dates for phases 2 and 3 are not estimated yet; they depend on Bitbucket access and hosting.
Benefits, limits and open decisions
Benefits
• One place to see AIDLC output and adoption across projects.
• Gaps are visible early, for example a feature with an SRS but no test cases.
• No change to how teams work, and no dependency on the Co-work platform team.
• Documents created before the dashboard existed are counted too, because they are already in Bitbucket.
Limits
• Only documents pushed to Bitbucket are counted. Anything left in a Co-work chat is not visible.
• The dashboard shows who last changed a file, which may not be the person who ran the skill.
• Numbers are as fresh as the last collector run, not live.
Decisions needed
[ ] Which repositories are in the pilot?
[ ] Is a document that was updated several times counted once, or once per revision?
[ ] Who may see the dashboard: everyone, or project members only?
[ ] How often should the collector run: hourly or daily?
