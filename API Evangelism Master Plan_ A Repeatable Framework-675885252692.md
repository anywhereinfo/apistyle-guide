# API Evangelism Master Plan: A Repeatable Framework

> Confluence Page ID: 675885252692, Version: 3

none

**Purpose**
-----------

To create a realistic, repeatable, and effective framework for driving the adoption of the new API Strategy. This plan is designed to be piloted with a key department and then scaled to the wider organization.

Core Message (The "Message House"):

We are launching a new, unified API framework. Everything we do anchors to these three pillars:

1. **Predictable APIs:** (Via the new Style Guide)
2. **Automated Governance:** (Via the new Publishing Playbook)
3. **Faster Delivery:** (By automating the process from idea to production)

Phase 0: Prerequisite Readiness (The "Go/No-Go" Gate)
-----------------------------------------------------

**Goal:** To finalize all assets and solve technical blockers *before* the first announcement. This phase is owned by the API Governance team.

| Action Item | Owner | Status | Details |
| --- | --- | --- | --- |
| **Define & Approve Developer Tooling (POC)** | Evangelism Lead (Maninder), API Platform Engineer | **Blocked** | **CRITICAL:** Current tool (SwaggerHub) does not support Spectral linting at design-time. Need POC + Legal/Licensing review for a VS Code + Spectral + 42 Crunch-like environment. |
| **Finalize Core Assets** | API Governance Team | **Blocked** | The **API Style Guide** , Governance Process, Centralized Repo Design, Developer workflow for authoring and working with governance workflow and the **API** Publishing Playbook (Public **vs. Internal)** must be 100% final and published on Confluence. |
| **Create Evangelism Toolkit** | Evangelism Lead (Maninder), Ravi | To-Do | Create the "master" PowerPoint deck. This deck *must* include the "Why" (Strategy), the "What" (Style Guide), and the "How" (a visual of the new Governance Process). |
| **Solve "Confluence Blocker"** | Evangelism Lead (Maninder) | To-Do | 1. Get the official "How to get a Confluence license" link.      2. Create a SharePoint site/folder.      3. Upload PDF versions of the Style Guide and Playbook to SharePoint. |
| **Create "Intake Form"** | Evangelism Lead (Maninder) | To-Do | Create the official Jira Form for teams to request "White Glove" support, as mentioned by management. |
| **Create FAQ / Objection Handling Doc** | Evangelism Lead (Maninder) | To-Do | Preemptively answer objections ("Will this slow us down?", "Why is this required for internal APIs?") to reduce friction. |

Phase 1: Pilot Launch (Target: Oscar's Department)
--------------------------------------------------

**Goal:** To test, refine, and prove our evangelism model with a friendly, high-context audience.

|  |  |  |  |
| --- | --- | --- | --- |
| **Step** | **Channel** | **Target Audience** | **Key Message & "Ask"** |
| 1. **The Announcement** | Mailing List | All (Engineers, PMs, Leaders) | "Introducing the new API Governance process, built on 3 pillars: Predictable APIs, Automated Governance, & Faster Delivery. We're piloting it with you."      **Links:** Confluence (w/ license link) & SharePoint (PDFs). |
| 2. **Developer Deep Dive** | Engineering Garage / Community of Practice (CoP) | Engineers / Developers | **"Live Demo: How to Publish Your API in 2026."**      We will walk through the *entire* `feature-branch` -->`governance-approved` --> `main` workflow to show how the pipeline enforces the new standards. |
| 3. **Product Manager Sync** | PM Forum / Scheduled Meeting | Product Managers | **"How the New Governance Process Accelerates Your Roadmap."**      We will focus on how this process enables the Partner/Wholesale models and reduces manual review time. |
| 4. **"White Glove" Pilot** | Direct Engagement | 2-3 Selected Projects | We will offer hands-on support for 2-3 key projects in this department. This is our chance to be embedded, find gaps in our process, and get our first "wins." |
| 5. **Pilot Retro & Success Story** | Internal Meeting | API Governance Team + Pilot Team Leads | 1. "What worked? What was confusing? What's missing in the docs?"      2. **Create a 1-slide "Success Story"** with a "Before & After" and a quote from the dev lead. Nothing accelerates adoption like early wins. |

Phase 2: General Rollout (Target: All PAX/STEPN)
------------------------------------------------

**Goal:** To scale the successful, refined pilot program to the entire engineering and product organization.

|  |  |  |  |
| --- | --- | --- | --- |
| **Step** | **Channel** | **Target Audience** | **Details** |
| 1. **Public Announcement** | MyEvive / SharePoint Blog Post | All Engineering & Product | Take the successful "Mailing Letter" (from Phase 1), refine it with feedback, and post it as the official "Big Bang" announcement. **Feature the "Success Story"** from the pilot. |
| 2. **Company-Wide Demos** | Brown Bags / Lunch & Learns (Scheduled) | All Engineers / Developers | Take the successful "Live Demo" (from Phase 1) and offer it as a recurring, scheduled workshop for all. |
| 3. **Open "White Glove" Queue** | Jira Intake Form | All Teams | Formally open the "White Glove" support queue (created in Phase 0) to all teams, with clear SLAs based on what we learned in the pilot. |

Phase 3: Sustain & Measure (The "Repeatable Framework")
-------------------------------------------------------

**Goal:** To ensure this effort has a lasting impact and to prove its value to leadership.

|  |  |  |  |
| --- | --- | --- | --- |
| **Activity** | **Frequency** | **Owner** | **Metrics for Success (Leadership KPIs)** |
| **Community of Practice (CoP)** | Monthly | API Governance Team | # of attendees; # of questions/topics submitted. |
| **API Evangelism Blog** | Monthly | Evangelism Lead (Maninder) | Post views; engagement; new topics from the "post ideas" list. |
| **White Glove Support** | Ongoing | API Architects | **% of new APIs conformant without human involvement**; queue backlog. |
| **Leadership Report** | Quarterly | Evangelism Lead (Maninder) | **Process Adoption:** Time from idea -> `governance-approved` OAS.    **Efficiency:** Reduction in manual review time.    **Discoverability:** % of APIs discoverable in the central catalog.    **Consolidation:** Reduction in API duplication. |