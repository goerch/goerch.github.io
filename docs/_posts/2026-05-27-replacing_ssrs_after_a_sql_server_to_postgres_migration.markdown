# Replacing SSRS After a SQL Server to PostgreSQL Migration

## Context

We inherited a legacy project. The database was SQL Server, the reporting layer was SQL Server Reporting Services (SSRS), and the project budget was comparatively limited. Our task: migrate the database to PostgreSQL and find a workable replacement for a handful of SSRS reports — mostly pivot tables over four data marts, with drilldown capability — serving a small group of internal business users.

This post reflects on the choices we made, the ones we didn't, and what we learned along the way.

---

## The Landscape: Why SSRS Replacement Is Non-Trivial

SSRS has always been an acquired taste. It is not particularly friendly to business users, self-service analytics is not its strength, and its architecture is firmly rooted in the Microsoft ecosystem. But it does one thing very well: pixel-perfect, parameterised, paginated reports — including pivot tables with hierarchical drilldown that most modern BI tools struggle to replicate cleanly.

When you leave that ecosystem, you quickly discover that "replace SSRS" is not a single well-defined problem. It depends entirely on what you were actually using it for.

For organisations staying within the Microsoft world, the natural path is **Power BI Report Server (PBIRS)** — the on-premise successor that Microsoft now ships in place of SSRS, which was notably [absent from SQL Server 2025](https://techradar.info/what-is-the-replacement-for-ssrs-modern-alternatives-for-2026/). PBIRS supports the same RDL report format as SSRS, meaning many existing reports can be migrated without a complete rewrite. For cloud-first teams, **Power BI paginated reports** in the Power BI Service is the equivalent cloud-native path.

However, once you have moved to PostgreSQL, Power BI becomes considerably less straightforward. PostgreSQL is [not a natively supported embedded data source for Power BI paginated reports](https://community.fabric.microsoft.com/t5/Desktop/Power-BI-Report-Builder-PostgreSQL/td-p/2519632) — only Azure SQL Database and Azure Synapse Analytics are supported out of the box. Connecting to Postgres requires routing through ODBC or a data gateway, and [that gateway only runs on Windows — not in a container, requiring a dedicated Windows server or VM](https://medium.com/@leadvic/postgresql-on-premise-and-powerbi-a-sad-story-that-tells-you-everything-you-need-to-know-before-9c0000bb3d11). For a team that has just moved away from the Microsoft stack, this is a significant and often underdiscussed friction point. Additionally, [Power BI Pro pricing increased 40% in April 2025](https://blog.5000fish.com/ssrs-alternatives), which is relevant context for any long-term cost assessment.

Other commonly mentioned alternatives include **Pentaho** (with a long history in the data warehouse space but a complicated open-source/enterprise split), **Jaspersoft**, and **Bold Reports** (which supports RDL import). For teams building from scratch on open-source tooling, **Apache Superset** and **Metabase** are the most actively maintained options.

---

## The Architectural Opportunity: Introducing a Proper OLTP/OLAP Split

Our predecessors had not separated operational and analytical concerns. Everything lived together in SQL Server. The migration to PostgreSQL gave us a natural opportunity to fix this — and we took it.

We introduced **dbt** (data build tool) as the transformation layer between the operational database and the reporting marts. This was not a deeply deliberated decision at the time — dbt was an obvious choice given the team's familiarity — but it proved sound. The four data marts that feed the reports are now explicitly modelled, versioned, and tested as dbt models, rather than being implicit artefacts of ad-hoc queries.

One honest note: [SQLMesh](https://sqlmesh.com) has emerged as a compelling alternative to dbt, with stronger support for incremental models and a different philosophy around state management. We would want to evaluate it seriously on future projects, though dbt continues to serve us well here.

The result of this separation is that the reporting layer now queries clean, purpose-built marts rather than the operational tables directly. This is the right architecture regardless of which reporting tool sits on top.

---

## Finding a Reporting Tool: The Pivot Table Problem

The central requirement was pivot table support with drilldown. This rules out a large portion of the BI tool landscape, or at least makes it uncomfortable.

**Apache Superset** has a Pivot Table V2 that supports row/column grouping and metric aggregation. However, it has real gaps: [collapsible hierarchical drilldown — the expand/collapse behaviour familiar from Excel and SSRS — was still being requested by users in 2024 with no native solution](https://github.com/apache/superset/discussions/27555), and [pivot table Excel export only landed in July 2025](https://preset.io/blog/superset-repo-update-july-2025/). Superset is a powerful and actively developed platform, but its pivot story was clearly less mature at the time of our evaluation.

**Metabase** has stronger pivot table support and a more polished user experience for non-technical users. We ran a short evaluation — a day or two — but did not reach a firm conclusion in that time. Metabase remains a serious contender and is our identified upgrade path if requirements grow.

**Rill** ([rilldata.com](https://www.rilldata.com)) was the other platform we evaluated seriously. It is a younger and more opinionated tool, built around fast operational dashboards and exploratory analytics, with a developer-first workflow that uses YAML and SQL configuration files rather than a GUI. We had prior positive experience using dbt and Rill together on an [R&D project](https://github.com/idesis-gmbh/GitHubExperiments/blob/master/docs/blog2/README.md), which informed our confidence here.

---

## How We Validated the Decision: Two Unconventional Methods

Faced with cost pressure and no external guidance, we needed a pragmatic way to validate our tooling choice without spending weeks on formal evaluation. We used two complementary approaches.

**1. Codebase size as a cognitive load proxy**

We used [scc](https://github.com/boyter/scc) — a fast, accurate code counter — to compare the size of the Metabase, Superset, and Rill codebases. The reasoning: a smaller, more focused codebase means a shorter path to understanding the tool, debugging unexpected behaviour, and contributing fixes. For a team without a dedicated BI engineer and with a depleted budget, this is arguably more operationally relevant than a feature matrix.

Rill is significantly smaller and more focused than either Metabase or Superset, which are both large, general-purpose platforms.

**2. Attempting to fix a real bug**

We went further than reading the code — we attempted to [fix an actual open Rill issue](https://github.com/rilldata/rill/issues/9489). This approach tells you things no feature comparison can: How approachable is the codebase? How does the team respond to outside contributors? What is the actual code quality day-to-day?

This kind of due diligence is almost never mentioned in migration write-ups, but it is one of the most honest signals available for evaluating an open-source dependency.

---

## What We Learned About Rill — Including the Caveats

Our evaluation surfaced a few things worth noting for anyone considering Rill:

- **Agentic coding in production**: Parts of the Rill codebase appear to involve AI-assisted development. This is new and relatively uncharted territory for an open-source project, and raises questions about long-term code consistency and maintainability.
- **Distributed development**: Development work appears to be partially distributed internationally. Not a disqualifying factor, but relevant context when assessing contributor dynamics and response times.
- **No on-premise enterprise offering**: Rill's hosted, multi-user product is cloud-only. For customers with GDPR constraints, data residency requirements, or simply a reluctance to place business-critical data in a third-party cloud, this is a potential hard blocker.

The third point is the most significant for our situation. Our customer would likely be uncomfortable with a cloud-only reporting solution for sensitive business data.

---

## Our Decision: `rill developer`, On-Premise

The resolution is pragmatic: we are deploying **`rill developer`** — Rill's local, single-user mode — on-premise, without IAM. This sidesteps the cloud dependency entirely for now. The business user gets a working, capable reporting environment. The team gets a tool with a manageable codebase and a developer-friendly workflow.

We have explicitly scoped any future requirement for multi-user access, SSO, or enterprise permissions as a **change request for the customer to fund**. At that point, Metabase becomes the most natural upgrade path — it is a more mature platform for multi-user deployments and has the pivot table capability our customer needs.

This is not a perfect solution. It is a well-reasoned, honest one.

---

## Summary

| Concern | Our Approach |
|---|---|
| OLTP/OLAP separation | Introduced dbt transformation layer |
| Reporting tool selection | Rill developer (on-premise, no IAM) |
| Pivot table requirement | Met by Rill; Metabase as upgrade path |
| Cost pressure | Open-source tooling throughout |
| Validation method | scc codebase comparison + bug fix attempt |
| Future migration | Metabase, scoped as customer CR |

The broader lesson: "replace SSRS" is not a product decision — it is an architectural decision. The reporting tool is the last piece. Getting the data layer right first (OLTP/OLAP separation, clean marts, dbt) means the reporting tool can be swapped without rebuilding the foundation.

---

*Prior work using dbt and Rill for R&D analysis is documented [here](https://github.com/idesis-gmbh/GitHubExperiments/blob/master/docs/blog2/README.md).*