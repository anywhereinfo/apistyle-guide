# API Publishing Playbook: Public vs. Internal Deployment Workflow

> Confluence Page ID: 675869589623, Version: 1

Purpose
-------

This playbook defines the mandatory governance process for all APIs, using the `lifecycleStage` metadata field to track an API's authoritative status. The `main` **branch** serves as the definitive **"Approved for Publish"** source for both Public and Internal APIs.

---

Stage 1: Technical Governance Approval (The Baseline)
-----------------------------------------------------

This stage establishes and validates the structural contract of the API and is required for **ALL** APIs. An automated process sets up the feature branch and domain folder structure (e.g., `/domains/<project>`).

1. **Developer Setup & Submission**

   * Developer populates the OAS and the `api-metadata.yaml` files within the pre-created feature branch.
   * Developer opens a PR from the feature branch, targeting the protected `governance-approved` branch.
   * **Lifecycle State Transition:** **Design** --> **Development**
2. **Pipeline Validation & Review**

   * The Pipeline is triggered to run technical validation against **Lumen Standards** on the OAS and metadata. Architectural review takes place.
   * **Lifecycle State:** Stays at **Development**
3. **Merge to** `governance-approved`

   * The PR is approved by the API Architect and merged.
   * **Lifecycle State Transition:** **Development** --> **Governance-Approved** (Contract is now locked as technically compliant.)

---

Publishing Path Fork
--------------------

Once the contract is technically approved (state: **Governance-Approved**), the workflow splits based on the API's designated audience.

### Path A: Public API Publishing (Two-Stage Review)

Public APIs require a second PR for human review of external-facing documentation quality.

4. **Documentation & Metadata Review (Stage 2)**

   * A new PR is created from `governance-approved`, targeting the `main` branch.
   * The Technical Writer reviews, edits the narrative, and modifies **only non-contract** OAS fields (e.g., summary, description, and examples). The Pipeline is triggered to run technical validation against **Lumen Standards** on the OAS and metadata.
   * **Lifecycle State:** Stays at **Governance-Approved**
5. **Merge to** `main` **& Publish**

   * The PR is merged into the `main` **branch**. The Public Publishing Pipeline is triggered.
   * **Tooling:** Pushes content to **Amazon S3/CloudFront** at `developer.lumen.com`.
   * **Lifecycle State Transition:** **Governance-Approved** -→ **Published**

### Path B: Internal API Publishing (Single-Stage Direct Merge)

Internal APIs skip the human documentation review and use automation to achieve the final state immediately after technical approval.

4. **Automated Merge Trigger**

   * Automation detects the successful merge into the `governance-approved` branch.
   * **Lifecycle State:** Stays at **Governance-Approved**
5. **Merge to** `main` **& Publish**

   * Automation performs a fast-forward merge from `governance-approved` to the `main` **branch**. The Internal Publishing Pipeline is triggered immediately.
   * **Tooling:** Syncs OAS to the **SwaggerHub** internal workspace.
   * **Lifecycle State Transition:** **Governance-Approved** --> **Published**

---

Post-Publishing Lifecycle
-------------------------

|  |  |
| --- | --- |
| **Future Action** | **Lifecycle State Transition** |
| **Deprecation Decision** | **Published** --> **Deprecated** |
| **API Shutdown** | **Deprecated** --> **Retired** |