# Pension Practitioner Frontend – HTTP API

[`openapi.json`](openapi.json) describes the browser-facing HTTP surface of this frontend: **168 operations on 102 paths**.

It was generated with the service-docs generator from `conf/prod.routes` and its included routers (`app`, `individual`, `company`, `partnership`, `deregister`). Each route was annotated with facts read from its controller: auth guard, form mapping and side effects. Four library routes were added by hand: bootstrap `health.Routes` (`/ping/ping`) and play-frontend-hmrc `hmrcfrontend.Routes`. The test-only router (`testOnlyDoNotUseInAppConf.routes`) is **not** part of the production router and is not in the spec.

Every endpoint returns HTML (or a redirect). There is no JSON API.

## Conventions that apply to every endpoint

| Aspect | Convention |
|---|---|
| Authentication | Government Gateway session (MDTP session cookie), checked through the auth service by one of the five guards (see below). Unauthenticated → **303** to login with `continue` set to the service entry. Static/info pages, assets, keep-alive, language switch and the entry redirect are public |
| Request bodies | POSTs are HTML forms (`application/x-www-form-urlencoded`) bound with Play `Form` mappings. Every POST needs a valid `csrfToken` (Play CSRF filter) → **403** otherwise. Body size is limited by Play's default parser settings (not overridden) |
| Parameter binding | No typed path or query parameters in production routes. `mode` is fixed per route (`NormalMode` for the plain URL, `CheckMode` for `change-…` URLs), so nothing fails binding with 400 |
| Responses | **200** HTML page. **303** redirect (Post/Redirect/Get after a successful post, journey navigation, or an auth/eligibility redirect). **400** form re-rendered with errors |
| Journey state | Guarded pages load the journey answers from the back-end cache. Most also require them: when they are absent → **303** to `/this-service-has-been-reset` |
| Errors | Unknown path → 404 page. Unhandled exceptions (upstream errors, JSON parse failures, tampered list index…) → **500** internal-server-error page with `Cache-Control: no-cache`. Registration declarations convert most failures to redirects (see validation.md) |
| Auditing | Explicit audit events on: start of registration, enrolment, amendment, deregistration, de-enrolment, deregistration email. Bootstrap's implicit request auditing is also active |

## Access-controlled endpoints

| Endpoints | Check | Failure (status and target) |
|---|---|---|
| All guarded endpoints | `authorised(User or Assistant)`; Organisation affinity | No session → 303 login. Agent → 303 `/cannot-register`. Individual account → 303 `/need-an-organisation-account`. Assistant without PSP enrolment → 303 `/do-not-have-access`. Missing retrievals / other auth errors → 303 `/unauthorised` |
| Guarded, user holds PSA **and** PSP enrolments | Administrator-session role choice | No choice → 303 MPS role chooser. "Administrator" → 303 MPS "cannot access as administrator" |
| Guarded, user holds the PSP enrolment | Minimal details `deceasedFlag` | 303 MPS "you must contact HMRC". Upstream failure → 500 |
| Registration triage, company/partnership UTR, name confirmation, UK-registration question, check-your-answers, declarations, `who-are-you-registering`, `tell-hmrc`, individual "what you will need" / UK resident / non-UK | Must **not** hold `HMRC-PODSPP-ORG` | 303 PSP dashboard |
| Individual "is this you", "tell HMRC", "use address for contact", CYA, declaration, confirmation, outside EU/EEA | No PSP enrolment **and** confidence level 250 | 303 IV uplift (`SeeOther`), or PSP dashboard |
| Individual email, phone, postcode, manual address | Confidence level 250 | 303 IV uplift |
| `/practitioner-details`, `/declare`, `/success`, `/update-contact-address-psp`, `/remove-psp/remove`, `/remove-psp/remove-company`, `/remove-psp/date-removed-company` | **Must** hold `HMRC-PODSPP-ORG` | 303 MPS "you need to register" |
| Company/partnership address, email, phone, name pages; confirm-address; confirmation pages; `/remove-psp/date-removed`; success pages; individual address list; `/your-action-was-not-processed` | Base guard only (no enrolment rule) | – |

⚠️ **Observation:** the change pages for address, email, phone and name are shared by registration and amendment, so they carry no enrolment rule. `POST /remove-psp/date-removed` (individual deregistration) uses the base guard while calling `pspIdOrException`, so a non-enrolled user gets a 500 rather than a redirect.

## Back-end calls by endpoint

| Endpoint(s) | Back-end calls (besides the journey-cache GET/POST) |
|---|---|
| `GET …/confirm-address` (company, partnership) | pension-practitioner `POST /register-with-id/organisation` |
| `GET /register/individual/is-this-you`, `/change/is-this-you` | pension-practitioner `POST /register-with-id/individual` (unless already cached for this NINO) |
| `POST …/enter-registered-address` (company, partnership) | pension-practitioner `POST /register-with-no-id/organisation`, only when the next page is not the "registered in UK?" question |
| `POST …/postcode*` | address-lookup `POST /lookup` |
| `POST …/declaration` (company, partnership, individual) | `POST /subscribePsp/PSPSubscription` → tax-enrolments `PUT /service/HMRC-PODSPP-ORG/enrolment` → email `POST /hmrc/email` |
| `GET /practitioner-details`, `/update-contact-address-psp` | `GET /getPsp-self` (first visit), `GET /get-minimal-details-self` |
| `POST /declare` | `GET /getPsp-self` (if no snapshot) → `POST /subscribePsp/PSPAmendment` → email |
| `GET /remove-psp/remove`, `/remove-company` | `GET /can-deregister-self`, `GET /get-minimal-details-self` |
| `GET`/`POST /remove-psp/date-removed*` | `GET /getPsp-self` (application date). POST also: `POST /deregisterPsp-self` → tax-enrolments `DELETE /groups/{groupId}/enrolments/HMRC-PODSPP-ORG~PSPID~{pspId}` → email |
| Confirmation pages (`registration-successful`, `/success`) | journey-cache `DELETE` |
| `GET /sign-out` | pension-administrator `DELETE session-data-self`, journey-cache `DELETE` |

## Endpoints with non-standard behaviour

- **`POST /register/individual/use-address-for-contact`** is routed to `onPageLoad`, so the submitted form is ignored and the page re-renders with 200. The view's form posts to `/register/individual/change/use-address-for-contact` (`CheckMode` route) even in the normal journey.
- **Route shadowing:** `/register/individual/you-need-to-tell-hmrc` is matched by the `individual.Routes` include (declared first) before the generic `/register/{entityType}/you-need-to-tell-hmrc`. `TellHMRCController` therefore serves only `company` and `partnership` in practice. `entityType` is otherwise unvalidated and used as a message key.
- **GETs with side effects:**
  - The confirm-address and is-this-you pages call register-with-id and write to the cache on every page load.
  - Confirmation pages clear the journey cache, so a refresh lands on "service reset".
  - `GET /remove-psp/remove` and `/remove-company` **replace** the whole journey cache with a new answers object holding only name and email.
- **`GET /language/{lang}`** redirects to the request's `Referer` header unchanged (open redirect). The fallback is the PSP dashboard.
- **`GET /sign-out`** works with or without a session (`MissingBearerToken` is recovered), and always ends the session.
- **`GET /keep-alive`** returns plain text `OK`. The library route `/hmrc-frontend/keep-alive` exists too.
- **`GET /`** is a plain redirect to `/register/who-are-you-registering`, with no auth.
- **Declarations** (`POST …/declaration`, `POST /declare`) bind no form fields, so 400 cannot occur. In the registration declarations, most failures become **303** to "service reset" rather than an error status.
- **`POST …/address-results*`** trusts the posted index; an out-of-range value causes a 500 (theoretically reachable only through a tampered form).
- **IV uplift** uses `SeeOther` to `urls.iv-uplift-entry`, with relative completion and failure URLs.

## Endpoint catalogue


### Entry and start

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/` | Service entry: redirects to the 'who are you registering' page | — |
| GET | `/pension-scheme-practitioner/register/who-are-you-registering` | Choose company/partnership or individual | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/who-are-you-registering` | Save registrant type, send PSPStartNew audit | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |

### Registration – organisation triage

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/register/what-you-will-need` | What you will need (organisation) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/are-you-in-the-uk-company` | Is the organisation UK-based? | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/are-you-in-the-uk-company` | Save answer | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/business-type` | Business type | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/business-type` | Save business type | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/business-details-not-found` | Business details not matched page | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/what-is-business-registered-as` | Company or partnership (non-UK path) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/what-is-business-registered-as` | Save registration type | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/non-uk-practitioner` | Non-UK organisations cannot register online (stop page) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |

### Registration – company

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/register/company/can-hmrc-use-this-address` | Can HMRC use this address for contact? | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/can-hmrc-use-this-address` | Save answer | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/postcode` | Contact address postcode | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/postcode` | Look up postcode (address-lookup) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/change-postcode` | Contact address postcode (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/change-postcode` | Look up postcode (address-lookup) (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/address-results` | Select address from lookup results | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/address-results` | Save selected address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/change-address-results` | Select address from lookup results (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/change-address-results` | Save selected address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/enter-contact-address` | Enter contact address manually | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/enter-contact-address` | Save contact address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/change-contact-address` | Enter contact address manually (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/change-contact-address` | Save contact address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/business-utr` | Company UTR | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/company/business-utr` | Save UTR | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/company/org-name` | Company name | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/org-name` | Save name (sets nameChange in variation) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/change-org-name` | Company name (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/change-org-name` | Save name (sets nameChange in variation) (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/confirm-name` | Confirm company name | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/company/confirm-name` | Save answer | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/company/confirm-address` | Confirm registered address – calls register-with-id (organisation) on page load | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/confirm-address` | Confirm/reject matched address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/contact-details/contact-email` | Contact email | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/contact-details/contact-email` | Save email | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/contact-details/change-contact-email` | Contact email (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/contact-details/change-contact-email` | Save email (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/contact-details/contact-phone` | Contact phone | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/contact-details/contact-phone` | Save phone | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/contact-details/change-contact-phone` | Contact phone (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/contact-details/change-contact-phone` | Save phone (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/check-your-answers` | Check your answers | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/company/declaration` | Declaration | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/company/declaration` | Subscribe PSP, enrol, send email | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/company/registration-successful` | Registration confirmation; clears journey cache | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/enter-registered-address` | Enter registered address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/enter-registered-address` | Save registered address (may call register-without-id) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/addresses/change-enter-registered-address` | Enter registered address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/company/addresses/change-enter-registered-address` | Save registered address (may call register-without-id) (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/company/is-company-in-uk` | Is the company registered in the UK? | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/company/is-company-in-uk` | Save answer | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |

### Registration – partnership

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/register/partnership/business-utr` | Partnership UTR | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/partnership/business-utr` | Save UTR | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/partnership/org-name` | Partnership name | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/org-name` | Save name (sets nameChange in variation) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/change-org-name` | Partnership name (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/change-org-name` | Save name (sets nameChange in variation) (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/confirm-name` | Confirm partnership name | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/partnership/confirm-name` | Save answer | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/partnership/confirm-address` | Confirm registered address – calls register-with-id (organisation) on page load | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/confirm-address` | Confirm/reject matched address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/can-hmrc-use-this-address` | Can HMRC use this address for contact? | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/can-hmrc-use-this-address` | Save answer | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/postcode` | Contact address postcode | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/postcode` | Look up postcode (address-lookup) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/change-postcode` | Contact address postcode (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/change-postcode` | Look up postcode (address-lookup) (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/address-results` | Select address from lookup results | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/address-results` | Save selected address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/change-address-results` | Select address from lookup results (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/change-address-results` | Save selected address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/enter-contact-address` | Enter contact address manually | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/enter-contact-address` | Save contact address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/change-contact-address` | Enter contact address manually (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/change-contact-address` | Save contact address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/contact-details/contact-email` | Contact email | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/contact-details/contact-email` | Save email | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/contact-details/change-contact-email` | Contact email (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/contact-details/change-contact-email` | Save email (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/contact-details/contact-phone` | Contact phone | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/contact-details/contact-phone` | Save phone | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/contact-details/change-contact-phone` | Contact phone (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/contact-details/change-contact-phone` | Save phone (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/check-your-answers` | Check your answers | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/partnership/declaration` | Declaration | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/partnership/declaration` | Subscribe PSP, enrol, send email | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/partnership/registration-successful` | Registration confirmation; clears journey cache | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/enter-registered-address` | Enter registered address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/enter-registered-address` | Save registered address (may call register-without-id) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/addresses/change-enter-registered-address` | Enter registered address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/partnership/addresses/change-enter-registered-address` | Save registered address (may call register-without-id) (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/partnership/is-partnership-in-uk` | Is the partnership registered in the UK? | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/partnership/is-partnership-in-uk` | Save answer | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |

### Registration – individual

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/register/individual/what-you-will-need` | What you will need (individual) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/are-you-uk-resident` | Are you UK resident? | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/are-you-uk-resident` | Save answer | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/change-are-you-uk-resident` | Are you UK resident? (check/change mode) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/change-are-you-uk-resident` | Save answer (check/change mode) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/is-this-you` | Is this you? – calls register-with-id (individual, NINO from auth) unless cached | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/is-this-you` | Save answer | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/change/is-this-you` | Is this you? – calls register-with-id (individual, NINO from auth) unless cached (check/change mode) | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/change/is-this-you` | Save answer (check/change mode) | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/you-need-to-tell-hmrc` | Tell HMRC about changed personal details | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/use-address-for-contact` | Use this address for contact? | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/use-address-for-contact` | Use this address for contact? | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/change/use-address-for-contact` | Use this address for contact? (check/change mode) | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/change/use-address-for-contact` | Save answer (check/change mode) | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/contact-details/contact-email` | Contact email (redirects unless UK resident) | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/contact-details/contact-email` | Save email | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/contact-details/change-contact-email` | Contact email (redirects unless UK resident) (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/contact-details/change-contact-email` | Save email (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/contact-details/contact-phone` | Contact phone | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/contact-details/contact-phone` | Save phone | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/contact-details/change-contact-phone` | Contact phone (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/contact-details/change-contact-phone` | Save phone (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/addresses/postcode-individual` | Contact address postcode | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/addresses/postcode-individual` | Look up postcode (address-lookup) | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/addresses/change-postcode-individual` | Contact address postcode (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/addresses/change-postcode-individual` | Look up postcode (address-lookup) (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/addresses/address-results-individual` | Select address from lookup results | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/individual/addresses/address-results-individual` | Save selected address | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/individual/addresses/change-address-results-individual` | Select address from lookup results (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/register/individual/addresses/change-address-results-individual` | Save selected address (check/change mode) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/register/individual/addresses/address-manual-individual` | Enter contact address manually | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/addresses/address-manual-individual` | Save contact address | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/addresses/change-address-manual-individual` | Enter contact address manually (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| POST | `/pension-scheme-practitioner/register/individual/addresses/change-address-manual-individual` | Save contact address (check/change mode) | Signed in; Organisation; confidence level 250 (IV uplift) |
| GET | `/pension-scheme-practitioner/register/individual/check-your-answers-individual` | Check your answers | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/declaration` | Declaration | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/register/individual/declaration` | Subscribe PSP, enrol, send email | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/registration-successful` | Registration confirmation; clears journey cache | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/outside-eu-eea` | Address outside EU/EEA stop page | Signed in; Organisation; confidence level 250; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/register/individual/non-uk-practitioner` | Non-UK individuals cannot register online (stop page) | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |

### Amend practitioner details

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/update-contact-address-psp` | Prompt to update contact address (loads subscription details) | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/practitioner-details` | View practitioner details (loads subscription and minimal details) | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/declare` | Amendment declaration (only if amendments exist) | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/declare` | Submit amendment, audit, send email | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/success` | Amendment confirmation; clears journey cache | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |

### Deregistration

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/remove-psp/remove-company` | Confirm stop being a practitioner (company) – checks can-deregister, loads minimal details | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/remove-psp/remove-company` | Save answer | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/remove-psp/date-removed-company` | Deregistration date (company) – loads application date | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/remove-psp/date-removed-company` | Deregister, audit, de-enrol, email | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/remove-psp/success-company` | Deregistration success (company) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/remove-psp/remove` | Confirm stop being a practitioner (individual) – checks can-deregister, loads minimal details | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| POST | `/pension-scheme-practitioner/remove-psp/remove` | Save answer | Signed in; Organisation; MUST hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/remove-psp/date-removed` | Deregistration date (individual) – loads application date | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| POST | `/pension-scheme-practitioner/remove-psp/date-removed` | Deregister, audit, de-enrol, email | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/remove-psp/success-individual` | Deregistration success (individual) | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |

### Session and language

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/sign-out` | Sign out: clears both journey caches and redirects to the feedback/sign-out URL | Optional: bearer token checked if present |
| GET | `/pension-scheme-practitioner/keep-alive` | Session keep-alive (plain `OK`) | — |
| GET | `/pension-scheme-practitioner/assets/{file}` | Static assets (versioned) | — |
| GET | `/pension-scheme-practitioner/language/{lang}` | Switch language and redirect to the Referer header (fallback: PSP dashboard) | — |

### Information and stop pages

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/pension-scheme-practitioner/remove-psp/deauthorise-from-schemes` | Cannot stop being a practitioner while authorised on schemes | — |
| GET | `/pension-scheme-practitioner/register/{entityType}/you-need-to-tell-hmrc` | Tell HMRC about changed company/partnership details | Signed in; Organisation; must NOT hold HMRC-PODSPP-ORG |
| GET | `/pension-scheme-practitioner/this-service-has-been-reset` | Session expired / service reset page | — |
| GET | `/pension-scheme-practitioner/unauthorised` | Unauthorised page | — |
| GET | `/pension-scheme-practitioner/cannot-register` | Agents cannot register page | — |
| GET | `/pension-scheme-practitioner/need-an-organisation-account` | Individual-account users need an organisation account | — |
| GET | `/pension-scheme-practitioner/do-not-have-access` | Assistant credential without enrolment page | — |
| GET | `/pension-scheme-practitioner/your-action-was-not-processed` | Upstream 5xx failure page | Signed in (User/Assistant); Organisation affinity only; no enrolment rule |
| GET | `/pension-scheme-practitioner/cannot-register-practitioner` | Already an active practitioner (ACTIVE_PSPID) page | — |

### Platform

| Method | Path | Purpose | Auth |
|---|---|---|---|
| GET | `/ping/ping` | Health check (bootstrap health.Routes) | — |
| GET | `/pension-scheme-practitioner/hmrc-frontend/assets/{file}` | hmrc-frontend static assets (play-frontend-hmrc hmrcfrontend.Routes) | — |
| GET | `/pension-scheme-practitioner/hmrc-frontend/keep-alive` | hmrc-frontend keep-alive (play-frontend-hmrc hmrcfrontend.Routes) | — |
| GET | `/pension-scheme-practitioner/hmrc-frontend/language/{lang}` | hmrc-frontend language switch (play-frontend-hmrc hmrcfrontend.Routes) | — |

### Test only

| Method | Path | Purpose | Auth |
|---|---|---|---|
