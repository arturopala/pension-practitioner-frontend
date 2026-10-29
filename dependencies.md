# Pension Practitioner Frontend – Dependencies

All outbound HTTP goes through the bootstrap **`HttpClientV2`**. The `HttpClientV2Module` is enabled in `application.conf`. Each call carries a `HeaderCarrier` built from the incoming request and session (`HeaderCarrierConverter.fromRequestAndSession`), so for internal hosts the platform forwards the user's `Authorization` bearer token, `X-Session-ID`, `X-Request-ID` and the other standard tracing headers. There is **no service-to-service credential**: every downstream call runs as the signed-in user.

Base URLs come from `ServicesConfig.baseUrl("<service>")`, i.e. `microservice.services.<service>.{protocol,host,port}`. Paths are mostly config keys under `urls.*`, appended to the base URL in `FrontendAppConfig` as `lazy val`s. Some are `String.format` templates, filled in per call and then passed whole to the `url"…"` interpolator.

## Overview

| # | Dependency | Type | Config key | Accessed via |
|---|---|---|---|---|
| 1 | pension-practitioner | Back-end microservice (HTTP/JSON) | `microservice.services.pension-practitioner` + `urls.*` | `UserAnswersCacheConnectorImpl`, `RegistrationConnectorImpl`, `SubscriptionConnector`, `MinimalConnectorImpl`, `DeregistrationConnectorImpl`, `EmailConnector` (callback URL only) |
| 2 | pension-administrator | Back-end microservice | `microservice.services.pension-administrator` | `SessionDataCacheConnector` |
| 3 | auth | Platform | `microservice.services.auth` | `AuthConnector` (bootstrap `AuthModule`) in `AuthenticatedAuthAction` and subclasses, `SignOutController` |
| 4 | tax-enrolments | Platform | `microservice.services.tax-enrolments` + `urls.tax-enrolments`, `urls.tax-de-enrolment` | `EnrolmentConnectorImpl` |
| 5 | address-lookup | Platform | `microservice.services.address-lookup` | `AddressLookupConnector` |
| 6 | email | Platform | `microservice.services.email` + `email.*` | `EmailConnector` |
| 7 | datastream / audit | Platform | bootstrap `auditing.*` | `AuditServiceImpl` → `AuditConnector` |
| 8 | identity-verification frontend | Browser redirect | `urls.iv-uplift-entry` | `AuthenticatedAuthAction.handleFailure` → `FrontendAppConfig.identityValidationFrontEndEntry` |
| 9 | manage-pension-schemes frontend, login, feedback, contact-frontend, tracking-consent | Browser redirects/links | `urls.*`, `microservice.services.contact-frontend`, `tracking-consent-frontend.*` | `FrontendAppConfig`, views |

---

## 1. pension-practitioner (back end)

- **Base URL:** `servicesConfig.baseUrl("pension-practitioner")`, default `http://localhost:8209`.
- **Auth:** the user's bearer token (forwarded `HeaderCarrier`). The `-self` endpoints derive the PSP from the caller's auth context; the frontend never sends a PSP ID in those requests.

| Operation | Method & path | Code | Notes / validation |
|---|---|---|---|
| Fetch journey answers | `GET /pension-practitioner/journey-cache` | `UserAnswersCacheConnectorImpl.fetch` | 200 → JSON; 404 → `None`; other → `HttpException` (500 to the user). Used by `DataRetrievalActionImpl` on every guarded page. |
| Save journey answers | `POST /pension-practitioner/journey-cache` | `.save` | Body is the whole `UserAnswers` JSON. Only **201** counts as success; anything else → `HttpException`. |
| Clear journey answers | `DELETE /pension-practitioner/journey-cache` | `.removeAll` | Status ignored (always `Ok`). |
| Register with ID – organisation | `POST /pension-practitioner/register-with-id/organisation` | `RegistrationConnectorImpl.registerWithIdOrganisation` | Header `utr: <UTR>`. Body `{organisationName, organisationType}`. 200 → `{organisation, address}` + `sapNumber` parsed; `JsResultException` on bad JSON. 404 → `NotFoundException` (→ "business details not found"). Other statuses via `HttpResponseHelper`. |
| Register with ID – individual | `POST /pension-practitioner/register-with-id/individual` | `.registerWithIdIndividual` | Header `nino: <NINO>` (from auth). Body `{}`. 200 → `{individual, address}` + `sapNumber`. |
| Register without ID – organisation | `POST /pension-practitioner/register-with-no-id/organisation` | `.registerWithNoIdOrganisation` | Body `{organisation:{organisationName}, address:{…}}`. 200 → `sapNumber`. Customer type is forced to `NonUK`. |
| Register without ID – individual | `POST /pension-practitioner/register-with-no-id/individual` | `.registerWithNoIdIndividual` | ⚠️ **Observation:** never called anywhere in the app. |
| Subscribe / amend PSP | `POST /pension-practitioner/subscribePsp/{journeyType}` | `SubscriptionConnector.subscribePsp` | `journeyType` ∈ `PSPSubscription`, `PSPAmendment`. Body = the raw `UserAnswers` JSON. 200 → `pspid` read from the response; other → `HttpResponseHelper`. 5xx/4xx become `UpstreamErrorResponse`, inspected by the declaration controllers (`ACTIVE_PSPID`). |
| Subscription details | `GET /pension-practitioner/getPsp-self` | `.getSubscriptionDetails` | 200 → JSON (becomes the amend journey's `UserAnswers`). 400 containing `INVALID_IDVALUE` / `INVALID_CORRELATIONID` and 404 → custom `SubscriptionException`s (all end in 500). |
| Application date | (same call) | `.getPspApplicationDate` | Reads `applicationDate`, splits on `T`, then `LocalDate.parse`. Used as the minimum deregistration date. |
| Minimal details | `GET /pension-practitioner/get-minimal-details-self` | `MinimalConnectorImpl.getMinimalPspDetails` | Extra header `loggedInAsPsp: true`. 200 → `MinimalPSP {email, organisationName?, individualDetails?, rlsFlag, deceasedFlag}`. Called by the auth action **on every guarded request by an enrolled PSP** (deceased check), by deregistration and by view details. |
| Can deregister | `GET /pension-practitioner/can-deregister-self` | `DeregistrationConnectorImpl.canDeRegister` | 200 → JSON boolean; **404 → `true`**; other → error. |
| Deregister | `POST /pension-practitioner/deregisterPsp-self` | `.deregister` | Body `{deregistrationDate: "yyyy-MM-dd", reason: "1"}`. 200 only. |
| Email status callback (target) | `POST /pension-practitioner/email-status-response/{journeyType}/{requestId}/{encryptedEmail}/{encryptedPspId}` | built in `EmailConnector.callBackUrl` / `FrontendAppConfig.emailCallback` | Registered with the email service as `eventUrl`; this frontend never receives it. |

Error mapping (`HttpResponseHelper.handleErrorResponse`): 400 → `BadRequestException`, 404 → `NotFoundException`, other 4xx → `UpstreamErrorResponse(status)`, 5xx → `UpstreamErrorResponse(status, reportAs 502)`, anything else → `UnrecognisedHttpResponseException`.

⚠️ **Observation:** the journey cache save treats only `201 Created` as success. A `200` from the back end would surface as a 500.

## 2. pension-administrator (back end)

- **Base URL:** `servicesConfig.baseUrl("pension-administrator")`, default `http://localhost:8205`.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Fetch session data | `GET /pension-administrator/journey-cache/session-data-self` | `SessionDataCacheConnector.fetch` | Called only for users holding both `HMRC-PODS-ORG` and `HMRC-PODSPP-ORG`. Reads `administratorOrPractitioner`. 404 → `None`; other non-200 → `HttpException` (500). |
| Clear session data | `DELETE /pension-administrator/journey-cache/session-data-self` | `.removeAll` | On sign-out; status ignored. |

## 3. auth (platform)

Used through `AuthorisedFunctions.authorised(...)`. See the [Auth section](#auth).

## 4. tax-enrolments

- **Base URL:** `servicesConfig.baseUrl("tax-enrolments")`, default `http://localhost:8202`.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Enrol | `PUT /tax-enrolments/service/HMRC-PODSPP-ORG/enrolment` | `EnrolmentConnectorImpl.enrol` | Body is `KnownFacts {identifiers:[{key:"PSPID",value}], verifiers:[{key: NINO\|CTUTR\|SAUTR\|CountryCode, value}]}`, built by `KnownFactsRetrieval`. **204** → success + audit `PensionSchemePractitionerEnrolmentSuccess`. Other → audit `…EnrolmentFailure` (with status) + `HttpResponseHelper` error. Retried by `RetryHelper` on `UpstreamErrorResponse`. |
| De-enrol | `DELETE /tax-enrolments/groups/{groupId}/enrolments/HMRC-PODSPP-ORG~PSPID~{pspId}` | `.deEnrol` | 204 → audit `PensionSchemePractitionerDeEnrolment`. Other → `HttpException`. ⚠️ **Observation:** `HttpException` is not an `UpstreamErrorResponse`, so **de-enrolment failures are never retried**, although the method is wrapped in `retryOnFailure`. |

Retry (`RetryHelper`): up to `retry.max.attempts` (10). The first wait is `retry.initial.wait.ms` (10 ms), multiplied by `retry.wait.factor` (1.5) per attempt. Only `UpstreamErrorResponse` triggers a retry. ⚠️ **Observation:** `RetryHelper` creates a new `ActorSystem()` per mixing class instance and never shuts it down.

## 5. address-lookup

- **Base URL:** `servicesConfig.baseUrl("address-lookup")`, default `http://localhost:9022`.
- **Headers:** `X-Hmrc-Origin: PODS`.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Lookup by postcode | `POST /lookup` body `{"postcode": "<pc>"}` | `AddressLookupConnector.addressLookupByPostCode` | 200 → `TolerantAddress.postCodeLookupReads` (lines, postcode, country code, town, county folded into four lines). Addresses with no lines are dropped. Non-200 → `HttpException` → **500**. An empty result → the postcode form re-rendered with 400 "no results". |

## 6. email

- **Base URL:** `servicesConfig.baseUrl("email")`, default `http://localhost:8300`.

| Operation | Method & path | Code | Notes |
|---|---|---|---|
| Send email | `POST /hmrc/email` | `EmailConnector.sendEmail` | Body `SendEmailRequest {to:[email], templateId, parameters:{pspName}, force: email.force, eventUrl}`. **202** → `EmailSent`. Any other status or exception → `EmailNotSent` (logged, **never fails the journey**). |

Templates: `email.pspSubscriptionTemplateId` (`pods_psp_register`), `email.pspAmendmentTemplateId` (`pods_psp_amend`), `email.pspDeregistrationTemplateId` (`pods_psp_deregister_self`).

`eventUrl` is the pension-practitioner callback (see §1). `pspId` and `email` are encrypted with `ApplicationCrypto.QueryParameterCrypto` and URL-encoded. `requestId` is `X-Request-ID`, falling back to the incoming `X-Session-ID` header and then `""`. ⚠️ **Observation:** `requestId` is interpolated without URL-encoding.

## 7. Audit

`AuditServiceImpl` sends `DataEvent`s (with `toAuditDetails()` request details) and `ExtendedDataEvent`s. `auditSource` is `appName`. Failures are logged only.

| Audit type | When | Detail |
|---|---|---|
| `PSPStartNew` | Who-are-you-registering submitted | `userType`, `existingUser` |
| `PensionSchemePractitionerEnrolmentSuccess` / `…Failure` | Enrol PUT | `userId` (external id), `pensionSchemePractitionerId`, `errorCode` |
| `PensionSchemePractitionerAmendment` (extended) | Amend declaration | `pensionSchemePractitionerId`, `from`/`to` lists of changed top-level keys |
| `PensionSchemePractitionerDeregistration` | After deregister succeeds | `pensionSchemePractitionerId` |
| `PensionSchemePractitionerDeEnrolment` | De-enrol 204 | `userId`, `pensionSchemePractitionerId` |
| `PensionSchemePractitionerDeregistrationEmail` | After the email call (sent or not) | `pensionSchemePractitionerId`, `emailId` (**email address**) |

## 8–9. Browser redirects

| Config key | Target / purpose |
|---|---|
| `urls.login`, `urls.loginContinue` | Government Gateway login, returning to the service entry |
| `urls.logout` | Feedback survey (after sign-out) |
| `urls.iv-uplift-entry` | IV uplift, `origin=pods&confidenceLevel=250`, relative completion/failure URLs |
| `urls.administratorOrPractitioner`, `urls.cannotAccessPageAsAdministrator` | MPS role chooser for dual-enrolled users |
| `urls.pspDashboard`, `urls.overview`, `urls.pspListSchemes`, `urls.youNeedToRegister`, `urls.youMustContactHMRC` | MPS frontend pages |
| `contact-frontend.*`, `tracking-consent-frontend.*`, `accessibility-statement.service-path` | Standard platform links |

---

## Datastore

The service has **no database of its own**.

| Store | Accessed through | Key | Expiry | Purpose |
|---|---|---|---|---|
| pension-practitioner `journey-cache` | `UserAnswersCacheConnectorImpl` | Determined by the back end from the forwarded session/auth headers (no id in the URL) | Back-end defined; cleared on confirmation pages and sign-out | All journey answers (`UserAnswers` JSON), including `registrationInfo`, `pspId`, `unchangedPspDetails`, `subscriptionType` |
| pension-administrator `session-data-self` | `SessionDataCacheConnector` | Back-end defined | Back-end defined; cleared on sign-out | Dual-enrolment role choice (`administratorOrPractitioner`) |

Encryption: none in this service, except the encrypted callback URL parameters. Any encryption at rest is up to the back end.

## Auth

All guards extend `AuthenticatedAuthAction`. Each runs `authorised(User or Assistant [and ConfidenceLevel.L250])` and retrieves `externalId, affinityGroup, allEnrolments, credentials, credentialRole, groupIdentifier, nino`. A missing required retrieval redirects to Unauthorised.

| Guard (binding) | Requires | Extra rule |
|---|---|---|
| `AuthAction` → `AuthenticatedAuthActionWithNoIV` | Organisation affinity | none |
| `@AuthWithIV` → `AuthenticatedAuthActionWithIV` | + CL250 | none |
| `@AuthMustHaveNoEnrolmentWithIV` | + CL250 | Holding `HMRC-PODSPP-ORG` → PSP dashboard |
| `@AuthMustHaveNoEnrolmentWithNoIV` | | Holding `HMRC-PODSPP-ORG` → PSP dashboard |
| `@AuthMustHaveEnrolmentWithNoIV` | | Not holding `HMRC-PODSPP-ORG` → "you need to register" |

Common steps, in order:

1. Both `HMRC-PODS-ORG` and `HMRC-PODSPP-ORG` held → fetch the administrator session data. No choice → role chooser; `Administrator` → "cannot access as administrator".
2. Agent → agent stop page. Individual → organisation-account page. Assistant without `HMRC-PODSPP-ORG` → no-access page.
3. Guard-specific enrolment rule.
4. If enrolled → minimal-details call; `deceasedFlag` → "you must contact HMRC".

`PSPUser` carries:

- `alreadyEnrolledPspId`, from `HMRC-PODSPP-ORG` / `PSPID`;
- `existingPSPId`, from the legacy `HMRC-PP-ORG` / `PPID`;
- `nino`, wrapped in `domain.Nino`;
- `userId`, the credentials provider id;
- `groupIdentifier`.

Failure mapping:

| Failure | Redirect |
|---|---|
| `NoActiveSession` | login |
| `InsufficientConfidenceLevel` | IV (303) |
| `InsufficientEnrolments`, `UnsupportedAuthProvider`, `UnsupportedAffinityGroup`, `UnauthorizedException` | Unauthorised |

`SignOutController` uses a bare `authorised()`; `MissingBearerToken` still signs out.

## Inbound callers and callbacks

- **Browsers only.** There are no service-to-service inbound endpoints.
- **Webhook registered elsewhere:** the email service is given the pension-practitioner `email-status-response` URL (§1, §6). Callbacks go to the back end, not to this frontend.
- **Test-only routes:** `testOnlyDoNotUseInAppConf.routes` (`/test-only/*`, error-handler probes) are mounted only when started with `-Dapplication.router=testOnlyDoNotUseInAppConf.Routes`.

## Libraries

| Library | Version | Used for |
|---|---|---|
| `bootstrap-frontend-play-30` | 10.8.0 | `HttpClientV2`, auth, audit, crypto, frontend filters (CSRF, session), health routes |
| `play-frontend-hmrc-play-30` | 13.15.0 | GOV.UK/HMRC Twirl components, `hmrcfrontend.Routes` |
| `govuk-frontend` (webjar) | 6.5.1 | Front-end assets |
| `domain-play-30` | 13.0.0 | `uk.gov.hmrc.domain.Nino` |
| `play-conditional-form-mapping-play-30` | 3.5.0 | Form mappings |
| `guice-multibindings` | 4.2.3 | Navigator set binding |
| `jackson-module-scala` | 2.22.3.1 | Jackson Scala support |
| Play (sbt-plugin) / Scala | 3.0.10 / 3.7.1 | Runtime |

## Local development

- Run `sm2 --start PODS_ALL`, then `sbt run` on port **8208**.
- Default local ports:

  | Dependency | Port |
  |---|---|
  | auth | 8500 |
  | address-lookup | 9022 |
  | pension-practitioner | 8209 |
  | pension-administrator | 8205 |
  | email | 8300 |
  | tax-enrolments | 8202 |
  | contact-frontend | 9250 |

- Login uses the auth-login stub (`localhost:9949`). IV goes to `localhost:9938/mdtp/uplift`, and MPS pages are on `localhost:8204`.
- Head-of-duty data for the back end comes from `pensions-scheme-stubs`.
