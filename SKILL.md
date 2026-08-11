---
name: safr-api
description: Use when integrating with, querying, operating, or troubleshooting the SAFR Computer Vision REST API, including face and people recognition, verification, signatures, groups, objects, directories, users, roles, tenants, access clearances, invitations, and zones. Use for constructing requests, generating client code, interpreting responses, or diagnosing SAFR authentication and privilege errors. Not for generic computer-vision work that does not use the SAFR API.
---

# SAFR API

Use this skill to work safely and accurately with the SAFR Computer Vision REST API.

## Read the API contract

Treat [references/openapi.yml](references/openapi.yml) as the authoritative contract for paths, methods, parameters, request bodies, content types, response schemas, and privilege requirements. The bundled contract describes OpenAPI 3.0 and SAFR API version `3.21.1022`.

Before constructing a request or client method:

1. Find the requested operation by path, `operationId`, or `summary` in the OpenAPI file.
2. Read the complete operation, including its description and required privileges.
3. Resolve every referenced parameter, request body, and schema under `components`.
4. Use only fields, enum values, media types, and query parameters declared by the operation.
5. Re-check the contract instead of relying on examples when they disagree.

Use `rg` to navigate the specification when a YAML-aware OpenAPI tool is unavailable. Useful searches include the endpoint path, `operationId`, `summary:`, `requestBody:`, and the referenced component name.

## Resolve the server

Obtain the SAFR base URL from the user, project configuration, or an existing environment variable such as `SAFR_BASE_URL`. The OpenAPI document has an empty `servers` array, so never invent a hostname, port, or URL prefix.

Normalize request URLs as:

```text
<base-url-without-trailing-slash><path-from-openapi>
```

Require HTTPS unless the user explicitly identifies a trusted local development deployment. Keep TLS certificate verification enabled; do not add an insecure TLS option automatically.

## Authenticate every request

Send this header on every SAFR API request:

```http
X-RPC-AUTHORIZATION: <userId>:<password>
```

Join the user ID and password with one colon. Send the resulting value directly; do not Base64-encode it, prefix it with `Basic` or `Bearer`, or substitute a standard `Authorization` header.

Treat `<userId>` as an opaque SAFR user identifier, not as an email address. Obtain the exact user ID from SAFR administration or runtime configuration; do not derive it from an email address, substitute an account email, or apply email-specific parsing, validation, or normalization. Preserve the supplied user ID unchanged when constructing the header.

Treat the OpenAPI `components.securitySchemes.Authentication` definition and root-level `security` requirement as applying to every operation, including operations that do not list an authentication parameter locally.

Handle the specification's naming inconsistency carefully:

- `components.parameters.XRpcAuthorization` actually defines the required `X-RPC-DIRECTORY` header.
- `components.securitySchemes.Authentication` defines the `X-RPC-AUTHORIZATION` credential header.
- Never place a directory name in `X-RPC-AUTHORIZATION`.
- Send `X-RPC-DIRECTORY` separately when an operation references that parameter or when directory selection is required. Use the caller's directory; use the documented `main` default only when it is appropriate for the target deployment.

If a deployment requires an additional proxy or gateway `Authorization` header, obtain that requirement and value from the user or deployment configuration. Do not infer one from the generic 401/403 response text in the OpenAPI file.

## Protect credentials and biometric data

Obtain credentials at runtime from a secret manager, protected environment variables, or an equivalent secure facility. Prefer names such as:

```text
SAFR_BASE_URL
SAFR_USER_ID
SAFR_PASSWORD
SAFR_DIRECTORY
```

Never commit credentials, write them into generated source code, place them in `SKILL.md`, or include their values in logs, diffs, examples, error reports, or final responses. Redact the authorization value as `X-RPC-AUTHORIZATION: <redacted>` when displaying a request.

Do not enable shell tracing while credentials are in scope. Do not write the combined authorization value to a temporary file. If the password contains a colon and the deployment's parsing behavior is not known, stop and ask how that deployment escapes or represents the credential.

Treat face images, signatures, identity records, access clearances, and recognition results as sensitive data. Confirm that the requested use and upload are authorized. Minimize returned personal data and avoid persisting response bodies unless the task requires it.

## Select the operation family

Use the OpenAPI contract to choose among these main areas:

- Use `/people`, `/verification`, `/signature`, `/rootpeople`, `/mergedpeople`, `/merge`, `/mergerecursive`, and `/unmerge` for face and person workflows.
- Use `/persontypes`, `/homelocations`, `/groups`, and `/group/...` for person classification and group membership.
- Use `/objects` and `/objecttypes` for object recognition and object records.
- Use `/admin/...` and `/config/...` for directories, roles, users, credentials, system configuration, tenants, usage, and limits.
- Use `/access/...`, `/invite/...`, `/peoplezones`, and the zone-count endpoints for clearances, invitations, and zone membership.

Do not guess which endpoint fits an ambiguous request. Present the plausible operations and ask for the missing identity, directory, desired state, or scope when choosing incorrectly could modify data.

## Classify side effects before execution

Distinguish between explaining or generating a request and actually sending it. Do not call the API when the user only asks for documentation, sample code, or a dry run.

Before sending a mutating request:

1. Identify the exact method, path, tenant or directory, and target identifiers.
2. State the expected state change.
3. Check the operation description for the required privilege.
4. Obtain confirmation before deleting records, merging or unmerging people, modifying users or roles, changing tenant or system configuration, altering license limits, or changing access controls.
5. Avoid automatic retries unless the operation is known to be idempotent and the request outcome is known.

Treat `POST /people` as mutating by default. Its documented defaults include `insert=true`, `update=true`, and `merge=true`. For detection or identification without database changes, explicitly send:

```text
insert=false&update=false&merge=false
```

Explain those flags before execution and verify that no other selected parameter introduces a write. Require the appropriate write privilege whenever insertion, update, merge, or another write behavior is enabled.

Treat all `DELETE` operations as destructive. Re-read the target immediately before sending the request and report the identifiers that will be affected.

## Construct requests exactly

Build every request from the selected OpenAPI operation:

- Preserve the documented HTTP method and path.
- Substitute and URL-encode all path parameters.
- Include required query and header parameters.
- Use the documented request media type. Face and object image operations commonly accept `image/jpeg` or `application/octet-stream`; structured operations use their declared JSON media type.
- Send binary files without text conversion or multipart encoding unless the operation explicitly requires multipart data.
- Serialize JSON bodies against the referenced schema and omit invented properties.
- Set `Accept` to a documented response media type when the caller needs a specific representation.

Use environment-variable placeholders in examples. A representative authenticated read request is:

```bash
: "${SAFR_BASE_URL:?Set SAFR_BASE_URL}"
: "${SAFR_USER_ID:?Set SAFR_USER_ID}"
: "${SAFR_PASSWORD:?Set SAFR_PASSWORD}"
: "${SAFR_DIRECTORY:?Set SAFR_DIRECTORY}"

curl --fail-with-body --silent --show-error \
  --request GET \
  --header "X-RPC-AUTHORIZATION: ${SAFR_USER_ID}:${SAFR_PASSWORD}" \
  --header "X-RPC-DIRECTORY: ${SAFR_DIRECTORY}" \
  --header "Accept: application/json" \
  "${SAFR_BASE_URL%/}/people/${PERSON_ID}"
```

A representative non-persisting face request is:

```bash
curl --fail-with-body --silent --show-error \
  --request POST \
  --header "X-RPC-AUTHORIZATION: ${SAFR_USER_ID}:${SAFR_PASSWORD}" \
  --header "X-RPC-DIRECTORY: ${SAFR_DIRECTORY}" \
  --header "Content-Type: image/jpeg" \
  --header "Accept: application/json" \
  --data-binary "@${IMAGE_PATH}" \
  "${SAFR_BASE_URL%/}/people?insert=false&update=false&merge=false"
```

Adapt these examples only after reading the selected operation. Do not assume that all operations accept JSON, require `X-RPC-DIRECTORY`, or return the same response type.

## Handle responses and errors

Capture the HTTP status and parse the response using its documented media type and schema. Preserve useful SAFR error fields such as `error`, `errorReason`, `message`, and `path`, while redacting credentials and unnecessary personal data.

Handle common failures as follows:

- For `400`, compare parameters, content type, and body against the operation and referenced schema.
- For `401`, verify the base URL and presence and exact spelling of `X-RPC-AUTHORIZATION`; then verify the runtime user ID and password without displaying them. Do not repeatedly retry invalid credentials.
- For `403`, inspect the operation description for required privileges such as read, write, delete, configuration, account, access, or super-user privileges. Do not work around authorization controls.
- For `404`, verify the path, directory, and target identifier.
- For `409`, inspect the operation's documented conflict conditions and correct the request rather than retrying it unchanged.
- For transient server or network failures, retry only safe or confirmed-idempotent operations with bounded backoff. Never blindly retry a mutation whose result is unknown.

## Report the result

Return or record:

1. The operation ID or method and path used.
2. The target base URL host and directory, without credentials.
3. The HTTP status.
4. The relevant result or error summary.
5. Whether the request changed state.
6. Any required manual follow-up or missing privilege.

When generating reusable client code, keep the base URL, user ID, password, and directory injectable through configuration. Centralize construction of `X-RPC-AUTHORIZATION`, redact it in instrumentation, and add tests that assert the header is present without snapshotting its value.
