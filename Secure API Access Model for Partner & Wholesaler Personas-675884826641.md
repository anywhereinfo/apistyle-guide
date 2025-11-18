# Secure API Access Model for Partner & Wholesaler Personas

> Confluence Page ID: 675884826641, Version: 2

none

User Story:
-----------

GNTSARCH-516f6b00489-ed93-3bc3-8783-4866a55cbebeSystem Jira

As an API Product Manager,

I want a secure, scalable, and standardized API security model for our Partner and Wholesaler personas,

So that they can securely access their end-customers' data to place orders and manage resources without forcing customers to share their API credentials.

Problem Statement / Business Context
------------------------------------

The current "SELL" strategy for digital services has identified three key personas, but our security implementation has critical gaps:

* **Lumen Customer (Enterprise):** [SOLVED] This is the standard Enterprise Customer model.
* **Wholesaler:** [PARTIAL GAP] The model for accessing their *own* resources vs. their *customers'* resources is unclear, especially regarding billing identification.
* **Partner:** [CRITICAL GAP] The desired model is "access to lumen customer resources. Lumen Customer will be billed directly." The current "solution" is an insecure workaround that requires the end-customer to share their API credentials or use a complex portal login. This is a major security risk and a significant point of business friction.

Goal
----

This story's goal is to **design the technical solution** to close these gaps, focusing on providing a formal, delegated access model for the Partner persona and clarifying the Wholesaler model.

Acceptance Criteria (AC)
------------------------

1. **[Design]** A formal **Capability Solution Document** is created that details the proposed security architecture.
2. **[Partner Flow]** The design MUST define a secure, 3-legged OAuth flow (e.g., Authorization Code Grant or a similar delegated authority pattern) for the Partner persona.
3. **[No Credential Sharing]** The solution MUST explicitly eliminate the need for end-customers to share their API credentials with Partners.
4. **[Wholesaler Flow]** The design MUST clarify the access and billing model for the Wholesaler persona, defining how they manage their own resources vs. their end-customer's resources.
5. **[Token Claims]** The resulting LIAM token MUST contain distinct, verifiable claims for the authenticated party (the Partner/Wholesaler) **and** the end-customer they are acting on behalf of (e.g., `partner_id`, `customer_ban`, `scope`).
6. **[Gateway Enforcement]** The API Gateway reference architecture is updated to demonstrate how to enforce this new, delegated authority model (i.e., validating claims for both the Partner and the Customer).
7. **[Onboarding Flow]** The solution must document the new "Partner Onboarding" and "Customer Consent" user experience (e.g., how a customer grants a partner permission).