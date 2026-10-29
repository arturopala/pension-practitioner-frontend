# Pension Practitioner Frontend – Summary

## Purpose

This is the citizen-facing web front end for **Pension Scheme Practitioners (PSPs)**. A PSP is a person or organisation that a pension scheme administrator authorises to manage a registered pension scheme on their behalf. The service lets a Government Gateway **organisation** account:

- register as a PSP, as a UK company, a UK partnership or a UK-resident individual;
- view and amend its registered contact details after registration;
- stop being a PSP (deregister).

It is a server-rendered GOV.UK-style UI. Every page is HTML and every question is a form post. It holds no data of its own. Journey answers are kept in a remote cache owned by the pension-practitioner back end, and all registration, subscription and deregistration work is delegated to that back end, which in turn talks to HMRC's head-of-duty systems. The service is reached from, and returns users to, the Manage Pension Schemes dashboard.

## Main Capabilities

### 1. Entry, eligibility and routing

- The entry point sends the user to a "who are you registering" choice: a company or partnership, or yourself as an individual. Choosing starts the journey and records a "start" audit event.
- Only organisation accounts can use the service:
  - **agent** accounts go to a "cannot register" page;
  - **individual** accounts go to "you need an organisation account";
  - **assistant** users without a PSP enrolment go to a "no access" page.
- Registration pages refuse users who already hold the PSP enrolment and send them to their PSP dashboard. Post-registration pages (view and amend details, deregister) require that enrolment and send everyone else to "you need to register".
- Users who hold **both** the administrator and the practitioner enrolments must first choose which role they are acting in. If they last chose "administrator", they are sent to a "cannot access as administrator" page.
- For an enrolled practitioner, the service checks on every guarded request whether HMRC has flagged the practitioner as **deceased**. If so, the user is sent to a "you must contact HMRC" page.
- Only **UK** organisations and **UK-resident** individuals can complete registration online. Non-UK answers lead to a stop page. An individual whose looked-up address is outside the EU/EEA also gets a stop page.

### 2. Registration – UK company or partnership

- The user picks the business type (limited company, unlimited company, business partnership, limited partnership, LLP) and enters the 10-digit UTR and the registered name.
- The service matches the organisation against HMRC records using the UTR, name and organisation type, and shows the matched name and registered address for confirmation.
  - **No match** leads to a "business details not found" page.
  - If the user says the name or address is wrong, they are told to update Companies House or HMRC first.
- The contact address is either the registered address or a different one, found by UK postcode lookup or entered manually. Contact email and phone are captured, followed by a check-your-answers page and a declaration.

### 3. Registration – UK individual

- Requires identity verification at confidence level 250 from the "is this you?" step onwards. The opening pages ("what you will need", "are you UK resident") and the address-results page do not enforce it.
- The individual is matched against HMRC records using the NINO from their authenticated identity, and asked "is this you?" with the name and address on record.
  - Answering "no" leads to a "tell HMRC about your changes" page.
- Contact address, email and phone are captured as in the organisation journey.

### 4. Subscription and enrolment (declaration)

- On declaration, the service submits the whole set of journey answers to the back end as a **PSP subscription** and receives a new **PSP ID**.
- It then enrols the user's group for the PSP service. The enrolment uses the PSP ID plus one known fact (NINO, Corporation Tax UTR, Self Assessment UTR, or country code for non-UK registrants) and is retried with back-off on upstream errors.
- A confirmation email is sent. Its delivery-status callback is directed to the back end, carrying the email address and PSP ID encrypted.
- Failure outcomes:
  - an upstream 5xx leads to "your action was not processed";
  - an "already an active PSP" rejection leads to "cannot register as practitioner";
  - **any other failure leads to the session-expired page**.

### 5. View and amend details

- An enrolled PSP can view their registered details (PSP ID, UTR where relevant, name, contact address, email, phone). These are loaded from the back end on first visit, and a snapshot of the originals is kept.
- Contact address, email and phone can be changed. Name can be changed only for non-UK registrants.
- A declaration is offered only when an amendment actually differs from the snapshot. Submitting it sends an **amendment** subscription, an extended audit event with the before/after differences, and a confirmation email.

### 6. Deregistration (stop being a PSP)

- First the service checks with the back end whether the practitioner may deregister. A practitioner still authorised on schemes is sent to a "deauthorise from schemes first" page. A 404 from that check counts as "may deregister".
- After confirmation, the user enters a deregistration date. It cannot be in the future or before the PSP's original application date.
- The service then deregisters with the back end (reason code 1), records a deregistration audit, removes the group's PSP enrolment (retried with back-off) and sends a confirmation email, which is also audited.

### 7. Journey state and session

- Answers are stored after every page in the back end's journey cache. The cache is cleared on confirmation pages and on sign-out.
- Sign-out also clears the administrator service's session-data cache (role choice), then redirects to the feedback/sign-out page.
- A keep-alive endpoint and a language switch are provided. English is the only configured language.

## Cross-cutting Concerns

| Concern | Behaviour |
|---|---|
| Authentication | Government Gateway via the platform auth service. Credential role must be User or Assistant. Affinity group decides eligibility. The no-session redirect goes to login, with a return to the service entry. |
| Authorisation | Five guard flavours combine three rules: identity verification at L250 (on/off), PSP enrolment (required, forbidden, or no rule), and the administrator-or-practitioner role choice for dual-enrolled users. The deceased flag is checked on every guarded request by enrolled users. |
| Identity verification | Insufficient confidence redirects to the IV uplift journey, with completion and failure return URLs restricted to relative URLs. |
| Auditing | Start of journey, enrolment success/failure, amendment (extended, with differences), deregistration, de-enrolment, deregistration email. |
| Validation | Server-side form validation with GOV.UK error summaries: UK postcode, 10-digit UTR, address-line character set and length, email, phone, date range. Names are only length-checked. |
| Data protection | No local persistence. The email callback URL carries the PSP ID and email encrypted with the platform query-parameter crypto. Journey data lives in the back-end cache. |
| Feature toggles | No business toggles. `email.force` is passed through to the email service. `play-frontend-hmrc.forceServiceNavigation` and `useRebrand` affect layout only. |

## Dependencies (generic)

| Kind | Role |
|---|---|
| Pension practitioner back end | Journey-answer cache; registration (with and without ID); PSP subscription and amendment; subscription details; minimal details (name, email, deceased and RLS flags); can-deregister; deregister; email-status callback receiver |
| Pension administrator back end | Session-data cache holding the dual-enrolment role choice |
| Platform auth | Identity, enrolments, confidence level, NINO, group identifier |
| Tax enrolments | Allocate and remove the PSP enrolment for the user's group |
| Address lookup | UK postcode-to-address search |
| Email | Transactional emails with a delivery-status callback |
| Audit (platform) | Explicit audit events |
| Identity verification frontend | Confidence uplift for individuals |
| Manage Pension Schemes frontend | Dashboards, role chooser, "you need to register" and "contact HMRC" pages |

## In One Sentence

A GOV.UK web journey that lets UK organisations and individuals register as Pension Scheme Practitioners, keep their contact details up to date and deregister, with all records held by the pension-practitioner back end and the enrolment managed through tax-enrolments.
