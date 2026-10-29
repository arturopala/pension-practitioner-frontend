# Pension Practitioner Frontend – Business Identifiers

This document lists every business and technical identifier the frontend handles: where it comes from, where it goes, and how it is validated. Rules are compared across three sources:

- **Local**: form mappings, regexes and normalisers in this repository (`app/forms/**`).
- **Library**: domain wrapper types from dependencies. The only one used is `uk.gov.hmrc.domain.Nino` (domain-play-30 13.0.0), whose constructor calls `require(Nino.isValid(..))`. This was confirmed with `javap`.
- **Contract**: bundled JSON or OpenAPI schemas. **The repository ships none.** The only JSON files in `conf/` are country lists, and all wire contracts live in the pension-practitioner back end. Where this document says "contract: n/a", the frontend has nothing to compare against.

Terminology:

- "PSP ID", `pspId`, `pspid` (in the subscribe response) and enrolment identifier `PSPID` all name the same value.
- "Existing PSP" / `PPID` is the **legacy** practitioner identifier from the old `HMRC-PP-ORG` enrolment.
- `idNumber` in `registrationInfo` is the NINO or the UTR, depending on `idType`.

## Summary

| Identifier | Identifies | Format (contract) | Local validation | Enters via | Main uses |
|---|---|---|---|---|---|
| PSP ID | Registered practitioner | n/a | none | Subscribe response `pspid` (trusted upstream); auth enrolment `HMRC-PODSPP-ORG`/`PSPID` (trusted) | Known-fact identifier, de-enrolment key, email callback (encrypted), audit, confirmation and view pages, cache `pspId` |
| Legacy PPID | Pre-PODS practitioner | n/a | none | Auth enrolment `HMRC-PP-ORG`/`PPID` | `existingPSP` in the subscription payload; `existingUser` flag in the start audit |
| NINO | Individual registrant | n/a | Library `Nino` regex (on construction) | Auth retrieval `nino` (trusted) | `nino` header to register-with-id; `registrationInfo.idNumber`; known-fact verifier `NINO` |
| UTR | Company (CT) / partnership (SA) | n/a | `^\d{10}$` after stripping spaces | **Form field** (untrusted) | `utr` header to register-with-id; `registrationInfo.idNumber`; known-fact verifier `CTUTR`/`SAUTR`; shown on view details |
| SAP number | ETMP business partner | n/a | must be a string | register-with(-no)-id response | `registrationInfo.sapNumber`, sent inside the subscription payload |
| Country code | Non-UK registrant (known fact) | n/a | Canonical country list | Form field (manual address) | Known-fact verifier `CountryCode` |
| Group identifier | GG group | n/a | none | Auth retrieval (trusted) | De-enrolment URL path |
| External ID | GG user | n/a | none | Auth retrieval (trusted) | Audit `userId` (enrolment and de-enrolment) |
| Credential provider ID | GG credential | n/a | none | Auth retrieval | Stored in `PSPUser.userId`, ⚠️ never used |
| Enrolment key | PSP enrolment | `HMRC-PODSPP-ORG~PSPID~{pspId}` | none | Built locally | De-enrolment URL path |
| Email address | Contact / notification target | n/a | Restrictive email regex | Form field; minimal details | Email `to`; callback URL (encrypted); deregistration email audit (plain) |
| Request / session ID | Correlation | n/a | none | Headers `X-Request-ID` / `X-Session-ID` | Forwarded by `HeaderCarrier`; email callback `requestId` (**unencoded**) |
| Journey cache key | Journey answers | n/a | – | Implicit (back end derives it from the forwarded session/auth) | `/pension-practitioner/journey-cache` |

## Practitioner IDs (PSP ID, legacy PPID)

**Lifecycle**

1. Issued by ETMP through the back end when `POST /pension-practitioner/subscribePsp/PSPSubscription` returns `{pspid}`.
2. The declaration controller saves it in the journey cache as `pspId` and immediately enrols the user's group with it as the `PSPID` identifier.
3. It is shown on the confirmation page and then the cache is cleared.
4. From then on it comes back **only from the auth enrolment** `HMRC-PODSPP-ORG` → `PSPUser.alreadyEnrolledPspId`.
5. The amend flow posts `PSPAmendment` and reads `pspid` again from the response.

**Trusted vs untrusted:** the PSP ID never comes from a request parameter, so it is always trusted. Every post-registration back-end call uses a `-self` endpoint (`getPsp-self`, `get-minimal-details-self`, `can-deregister-self`, `deregisterPsp-self`), which lets the back end resolve the practitioner from the auth context instead of a client-supplied ID. This is good practice.

**Uses:**

- known-fact identifier `PSPID`;
- the de-enrolment path `/tax-enrolments/groups/{groupId}/enrolments/HMRC-PODSPP-ORG~PSPID~{pspId}`;
- the email callback URL (encrypted with `QueryParameterCrypto`, then URL-encoded);
- audit `pensionSchemePractitionerId`;
- the view-details row;
- the amend journey's `pspId` answer, set by `PspDetailsService.uaWithUkAnswer`.

**Validation:** none locally; no contract. ⚠️ **Observation:** a request handled by a guard that does not require the enrolment, but which calls `pspIdOrException`, throws `PspIdNotFoundException` → 500. Example: `POST /remove-psp/date-removed` uses the default guard.

**Legacy PPID:** taken from `HMRC-PP-ORG`/`PPID`. It is written to `existingPSP {isExistingPSP, existingPSPId}` **only in the payload sent** to `subscribePsp`; the cached answers saved afterwards do not include it. No validation.

## User and enrolment IDs

| ID | Source | Use | Notes |
|---|---|---|---|
| External ID | `Retrievals.externalId` | Audit `userId` in `PSPEnrolmentSuccess/Failure` and `PSPDeenrolment` | Required; a missing value → unauthorised |
| Group identifier | `Retrievals.groupIdentifier` | De-enrolment URL | ⚠️ Inserted with `String.format` before the `url"…"` interpolator receives the whole string, so it is **not path-encoded**. It comes from auth, so the risk is low |
| Enrolment keys checked | `HMRC-PODSPP-ORG` (PSP), `HMRC-PODS-ORG` (PSA), `HMRC-PP-ORG` (legacy) | Guard decisions, dual-role chooser, existing-PSP flag | Presence only; no identifier format checks |
| Credential role / affinity | Auth | Eligibility | Assistant without PSP enrolment → no access |

## Party and tax IDs

### NINO

- **Lifecycle:** taken from the auth retrieval `nino` and wrapped in `uk.gov.hmrc.domain.Nino`.
- **Library rule:** prefix letters exclude D, F, I, Q, U and V (and O in second position); the prefixes BG, GB, NK, KN, TN, NT and ZZ are rejected; then six digits with optional spaces and a suffix A–D.
- ⚠️ **Observation:** the wrapper is constructed inside the auth action. An auth-supplied NINO that fails the library rule would throw `IllegalArgumentException`, which `handleFailure` does not catch, so the user would get a **500** on every guarded page.
- **Uses:**
  - header `nino` on `register-with-id/individual`;
  - `registrationInfo {idType: "NINO", idNumber}`;
  - known-fact verifier `NINO` (UK individuals).
- `IsThisYouController` reuses the cached registration only when `info.idType == NINO` **and** `info.idNumber == auth nino`. This is a sound binding of cached data to the authenticated user.
- **Never from user input.**

### UTR

- **Lifecycle:** typed by the user on `/register/{company|partnership}/business-utr`, stored as `utr`, sent as header `utr` to `register-with-id/organisation`, then recorded as `registrationInfo.idNumber` with `idType: "UTR"`.
- **Known facts:** company legal status → `CTUTR`; partnership → `SAUTR`. Unlimited companies map to the company legal status.
- **Local rule:** spaces stripped, `^\d{10}$`. No check-digit test, and other separators (`-`) or a trailing `K` are rejected rather than normalised.
- ⚠️ **Observation:** the UTR is **not bound to the authenticated user**. No CT/SA enrolment or credential check ties it to the GG account; the only gate is the ETMP match on UTR, name and type in the back end. The matched organisation name then replaces the one the user typed.

### SAP number and country code

- **SAP number:** comes from the register responses and is used only inside `registrationInfo` (sent in the subscription payload).
- **Country code:** used as a known fact for `NonUK` registrants. It is taken from the cached registered address (`CompanyRegisteredAddressPage`, `PartnershipRegisteredAddressPage` or `IndividualAddressPage`).

### Party identifiers table

| Identifier | Frontend key(s) | Upstream key (per flow) | Local rule | Contract rule |
|---|---|---|---|---|
| NINO | `registrationInfo.idNumber` (`idType: NINO`) | register-with-id individual: header `nino`. Enrol: verifier `NINO`. Subscribe: inside `registrationInfo` | Library `Nino.isValid` | n/a |
| UTR (company) | `utr`, `registrationInfo.idNumber` (`idType: UTR`) | register-with-id organisation: header `utr`. Enrol: verifier `CTUTR`. Subscribe: `utr` + `registrationInfo` | `^\d{10}$` after space strip | n/a |
| UTR (partnership) | as above | Enrol: verifier `SAUTR` | same | n/a |
| Country | `registeredAddress.country` | Enrol: verifier `CountryCode` | Canonical list; GB enforced on UK address forms | n/a |
| PSP ID | `pspId` | Enrol: identifier `PSPID`. De-enrol: enrolment key path | none | n/a |

**Normalisation:** the UTR has spaces removed (`Transforms.strip`). The NINO is used exactly as auth returns it, so the cached-registration equality check is a raw string compare.

## Technical IDs

- **Correlation:** `X-Request-ID` and `X-Session-ID` are forwarded automatically to every downstream call. For emails, `requestId = hc.requestId`, else the raw `X-Session-ID` request header, else `""`. ⚠️ It is placed in the callback **path** without encoding, so an empty value produces a `//` segment.
- **Journey cache:** no key is sent explicitly; the back end determines it.
- **Email callback:** path `/pension-practitioner/email-status-response/{journeyType}/{requestId}/{encryptedEmail}/{encryptedPspId}`. The email address and PSP ID are encrypted and URL-encoded.

## Validation coverage matrix

| Identifier | Format checked locally? | Checked against the authenticated user? | Encoded when put in a URL? |
|---|---|---|---|
| PSP ID | ❌ | ✅ always taken from the enrolment, or the `-self` endpoints | ⚠️ de-enrol path: not encoded. Callback: ✅ encrypted and encoded |
| Legacy PPID | ❌ | ✅ from the enrolment | n/a |
| NINO | ✅ library (throws → 500) | ✅ from auth; cached match compared with auth | n/a (header) |
| UTR | ✅ `^\d{10}$` | ❌ user-supplied, verified only by the ETMP match | n/a (header) |
| SAP number | ❌ | n/a | n/a |
| Group identifier | ❌ | ✅ from auth | ⚠️ not encoded |
| Country code (known fact) | ✅ canonical list | n/a | n/a |
| Email address | ✅ regex (no length) | ❌ user-entered | ✅ encrypted and encoded in the callback |
| Request / session ID | ❌ | n/a | ❌ not encoded in the callback |

## Recommendations

1. Build URLs with `url"$base/groups/$groupId/enrolments/$key"` so the interpolator encodes each segment, rather than formatting the whole URL first. The same applies to `subscribePsp/%s` and the email callback `requestId`.
2. Construct `Nino` defensively in the auth action (e.g. `Nino.isValid` → unauthorised), so an unexpected value does not become a 500 on every page.
3. Use the enrolment-requiring guard for `remove-psp/date-removed` and the success pages, so `pspIdOrException` cannot fail.
4. Consider normalising UTRs (accept and strip a `K` suffix and separators) and adding the UTR check-digit test before calling ETMP.
5. Keep the legacy `existingPSP` answer in the saved cache, or document that it exists only in the subscription payload.
6. Avoid plain email addresses in the `PensionSchemePractitionerDeregistrationEmail` audit if the audit policy does not require them.
