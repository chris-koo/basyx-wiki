# AAS Registry

![GitHub](https://img.shields.io/github/license/eclipse-basyx/basyx-go-components)

The BaSyx AAS Registry implements the Asset Administration Shell Registry Service. It stores AAS Descriptors and the Submodel Descriptors associated with a registered AAS so that clients can discover where AASs and their submodels are available.

The Registry manages descriptive and routing metadata. It does not store or serve the AAS or Submodel content itself.

## What Is an AAS Descriptor?

An AAS Descriptor identifies an AAS and describes how clients can reach it. Depending on the deployment and use case, it can contain information such as:

- the AAS identifier and `idShort`;
- asset-identification information;
- endpoints at which the AAS can be accessed;
- Submodel Descriptors associated with that AAS.

A complete list of AAS Descriptor attributes is found in the [specification of the AAS](https://industrialdigitaltwin.io/aas-specifications/IDTA-01002/v3.2/specification/interfaces-payload.html#_assetadministrationshelldescriptor). <br>
A Submodel Descriptor plays the same role for a submodel: it identifies the submodel and advertises endpoints and other discovery metadata. In the AAS Registry, Submodel Descriptors are scoped to their parent AAS Descriptor.

## Registry or Repository?

Use the AAS Registry when a client needs to discover an AAS or determine which service endpoint provides it. Use an AAS Repository when a client needs to store, retrieve, or modify the actual AAS content.

A typical deployment uses both components:

1. An AAS Repository stores and serves the AAS.
2. The AAS Registry stores a descriptor containing the AAS identifier and the Repository endpoint.
3. A client searches the Registry and receives the descriptor.
4. The client follows the advertised endpoint to interact with the AAS in the Repository.

The Registry and Repository do not have to run in the same process or at the same network location. This separation allows one Registry to advertise AASs provided by multiple services or organizations.

## Main Capabilities

The AAS Registry supports management of AAS Descriptors and their associated Submodel Descriptors. It provides operations for listing, creating, retrieving, updating, and deleting descriptors.

It also supports:

- cursor-based pagination for descriptor collections;
- filtering AAS Descriptors by asset-related properties and change timestamps;
- structured queries for more expressive descriptor searches;
- asynchronous bulk creation, update, and deletion of AAS Descriptors;
- service self-description.

See [API Documentation](#api-documentation) for the authoritative list of operations, parameters, schemas, and responses.

## Important Behavior

### AAS-scoped Submodel Descriptors

Submodel Descriptors in this component belong to an AAS Descriptor. Clients therefore address a Submodel Descriptor in the context of both its AAS identifier and its Submodel identifier. Use the standalone Submodel Registry when Submodel Descriptors must be managed independently of an AAS Descriptor.

### Identifier Encoding

Identifiers used in request paths must be encoded as UTF-8 Base64 URL values. Encode the original identifier with the URL-safe Base64 alphabet before placing it in a path. Do not send an arbitrary AAS or Submodel identifier directly as a path segment, because identifiers can contain characters that have a special meaning in URLs.

When an operation also has a request body, the body continues to contain the original, unencoded identifier. Encoding is required for the path parameter only. Consult Swagger for the encoding requirement of each operation.

### Pagination and Filters

Collection requests use cursor-based pagination. Treat the returned cursor as an opaque value and pass it unchanged when requesting the next page. The `limit` controls the requested page size; it is not an offset.

Asset-related filters help clients narrow discovery results before contacting a Repository. For example:

- `assetKind` limits results by the kind of asset represented by the AAS;
- `assetType` selects descriptors whose asset information uses a particular asset-type value;
- asset-identifier filters locate descriptors associated with known asset identifiers;
- creation and update filters support synchronization workflows that only need descriptors changed after a given time.

Filter syntax, allowed values, and combination rules are part of the API contract and are documented in Swagger.

### Database Schema

The Registry uses PostgreSQL and expects the shared BaSyx database schema to be initialized and migrated by the BaSyx Configuration Service. The Registry validates the schema during startup and does not initialize it itself. See [Setting Up the AAS Registry](setup) for the required startup order.

## Configuration

The AAS Registry loads the shared BaSyx Go configuration through the `-config` command-line option and environment variables. Environment variables override values from the YAML file. The parameters below affect the Registry's request handling, persistence, or supporting services. See [General Configuration](../common/configuration) for environment-variable names and configuration validation.

### HTTP Server and Public URLs

| Parameter | Effect on the AAS Registry |
| --- | --- |
| `server.host` | Address on which the HTTP server listens; also used to construct the server URL in OpenAPI. |
| `server.port` | Listening port; also included in the OpenAPI server URL. |
| `server.contextPath` | Base path for the Registry API, verification, health, Swagger, and OpenAPI endpoints. |
| `server.readHeaderTimeoutSeconds` | Maximum time to read request headers. |
| `server.readTimeoutSeconds` | Maximum time to read the entire request, including its body. |
| `server.writeTimeoutSeconds` | Timeout for writing the response. |
| `server.idleTimeoutSeconds` | Maximum wait for another request on an idle keep-alive connection. |
| `server.shutdownTimeoutSeconds` | Time allowed for in-flight requests to finish during graceful shutdown. |
| `general.externalUrl` | Public base URL used when constructing resource Location headers. If multiple comma-separated URLs are supplied, the first entry is used. Include the externally visible base path. |
| `general.trustProxyHeaders` | Allows forwarded headers from trusted proxies to influence generated request-based URLs and the source IP recorded in extended audit metadata. |
| `general.trustedProxyCIDRs` | Source-address allowlist for proxies permitted to supply those forwarded headers. Used together with `general.trustProxyHeaders`. |

### PostgreSQL Connections

The Registry requires a writer connection. An optional `postgres.reader` connection routes eligible retrieval operations to a separate database endpoint. Writes, mutation checks, schema validation, and asynchronous jobs use the writer. Reader results can lag behind successful writes. Both configured connections must be reachable at startup; a later reader failure does not automatically redirect reads to the writer.

Each parameter below applies under both `postgres` and `postgres.reader`. The reader has its own connection details and pool limits; it does not inherit the writer's values. Omitting the reader configuration makes reads use the writer pool.

| Parameter within either connection | Effect |
| --- | --- |
| `dsn` | Complete PostgreSQL connection string. Mutually exclusive with explicitly configured individual connection fields; pool settings can still be supplied. |
| `host` | Database server address. |
| `port` | Database server port. |
| `user` | Database login role. |
| `password` | Password for that role. |
| `dbname` | Database containing the shared BaSyx schema. |
| `sslmode` | TLS and certificate-verification behavior for the database connection. |
| `sslcert` | Path to the PostgreSQL client certificate. |
| `sslkey` | Path to the corresponding client private key. |
| `sslrootcert` | Path to the CA certificate used to verify the database server. |
| `connectTimeoutSeconds` | Time limit for establishing a database connection. |
| `applicationName` | Application name reported by PostgreSQL for Registry connections. |
| `searchPath` | Schema search path for database sessions. |
| `options` | Additional PostgreSQL startup options. |
| `timezone` | Session time zone. |
| `maxOpenConnections` | Maximum open connections in this pool, per Registry process. |
| `maxIdleConnections` | Maximum idle connections retained in this pool. |
| `connMaxLifetimeMinutes` | Maximum connection lifetime before recycling. |
| `connMaxIdleTimeMinutes` | Maximum idle duration before a connection is recycled. |

### Descriptor Processing, Queries, and Discovery

| Parameter | Effect on the AAS Registry |
| --- | --- |
| `server.strictVerification` | Controls semantic verification during model conversion: `off` skips semantic checks, `permissive` logs violations, and `strict` rejects violations. Structural parsing requirements still apply. |
| `general.supportsSingularSupplementalSemanticId` | Accepts the singular `supplementalSemanticId` field for nested Submodel Descriptors and uses that name in output. The value remains a list of references. When enabled, singular input takes precedence if both spellings are present. |
| `general.enableImplicitCasts` | Allows implicit type casts when simplifying descriptor-query and ABAC filter expressions for database execution. |
| `general.enableDescriptorDebug` | Enables additional descriptor SQL and query-timing diagnostics. Debug records also require a logging level that includes `debug`. |
| `general.bulkBatchLimit` | Limits the number of rows per generated bulk SQL statement. It controls database batching, rather than the number of descriptors allowed in a bulk API request. |
| `general.discoveryIntegration` | Links descriptor asset identifiers to AAS identifiers in the shared database for discovery. Descriptor writes also create a discovery asset link for a supplied `globalAssetId`. A Discovery Service using that database can use these links. |

### Verification Endpoint and Upload Limits

These upload limits affect the Registry's optional `/verify` endpoint, which accepts JSON, XML, and AASX content for verification.

| Parameter | Effect on verification |
| --- | --- |
| `server.verificationEndpointAvailable` | Registers or removes `/verify` and its OpenAPI documentation. |
| `general.uploadMaxSizeBytes` | Maximum verification payload size in bytes. Multipart requests receive a separate allowance for form overhead. |
| `general.aasxMaxPartCount` | Maximum number of non-directory entries in an AASX package. |
| `general.aasxMaxOPCMetadataSizeBytes` | Maximum combined expanded size of AASX package metadata. |
| `general.aasxMaxPartExpandedSizeBytes` | Maximum expanded size of an individual AASX payload part. |
| `general.aasxMaxTotalExpandedSizeBytes` | Maximum combined expanded size of AASX payload parts. |
| `general.aasxMaxThumbnailSizeBytes` | Maximum expanded thumbnail size. |

### Cross-Origin Requests

| Parameter | Effect on browser access |
| --- | --- |
| `cors.allowedOrigins` | Origins allowed to make cross-origin requests. |
| `cors.allowedMethods` | HTTP methods permitted by the CORS policy. |
| `cors.allowedHeaders` | Request headers permitted by the CORS policy. |
| `cors.allowCredentials` | Allows credentials in cross-origin requests. |

### Authentication and Authorization

`abac.enabled` activates both OIDC authentication middleware and attribute-based access control for the protected API router. The OIDC trustlist is loaded as part of this security setup. Health and Swagger endpoints are registered outside that router.

| Parameter | Effect on security |
| --- | --- |
| `abac.enabled` | Enables authentication and policy-based authorization for Registry, description, bulk, and verification requests. |
| `oidc.trustlistPath` | Path to the JSON trustlist defining accepted identity providers and token-validation settings. |
| `abac.modelPath` | Path to the access-rules file used for policy import. Also supplies the policy-file fingerprint for extended audit metadata. |
| `abac.policyFileImport` | Controls startup import of the policy file: `always`, `if_missing`, or `never`. The latter requires an active database policy. |
| `abac.policyScope` | Database namespace in which the Registry stores and loads its authorization policies. |
| `abac.managementApi.enabled` | Exposes protected ABAC policy-management routes and their OpenAPI entries when ABAC is enabled. |

Trustlist entries configure `issuer`, `audience`, `discoveryUrl`, and `scopes` for each provider. `scopeClaims` selects token claims containing OAuth scopes. `claimMappings` uses `target`, `mode`, and `sources` to map provider claims into the `basyx.*` namespace used by authorization rules. Access rules themselves are defined in the policy file or managed through the policy API. See [Security Files](../common/configuration#security-files) for file mounting.

### History and Audit

The Registry records descriptor mutations through the shared history implementation. History settings also govern mutation-coverage checks and the metadata attached to recorded changes.

| Parameter | Effect on descriptor history |
| --- | --- |
| `history.mode` | Selects `off`, `api`, or `audit` history handling. Enabling history records descriptor versions and activates checks for mutation routes without history coverage. |
| `history.fullSnapshotInterval` | Controls the interval between full snapshots, with intermediate versions represented as deltas. |
| `history.immutability` | Selects unguarded history (`none`) or PostgreSQL protection against history updates and deletion (`postgres_guarded`). |
| `history.auditIdentityMode` | Selects `none`, `minimal`, or `extended` identity metadata. Extended records include source IP, user agent, and additional authorization context. |

#### External History Evidence

The following parameters configure evidence artifacts written during Registry mutations. They take effect when evidence storage is enabled.

| Parameter | Effect on evidence storage |
| --- | --- |
| `history.evidence.enabled` | Enables external evidence writing for descriptor mutations. |
| `history.evidence.provider` | Selects the evidence backend; enabled evidence currently requires `s3`. |
| `history.evidence.bucket` | S3 bucket receiving the artifacts. |
| `history.evidence.prefix` | Object-key prefix within the bucket. |
| `history.evidence.region` | S3 region used by the client. |
| `history.evidence.endpoint` | Custom endpoint for S3-compatible storage. |
| `history.evidence.accessKeyId` | Explicit S3 access-key ID, configured together with the secret key. |
| `history.evidence.secretAccessKey` | Secret for the explicit S3 credentials. When neither credential is supplied, the AWS SDK credential chain is used. |
| `history.evidence.pathStyle` | Selects path-style bucket addressing. |
| `history.evidence.retentionMode` | Object Lock retention mode: `governance` or `compliance`. |
| `history.evidence.retentionDays` | Object Lock retention duration for evidence artifacts. |
| `history.evidence.writeTimeoutSeconds` | Time limit for an evidence write. Evidence-write failures can cause the associated mutation to fail. |

Database-history pruning and external integrity anchors are not implemented. `history.retentionDays` and `history.integrityAnchor.provider` therefore do not provide configurable retention or anchoring behavior. Evidence-manifest signing keys are used by the separate evidence verifier; the Registry's mutation path does not use them to sign artifacts.

### Logging and Telemetry

| Parameter | Effect on logging |
| --- | --- |
| `logging.format` | Selects `text` or `json` diagnostic output on standard error. |
| `logging.level` | Minimum emitted severity: `debug`, `info`, `warn`, or `error`. |

OpenTelemetry uses environment variables independently of the YAML configuration. It instruments Registry HTTP requests and database pools.

| Environment variable | Effect on telemetry |
| --- | --- |
| `OTEL_SDK_DISABLED` | Disables the telemetry SDK. |
| `OTEL_TRACES_EXPORTER` | Selects trace export through `otlp` or disables it with `none`. |
| `OTEL_METRICS_EXPORTER` | Selects metric export through `otlp` or disables it with `none`. |
| `OTEL_SERVICE_NAME` | Service identity attached to telemetry. |
| `OTEL_RESOURCE_ATTRIBUTES` | Additional resource attributes attached to telemetry. |
| `OTEL_TRACES_SAMPLER` | Trace sampling strategy. |
| `OTEL_TRACES_SAMPLER_ARG` | Argument for the selected sampling strategy. |
| `OTEL_PROPAGATORS` | Context propagation using `tracecontext`, `baggage`, or `none`. |
| `OTEL_BSP_SCHEDULE_DELAY` | Delay between scheduled trace-batch exports, in milliseconds. |
| `OTEL_BSP_EXPORT_TIMEOUT` | Trace-batch export timeout, in milliseconds. |
| `OTEL_BSP_MAX_QUEUE_SIZE` | Maximum queued spans. |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | Maximum spans per export batch. |
| `OTEL_METRIC_EXPORT_INTERVAL` | Metric export interval, in milliseconds. |
| `OTEL_METRIC_EXPORT_TIMEOUT` | Metric collection and export timeout, in milliseconds. |

Exporter settings use the prefix `OTEL_EXPORTER_OTLP_`. The signal-specific prefixes `OTEL_EXPORTER_OTLP_TRACES_` and `OTEL_EXPORTER_OTLP_METRICS_` override the corresponding shared settings for their signal.

| Suffix for each exporter prefix | Effect |
| --- | --- |
| `ENDPOINT` | Destination for telemetry export. |
| `PROTOCOL` | Export transport: `grpc` or `http/protobuf`. |
| `HEADERS` | Headers sent with export requests. |
| `COMPRESSION` | Export compression setting. |
| `TIMEOUT` | Export request timeout, in milliseconds. |

See [Observability](../common/observability) for telemetry operation and collection.

### Swagger and OpenAPI

| Parameter | Effect on API documentation |
| --- | --- |
| `swagger.enabled` | Enables or disables Swagger UI and the served OpenAPI/schema endpoints. |
| `swagger.contactName` | Contact name inserted into OpenAPI metadata. |
| `swagger.contactEmail` | Contact email inserted into OpenAPI metadata. |
| `swagger.contactUrl` | Contact URL inserted into OpenAPI metadata. |

The shared model also contains settings without an implemented Registry effect. In particular, `server.cacheEnabled` is stored by the Registry backend but does not activate a cache. Repository synchronization settings, AAS preconfiguration, custom middleware header injection, and event-publishing placeholders are not active Registry features.

This scope follows the [AAS Registry startup](https://github.com/eclipse-basyx/basyx-go-components/blob/main/cmd/aasregistryservice/main.go), [Registry persistence](https://github.com/eclipse-basyx/basyx-go-components/blob/main/internal/aasregistry/persistence/PostgreSQLAASRegistryDatabase.go), and the shared configuration, descriptor, security, history, and telemetry implementations they invoke.

## Limitations and Operational Notes

- The Registry stores descriptors, not AAS or Submodel payloads.
- Advertised endpoints should be reachable from the clients that consume the descriptor; container-internal addresses are often unsuitable for external clients.
- Cursor values are implementation-managed and should not be parsed or constructed by clients.
- A configured PostgreSQL reader is eventually consistent, so a descriptor mutation may not be visible immediately through a reader-routed request.
- Exact API behavior, including status codes and validation constraints, is defined by the served OpenAPI document.

## API Documentation

Use the following API documentation depending on whether you need the behavior of a running BaSyx component or the standardized API definition:

- **BaSyx Go Swagger UI** describes the API actually exposed by the running component, including its configured base path and runtime OpenAPI adjustments. It contains the complete operation list, parameters, request and response schemas, status codes, and interactive requests.
- **[IDTA AAS Registry Swagger UI v3.2.0](https://industrialdigitaltwin.io/aas-specs-api/docs/swagger-ui.html?url=..%2FAssetAdministrationShellRegistryServiceSpecification%2FV3.2_SSP-001.yaml&version=v3.2.0)** presents the standardized AAS Registry Full Profile interactively.
- **[IDTA Specification of the Asset Administration Shell, Part 2: Application Programming Interfaces v3.2.0](https://industrialdigitaltwin.io/aas-specifications/IDTA-01002/v3.2/index.html)** is the normative specification for the standardized operations, service specifications, profiles, and serialization behavior on which this component is based.

The OpenAPI document shipped with the current BaSyx Go AAS Registry identifies API version 3.2.0 and declares the AAS Registry Full, Bulk, and Query profiles (`SSP-001`, `SSP-003`, and `SSP-004`). Use the Swagger UI of the running component to determine its actual exposed contract and configuration.

With the default empty context path, the service exposes:

- Swagger UI at `/swagger`;
- the OpenAPI document at `/api-docs/openapi.yaml`.

When `server.contextPath` is configured, both locations are served below that context path. Swagger can also be disabled through configuration. See [Swagger UI Docs](../common/swagger) for details.

## Related Documentation

- [Setting Up the AAS Registry](setup)
- [General Configuration](../common/configuration)
- [Common / Shared Features](../common/shared_features)
- [Swagger UI Docs](../common/swagger)

```{toctree}
:hidden:
:maxdepth: 1

setup
```
