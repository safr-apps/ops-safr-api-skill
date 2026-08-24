---
name: safr-api
description: Use when integrating with, querying, operating, or troubleshooting the SAFR Platform REST APIs—COVI, CVEV, CVOS, or VIRGA/VRGA—including choosing dev, int2, prod, or custom service endpoints. Use for constructing requests, generating clients, interpreting responses, or diagnosing SAFR authentication and privilege errors. Not for generic computer-vision work that does not use SAFR.
---

# SAFR Platform APIs

Use this skill to select the correct SAFR service, contract, and deployment before constructing or sending a request.

## Select the service contract

SAFR exposes four services on separate base URLs. Read only the contract or contracts involved in the request.

| Service | Use it for | Authoritative operation contract |
| --- | --- | --- |
| COVI | Face detection and recognition; people, groups, objects, directories, users, tenants, access, and invitations | [references/covi.yml](references/covi.yml), API `3.21.1022` |
| CVEV | Recognition events, long polling, alarms, event search, event archiving, and event biometric indexing | [references/cvev.yaml](references/cvev.yaml), API `3.34.830` |
| CVOS | Person images, object and stream storage, preferences, logs, common objects, and maps | [references/cvos.yaml](references/cvos.yaml), API `3.0.297.1` |
| VIRGA (VRGA) | Video workers and feeds, status, tasks, configuration, image streams, and zones | [references/vrga.yaml](references/vrga.yaml), API `3.34.816` |

Use [references/SAFRSystemAPIsv1_3.pdf](references/SAFRSystemAPIsv1_3.pdf) for system topology, deployment endpoints, and older cross-service workflow examples. The PDF is a version 1.3 guide and the YAML contracts describe different service versions. Therefore:

- Use the selected YAML for paths, methods, parameters, headers, bodies, media types, response schemas, and documented privileges.
- Use this skill's environment table, derived from the PDF, for standard cloud base URLs.
- Treat PDF request examples as workflow context. If an example conflicts with a YAML operation, follow the YAML for the current HTTP contract and identify any unresolved deployment-specific behavior.
- Never assume that a path documented for one service exists on another service.

All four YAML contracts declare `servers: []`; they do not supply usable environment hosts. Do not derive a base URL from an OpenAPI default or reuse a host merely because two services expose similar paths.

## Select the environment and base URL

For a live request, resolve both the service and target environment. If the target is missing, ask whether to use `dev`, `int2`, `prod`, or a custom endpoint. For a sample or reusable client, leave the environment configurable instead of choosing one silently. Never default to production.

| Service | `dev` | `int2` (pre-production / Partner Cloud) | `prod` |
| --- | --- | --- | --- |
| COVI | `https://covi.dev.real.com` | `https://covi.int2.real.com` | `https://covi.real.com` |
| CVEV | `https://cv-event.dev.real.com` | `https://cv-event.int2.real.com` | `https://cv-event.real.com` |
| VIRGA (VRGA) | `https://virga.dev.real.com` | `https://virga.int2.real.com` | `https://virga.real.com` |
| CVOS | `https://cvos.dev.real.com` | `https://cvos.int2.real.com` | `https://cvos.real.com` |

The `dev` hosts follow the user's explicit rule: take the corresponding `int2` host and replace only the `int2` DNS label with `dev`. The production hosts omit the environment label. The documentation page for a standard cloud service is `<base-url>/docs/index.html`; do not append that documentation path to API requests.

Always allow a custom endpoint. If the user selects `custom` or describes an on-premises/private deployment, ask for the full base URL for each involved service. A custom URL may include a path prefix, so preserve it. Do not transform a custom hostname using the `int2`/`dev` rule. Confirm that it is the intended trusted deployment before sending credentials.

The PDF records these historical local defaults; treat them as candidates to confirm, not automatic choices:

| Service | HTTP | HTTPS |
| --- | --- | --- |
| COVI | `http://localhost:8080/covi-ws` | `https://localhost:8081/covi-ws` |
| CVEV | `http://localhost:8082` | `https://localhost:8083` |
| VIRGA (VRGA) | `http://localhost:8084` | `https://localhost:8085` |
| CVOS | `http://localhost:8086` | `https://localhost:8087` |

Require HTTPS for remote deployments. Permit HTTP only when the user identifies a trusted local deployment. Keep TLS certificate verification enabled unless the user explicitly provides a deployment-specific reason and authorization to change it.

Prefer service-specific configuration such as `SAFR_COVI_BASE_URL`, `SAFR_CVEV_BASE_URL`, `SAFR_CVOS_BASE_URL`, and `SAFR_VIRGA_BASE_URL`. Use a generic `SAFR_BASE_URL` only when the service is unambiguous. In a cross-service workflow, resolve a base URL for every service rather than reusing one service's host.

Construct a request URL as:

```text
<selected-base-url-without-trailing-slash><path-exactly-as-declared-in-the-selected-yaml>
```

## Resolve the operation

Before constructing a request or client method:

1. Choose the service by responsibility, then find the operation by path, `operationId`, summary, or tag in its YAML.
2. Read the entire operation, including its description, security declaration, required privilege, parameters, request body, and responses.
3. Resolve every referenced parameter, request body, and schema under `components`.
4. Use only declared fields, enum values, query parameters, headers, and media types.
5. Re-check the contract instead of copying a PDF or Swagger UI example when they differ.

Useful anchors for common workflows include:

- COVI `POST /people` for image recognition and optional identity insertion; `GET /people`, `GET /rootpeople`, and `/people/{personId}` for identity records.
- CVEV `GET /events` for event queries and `GET /event/status` for long polling. Event time filters are epoch milliseconds and select events overlapping the requested window; add-date filters have different archive-ingestion semantics.
- CVOS `GET /person/{personId}/face` for an enrolled person's reference image. Event captures instead use `GET /obj/{base64(eventId)}/face` or `/sceneThumb`, if the client was configured to store them; URL-encode the Base64 path value and handle the result as binary.
- VIRGA `GET /status` reads worker status. `POST /status` is a worker-only status update requiring `X-CLIENT-ID`, `X-CLIENT-TYPE`, a body, and the other declared inputs. Ask which action is intended when “status request” is ambiguous. Use `/config/...`, `/task...`, and `/zone...` for the other video-worker administration workflows.

Some workflows cross services. For example, query identities through COVI and fetch their images through CVOS, or query events through CVEV and retrieve associated stored objects through CVOS. Validate each leg against its own contract and base URL.

For “events from the last N seconds,” capture one current epoch-millisecond value and derive both bounds from it. Decide whether the user means event-time overlap or archive-add time. Follow the CVEV paging object until complete when the request says “all” or “each.” For event long polling, a timeout or HTTP `204` is a normal no-change result; on `200`, follow the PDF workflow while validating `sinceModDate` and `lastModDate` against the current CVEV schema.

The imported YAML files contain conversion artifacts. Treat them as reasons to verify, not permission to invent request details:

- `Authentication: [global]` on an API-key scheme means the credential header is required; it does not define an OAuth scope.
- VIRGA paths containing literal `**` do not declare the concrete route segment. Obtain the deployment's exact path before calling one.
- CVOS path casing is significant: `/sharedStream/{id}` and `/sharedstream/{id}` are distinct operations.
- A few CVOS operations declare required path parameters absent from the path template. Do not append invented segments; compare the sibling operation and confirm the deployed route.

## Authenticate and select a directory

The four contracts define the API-key header `X-RPC-AUTHORIZATION`. Treat the user ID as an opaque SAFR identifier, not an email address, and preserve it exactly.

The imported sources disagree about password representation:

- The COVI contract and the PDF's direct HTTP examples use `X-RPC-AUTHORIZATION: <userId>:<password>`.
- The CVEV, CVOS, and VIRGA Swagger introductions tell users of “Try it out” to enter `<userId>:<base64(password)>`, while the PDF's direct HTTP examples for those services show the raw password.

For COVI, use the contract's raw `<userId>:<password>` form unless the deployment says otherwise. For CVEV, CVOS, or VIRGA, prefer a prebuilt authorization value or a known deployment convention. If only a user ID and password are available and the convention is unknown, ask whether that deployment expects a raw or Base64-encoded password before constructing or sending the header. Never Base64-encode the entire `userId:password` pair unless deployment documentation explicitly requires it.

Use `X-RPC-DIRECTORY` when the selected YAML operation declares it. The PDF's older examples use `Authorization: main` as the directory header; do not copy that legacy header name unless the target deployment explicitly requires it. Use `main` only when its documented default is appropriate or the caller chooses it. For a custom deployment or live destructive request, obtain or explicitly confirm the directory rather than silently accepting `main`. Do not confuse either directory header with `X-RPC-AUTHORIZATION`.

Do not infer an extra gateway `Authorization` header from generic 401/403 text. Add proxy or gateway credentials only when the user or deployment configuration supplies that requirement.

## Protect credentials and sensitive data

Obtain credentials at runtime from protected environment variables or a secret manager. Never commit credentials, embed them in generated code, or expose them in commands, logs, diffs, examples, error reports, or final responses. Display the credential header only as:

```http
X-RPC-AUTHORIZATION: <redacted>
```

Do not enable shell tracing while secrets are in scope. Treat face images, signatures, identity records, recognition events, access clearances, and stored scene images as sensitive. Confirm that uploads and access are authorized, minimize returned personal data, and avoid persisting response bodies unless required.

Runnable examples may reference a protected runtime variable such as `${SAFR_AUTHORIZATION}`, but must never contain or print its literal value. Treat the bundled system guide, which is marked RealNetworks Confidential, as internal material and do not reproduce large excerpts.

Never reuse production credentials against `dev`, `int2`, or a custom host merely because the authentication header has the same name.

## Construct and execute requests safely

Distinguish explaining or generating a request from actually sending it. Do not call an API when the user asks only for documentation, sample code, or a dry run.

For every request:

- Preserve the declared method and path, URL-encode path and query values, and include all required parameters.
- Use the operation's declared content type. Send binary images with binary-safe handling, not text conversion or multipart encoding unless the operation explicitly requires multipart.
- Serialize structured bodies against the referenced schema and omit invented properties.
- Check the operation description for required read, write, delete, configuration, access, or super-user privileges.
- Keep the base URL, authorization value, and directory injectable in reusable code and redact secrets in instrumentation.

Treat COVI `POST /people` as mutating by default: its documented defaults include `insert=true`, `update=true`, and `merge=true`. For recognition without directory changes, explicitly use:

```text
insert=false&update=false&merge=false
```

Explain those flags before execution and confirm that no other selected parameter causes a write.

Do not silently add COVI's face-quality bypass parameters or change its face-grouping threshold; both materially affect recognition quality and false-positive risk. For `DELETE /people/{personId}`, explain and explicitly choose whether `recursive=true` should remove all people merged under the root identity and whether `async=true` is acceptable.

Before a destructive or high-impact request, verify the environment, service, method, exact path, directory or tenant, target identifiers, and expected state change. Obtain confirmation before deletion, identity merge or unmerge, user/role/tenant changes, system configuration changes, access-control changes, or any production mutation whose scope was not already explicit. Do not blindly retry a mutation whose outcome is unknown. Retry only safe or confirmed-idempotent operations with bounded backoff.

## Handle and report results

Parse the response using its declared media type and schema. Preserve useful SAFR error fields while redacting credentials and unnecessary personal data.

- For `400`, compare the parameters, content type, and body with the selected operation.
- For `401`, verify the selected environment and service host, exact credential-header spelling, and target deployment's credential representation without displaying it. Do not repeatedly retry rejected credentials.
- For `403`, identify the privilege named by the operation; do not work around authorization controls.
- For `404`, verify that the path belongs to the selected service and check the base-path prefix, directory, and identifier.
- For `409`, inspect the documented conflict condition rather than retrying unchanged.

Report the service and operation, environment or custom host, directory when applicable, HTTP status, relevant result or error, whether state changed, and any missing privilege or follow-up. Never include credentials in the report.
