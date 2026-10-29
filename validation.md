# Pension Practitioner Frontend – Data Validation

Validation is **form-centric and mostly syntactic**. User input arrives only as HTML form posts bound by Play `Form` mappings. Each mapping checks presence, length and a regex, and a failure re-renders the page with **400** and a GOV.UK error summary. Data that comes back from upstream systems (registration matches, address-lookup results, subscription details) is parsed **structurally** and then trusted: it is not re-checked against the same rules as user input. Outbound payloads are not schema-checked; the journey answers are sent to the back end as-is. Most non-form failures become a redirect, either to "session expired" or to a stop page, or a generic **500** page.

## Validation layers at a glance

| # | Layer | Mechanism | Where | On failure |
|---|---|---|---|---|
| 1 | Routing | Static paths; one `String` path param (`entityType`), one `String` (`lang`), `*file` assets | `conf/*.routes` | Unknown path → 404 page (`FrontendErrorHandler.onClientError`) |
| 2 | CSRF | Play CSRF filter (bootstrap frontend filters) on every POST | platform | 403 standard error template |
| 3 | Authentication / eligibility | `AuthenticatedAuthAction` family | `controllers.actions.AuthAction` | 303 redirects (login, IV, unauthorised, stop pages, dashboard) |
| 4 | Journey state presence | `DataRequiredActionImpl`, `Retrievals.retrieve`, `DataRetrievals` | controllers | 303 → session expired (or unauthorised in `IsThisYouController`) |
| 5 | Form binding | `Mappings`/`Formatters`/`Constraints`/`AddressMappings`/`LocalDateFormatter` | `app/forms/**` | 400 re-rendered page |
| 6 | Business rules | Navigators (UK only, EU/EEA region), `PspDetailsService.amendmentsExist`, can-deregister check | `app/navigators`, controllers | 303 to stop/info page |
| 7 | External match | register-with-id (organisation, individual) | `RegistrationConnectorImpl` | Organisation 404 → "business details not found"; other → 500 |
| 8 | Upstream response parsing | Play JSON `validate`/`as` | connectors, `UserAnswers` | `JsResultException` → 500 |
| 9 | Outbound payload | none | – | Back end/ETMP rejects → mapped by declaration controllers |

## Route and path parameters

| Parameter | Rule | Code |
|---|---|---|
| `entityType` (`/register/{entityType}/you-need-to-tell-hmrc`) | Any string; used as a message key (`messages(entityType)`). Twirl escapes the output, so it is safe, but unvalidated. | `TellHMRCController.onPageLoad` |
| `lang` (`/language/{lang}`) | Looked up in `languageMap` (`english`, `cymraeg`); unknown → default language | `LanguageSwitchController` |
| `file` (assets) | Play `Assets` | – |
| `mode` | Not a request parameter: fixed per route (`NormalMode` / `CheckMode`) | routes files |

## Request headers

| Endpoint | Headers | Validation |
|---|---|---|
| All guarded pages | Session cookie / `Authorization` | Validated by the auth service |
| `GET /language/{lang}` | `Referer` | ⚠️ **Observation:** none. The user is redirected to whatever `Referer` holds, which makes it an open redirect if an attacker can control the Referer. |
| Declarations, deregistration | `X-Request-ID` / `X-Session-ID` | Read into the email callback `requestId` without validation or URL-encoding |

## Request bodies (form posts)

### Yes/No questions

Applies to: are-you-UK, confirm name, confirm address, use same address, is-registered-in-UK, is-this-you, use address for contact, confirm deregistration.

- **Mapping:** `boolean(requiredKey)`. The field `value` is trimmed and must be `true` or `false`.
- **Errors:**
  - missing or empty → the page-specific `*.error.required` message;
  - any other value → `error.boolean`.

### Enumerations

| Form | Field | Allowed | Error |
|---|---|---|---|
| Who are you registering | `value` | `companyOrPartnership`, `yourselfAsIndividual` | `whatTypeBusiness.error.required` / `error.invalid` |
| Business type | `value` | `limitedCompany`, `businessPartnership`, `limitedPartnership`, `limitedLiabilityPartnership`, `unlimitedCompany`, **`overseasCompany`** | `businessType.error.required` / `error.invalid` |
| Business registration type | `value` | `company`, `partnership` | `businessRegistrationType.error.required` |

⚠️ **Observation:** `overseasCompany` is accepted by the form although no radio button offers it.

- **Next page:** the navigator sends that value to "session expired".
- **Crash path:** if the user then goes straight to `/register/company/confirm-address` with a cached UTR and name, `Organisation.apply` throws `IllegalArgumentException` → **500**.

### Text fields

| Field (form) | What is checked | What is not checked | Error keys |
|---|---|---|---|
| UTR (`BusinessUTRFormProvider`) | Required. Spaces stripped. `^\d{10}$` | No check digit; a `K` suffix or other separators are rejected rather than normalised | `businessUTR.company.error.required` / `.invalid` |
| Company/partnership name (`BusinessNameFormProvider`) | Required. Trimmed. Max 105 | **No character set**, so any Unicode or control characters pass | `companyName.error.required` / `.length` (the `invalidKey` is never used) |
| Email (`EmailFormProvider`) | Required. Trimmed. Restrictive email regex | **No max length:** `maxEmailLength = 132` is declared but never applied | `*.error.required`, `email.error.invalid` |
| Phone (`PhoneFormProvider`) | Required. Trimmed. `^[0-9 ()+--]{1,24}$` | ⚠️ `+--` is a character range from `+` to `-`, so a **comma** is also allowed | `phone.error.invalid` |
| Postcode for lookup (`PostcodeFormProvider`) | Required. Upper-cased. Spaces collapsed. UK postcode regex. Space inserted before the inward code | – | `postcode.error.required` / `.invalid` |
| Address index (`AddressListFormProvider`) | Required integer (commas removed, decimals rejected) | ⚠️ **No range check:** an index ≥ number of results, or negative, makes `addresses(value)` throw `IndexOutOfBoundsException` → **500** | page-specific required key |

### Manual address forms

| Form | Used by | Fields |
|---|---|---|
| `UKAddressFormProvider` | company/partnership contact address and registered address | `line1`, `line2` required: ≤ 35 chars, `^[A-Za-z0-9 \-,.&'\/]{1,35}$`. `line3`, `line4` optional: same rules after whitespace standardisation. `postcode`: required when country = GB, UK regex, normalised; non-GB ≤ 10 chars. `country`: in the canonical list **and** = `GB`. |
| `AddressFormProvider` | individual contact address | Same, but `country` can be any canonical-list code |
| `NonUKAddressFormProvider`, `RegisteredAddressFormProvider` | defined | ⚠️ Not used by any controller |

### Deregistration date (`DeregistrationDateFormProvider`)

- Fields: `deregistrationDate.day`, `.month`, `.year`. All three are required; missing parts give per-field "one required" errors.
- Each part goes through the int formatter; `LocalDate.of` must succeed, otherwise "invalid".
- `maxDate(today)` and `minDate(applicationDate)`. The minimum comes from `getPsp-self.applicationDate`; failing to fetch it → 500 before the form is bound.
- ⚠️ `allRequiredKey` and `twoRequiredKey` are passed in but never used. Every missing field produces a "one required" error.

### Forms with no body binding

The registration and amend **declarations** post only the CSRF token, and the controllers bind nothing.

`POST /register/individual/use-address-for-contact` is routed to `onPageLoad`, so it ignores the body and re-renders the page. The page's form actually posts to `/register/individual/change/use-address-for-contact`.

### Normalisation and derivation

| Input | Rule | Code |
|---|---|---|
| All `text()` fields | `trim`; empty → required error | `Formatters.stringFormatter` |
| Optional text | Whitespace runs collapsed, trimmed, empty → `None` | `optionalStringFormatter` |
| UTR | All spaces removed | `UtrMapping` |
| Postcode | Upper-case, multiple spaces collapsed, space inserted before the final 3 chars | `Transforms.postCodeTransform` / `postCodeValidTransform` |
| Country | Spaces stripped, upper-case (for the postcode rule) | `countryDataTransform` |
| Address chosen from lookup | `countryOpt` forced to `GB`. Converted with `TolerantAddress.toAddress`, shuffling lines when 1/2 are missing. If no conversion is possible, the user goes to manual entry pre-filled | `AddressListController.post` |
| "Use same address" | Uses the registered or matched address only if lines 1 and 2 exist; otherwise answer stored as `false` | `CompanyUseSameAddressController.getResolvedAddress` |
| Business type → organisation type | limited/unlimited company → `Corporate Body`; business or limited partnership → `Partnership`; LLP → `LLP`; overseas → **exception** | `Organisation.apply` |
| Registered address → customer type | Country `GB`/`UK` → UK; other → NonUK; **none → exception (500)** | `RegistrationCustomerType.fromAddress` |
| Region | GB → UK; in the EU/EEA canonical list → EuEea; else rest of world | `CountryOptions.regions` |
| Variation flags | In amend mode, sets `nameChange` / `addressChange`. ⚠️ Email and phone edits also set **`addressChange`** | `Variation.setChangeFlag`, email and phone controllers |

## External validation services

| Service | Request | Decision rule | Fail behaviour |
|---|---|---|---|
| Register-with-id organisation (back end → ETMP) | UTR header, name and organisation type | 200 = match. The matched name **overwrites** the user-entered name. The user confirms the address | 404 → "business details not found" (fail closed). Other errors and parse errors → 500 |
| Register-with-id individual | NINO header (from auth) | 200 = match; user confirms "is this you" | Any error, including 404 → **500** (no recovery) |
| Address lookup | Postcode | Empty list → 400 "no results" | Non-200 → 500 (fail closed) |
| Can-deregister | – | `false` → "deauthorise from schemes" page | **404 → `true`: fails open**. Other errors → 500 |
| Minimal details (deceased flag) | – | `deceasedFlag` → "contact HMRC" | Errors → 500 on **every** guarded page for enrolled users |

## Outbound payload validation

- There are **no JSON schemas** in the repository and no pre-send validation.
- `subscribePsp` posts the **entire `UserAnswers` document**, including the internal journey keys (`postcode` lookup results, `addressList`, `confirmName`, `declaration`, `existingPSP`, and in amend mode `unchangedPspDetails`, `subscriptionType`, `nameChange`, `addressChange`). Mapping and validation are left to the back end.
- Addresses selected from lookup or returned by register-with-id are **not re-validated** against the 35-character/charset rules applied to manual entry, so lookup lines that break those rules are sent upstream.
- Failures are only seen as upstream status codes. The declaration controllers map them as follows:

  | Upstream result | Outcome |
  |---|---|
  | 5xx | "your action was not processed" |
  | 4xx with `ACTIVE_PSPID` | "cannot register" |
  | anything else | "session expired" |

- **Logging:** connectors log full upstream response bodies on failure (`HttpResponseHelper` messages, `MinimalConnector` JSON errors). `EnrolmentConnector` logs the full known facts (PSP ID plus NINO or UTR) at **debug** on failure. There is no PII masking.

## Upstream response validation

| Response | Validation | Failure result |
|---|---|---|
| journey-cache GET | 200 → JSON. `DataRetrievalAction` does `data.as[JsObject]` | Non-object → `JsResultException` → 500 |
| journey-cache POST | Status must be 201 | `HttpException` → 500 |
| register-with-id org/individual | `organisation`/`individual` + `address` (all address fields optional) + `sapNumber` string | `JsResultException` → 500 |
| register-with-no-id | `sapNumber` | 500 |
| subscribePsp | `pspid` string | 500 |
| getPsp-self | Opaque JSON (`as[JsObject]`); `applicationDate` string split on `T`, `LocalDate.parse` | Parse error → 500. 400 `INVALID_IDVALUE`/`INVALID_CORRELATIONID` and 404 → custom exceptions → 500 |
| get-minimal-details-self | `MinimalPSP` format (`email`, `rlsFlag`, `deceasedFlag` required) | 500 |
| can-deregister-self | JSON boolean | 500 |
| address lookup | `address.lines`, `address.postcode`, `address.country.code` required per item; one bad item fails the whole list | 500 |
| session-data-self | `as[JsObject]` and an optional `administratorOrPractitioner` enum | 500 |
| email | Status only | `EmailNotSent`, journey continues |
| tax-enrolments | Status 204 | Enrol: mapped error (retried for `UpstreamErrorResponse`). De-enrol: `HttpException` → 500 |

## Inbound callbacks

None. The email delivery callback targets the pension-practitioner back end.

## Cache / opaque stores

The journey cache accepts any JSON object. Page reads use `Reads.optionNoError`, so a value of the wrong type reads as **absent**, which usually leads to a "session expired" redirect, not an error. Writes use `setObject`. `getOrException` / `setOrException` / `removeOrException` throw `RuntimeException` → 500. Example: `UpdateContactAddressController` when the contact address is missing.

## Error mapping summary

| Source | Exception / value | HTTP to caller |
|---|---|---|
| Form binding | `FormError` | 400 (page re-rendered) |
| CSRF filter | missing or invalid token | 403 |
| Auth | `NoActiveSession` / `InsufficientConfidenceLevel` / other `AuthorisationException`s | 303 login / 303 IV / 303 unauthorised |
| Journey state | missing page value | 303 session expired |
| Registration declarations | `UpstreamErrorResponse` 5xx / 4xx + `ACTIVE_PSPID` / **any other exception** (known facts missing, enrolment failure, cache failure, email…) | 303 not processed / 303 cannot register / **303 session expired** |
| Amend declaration | 5xx / any other | 303 not processed / 303 session expired |
| Organisation confirm address | `NotFoundException` | 303 business details not found |
| Deregistration | any exception (deregister, de-enrol, minimal details, application date) | 500 |
| Address list | `IndexOutOfBoundsException` | 500 |
| Any upstream JSON | `JsResultException` | 500 |
| `PspIdNotFoundException` (`pspIdOrException` without enrolment) | thrown in the action body | 500 |
| `ApplicationException(result)` | – | that result (handler passthrough) |
| Anything else | – | 500 internal-server-error page, `Cache-Control: no-cache` |

## Recommendations

1. Add a range check to `AddressListFormProvider` (`min 0`, `max addresses.size-1`), or guard `addresses.lift(value)`, so that tampered indexes return 400 rather than 500.
2. Remove `overseasCompany` from the accepted `BusinessType` values (or handle it), and make `RegistrationCustomerType.fromAddress` fail gracefully when the country is missing.
3. Apply `maxEmailLength` (132) to email, add a character-set rule to business names, and fix the phone regex (`[0-9 ()+\-]`).
4. Validate or truncate addresses from address lookup and register-with-id before they are stored, using the same line rules as manual entry.
5. Stop mapping every non-5xx declaration failure to "session expired". Above all, a failure **after** a successful subscription (enrolment, cache save) leaves a PSP ID with no enrolment and a misleading message.
6. Validate the `Referer` in the language switch (relative URL or allow-list) to close the open redirect.
7. Send the back end an explicit subscription DTO instead of raw `UserAnswers`, and URL-encode `requestId` in the email callback.
8. Do not log the full known facts or upstream bodies that contain personal data.
