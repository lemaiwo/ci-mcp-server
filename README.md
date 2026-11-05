# CI MCP Server

An MCP (Model Context Protocol) server for SAP Cloud Integration (CPI), powered by [odata-mcp-proxy](https://www.npmjs.com/package/odata-mcp-proxy). It exposes CPI OData APIs as MCP tools, allowing AI assistants like Claude to manage your integration landscape through natural language.

The entire server is defined through a single JSON config file -- no custom code required.

## How It Works

This project uses the `odata-mcp-proxy` npm package, which maps OData/REST services to MCP tools based on a configuration file. You provide a config describing your APIs and entity sets, and the proxy generates the corresponding MCP tools automatically.

```
AI Assistant (Claude, Cursor, etc.)
        |
        | MCP Protocol (HTTP or stdio)
        v
  odata-mcp-proxy
        |
        | REST + OAuth2 (via BTP Destination Service)
        v
  SAP Cloud Integration OData API
```

Think of it like the [SAP Application Router](https://www.npmjs.com/package/@sap/approuter) -- a ready-made runtime you configure, not code you write.

## Exposed CPI APIs

The config file (`ci-api-config.json`) exposes the SAP Cloud Integration OData API, organized into the following categories:

### Integration Content

| Tool | Operations | Description |
|------|-----------|-------------|
| `IntegrationPackages` | list, get, create, update, delete | Logical containers that group iFlows, value mappings, and other design-time artifacts |
| `IntegrationDesigntimeArtifacts` | list, get, create, update, delete | iFlow design-time definitions (editable integration logic before deployment) |
| `IntegrationRuntimeArtifacts` | list, get | Deployed integration artifacts (deployment status, version, and errors) |
| `ValueMappingDesigntimeArtifacts` | list, get, create, update, delete | Lookup tables that translate codes/identifiers between sender and receiver systems |
| `MessageMappingDesigntimeArtifacts` | list, get, create, update, delete | Graphical structure-to-structure transformations between message formats |
| `ScriptCollectionDesigntimeArtifacts` | list, get, create, update, delete | Reusable Groovy or JavaScript libraries shared across iFlows |
| `CustomTagConfigurations` | list, get, create, update, delete | Tenant-level labels for categorizing and filtering integration packages |
| `BuildAndDeployStatus` | list, get | Track whether an iFlow deployment is queued, running, or finished |

### Message Processing Logs

| Tool | Operations | Description |
|------|-----------|-------------|
| `MessageProcessingLogs` | list, get | Execution history for iFlows, used to debug failed messages or monitor processing |
| `IdMapFromId2s` | list | ID mapping entries for exactly-once processing (source-to-target ID mappings) |
| `IdempotentRepositoryEntries` | list | Duplicate-check records ensuring a message is processed only once |

### Message Stores

| Tool | Operations | Description |
|------|-----------|-------------|
| `DataStoreEntries` | list, get, delete | Key-value records persisted by iFlows for cross-message data sharing |
| `Variables` | list, get | Runtime variables persisted between iFlow executions (timestamps, counters, delta tokens) |
| `NumberRanges` | list, get | Auto-incrementing counters for generating unique sequence numbers |
| `MessageStoreEntries` | list, get | Full messages persisted via the Persist step for later retrieval or retry |
| `JmsBrokers` | list, get | Messaging broker instances provisioned on the tenant |
| `JmsResources` | list | Individual JMS message queues with depth, capacity, and consumer status |

### Log Files

| Tool | Operations | Description |
|------|-----------|-------------|
| `LogFiles` | list, get | Tenant-level runtime logs (HTTP, default trace, audit) for troubleshooting |
| `LogFileArchives` | list, get | Compressed historical log bundles available for download |

### Security Content

| Tool | Operations | Description |
|------|-----------|-------------|
| `KeystoreEntries` | list, get, delete | SSL/TLS certificates, key pairs, and trusted CA certificates |
| `CertificateResources` | list, get | Full X.509 certificate chains for verifying trust paths |
| `SSHKeyResources` | list, get | Public/private key pairs for SFTP adapter connectivity |
| `UserCredentials` | list, get, create, update, delete | Stored username/password pairs for basic-auth connections |
| `OAuth2ClientCredentials` | list, get, create, update, delete | Client ID/secret pairs and token endpoints for OAuth2 connections |
| `SecureParameters` | list, get, create, update, delete | Encrypted key-value entries for sensitive configuration values |
| `CertificateUserMappings` | list, get, create, update, delete | Rules mapping inbound client certificates to CPI user roles |
| `AccessPolicies` | list, get, create, update, delete | Fine-grained authorization rules for integration artifacts |

### Partner Directory

| Tool | Operations | Description |
|------|-----------|-------------|
| `Partners` | list, get, create, update, delete | Trading partner entries driving dynamic iFlow routing |
| `StringParameters` | list, get, create, update, delete | Partner-specific text configuration values (endpoints, format codes) |
| `BinaryParameters` | list, get, create, update, delete | Partner-specific file-based configuration (XSLT, certificates, mappings) |
| `AlternativePartners` | list, get, create, update, delete | Additional partner identifiers (DUNS, GLN) mapping to a primary partner |
| `AuthorizedUsers` | list, get, create, update, delete | Users permitted to send messages on behalf of a specific partner |

### B2B Monitoring

The B2B monitor of Trading Partner Management (API `B2BScenarios` in the `CloudIntegrationAPI` package). Read only.

| Tool | Operations | Description |
|------|-----------|-------------|
| `BusinessDocuments` | list, get | B2B interchanges with partners, message types, control numbers and status; navigation to error details, events, payload entries and acknowledgements |
| `OrphanedInterchanges` | list, get | Interchanges that no TPM agreement matched |
| `TechnicalAcknowledgements` | get | Technical acknowledgements (AS2 MDN, CONTRL) |
| `FunctionalAcknowledgements` | get | Functional acknowledgements (997, CONTRL) |
| `B2BArchivingConfigurations` | get | B2B data archiving configuration of the tenant |
| `B2BArchivingKeyPerformanceIndicators` | list | Key figures of the B2B archiving runs |

All `_list` tools support OData query parameters: `$filter`, `$select`, `$expand`, `$orderby`, `$top`, `$skip`.

## Prerequisites

- **Node.js** 18+ (20+ recommended)
- **SAP BTP account** with a Cloud Foundry environment
- **SAP Cloud Integration** tenant (part of SAP Integration Suite)
- **BTP Destination** configured for the CPI OData API with OAuth2 authentication
- **Cloud Foundry CLI** (`cf`) and **MBT Build Tool** (`mbt`) for deployment

## Project Structure

```
ci-mcp-server/
├── package.json              # Start script + odata-mcp-proxy dependency
├── ci-api-config.json        # API configuration (defines all MCP tools)
├── mta.yaml                  # BTP Cloud Foundry deployment descriptor
├── xs-security.json          # XSUAA OAuth2 configuration
├── default-env.json          # Local dev credentials (gitignored)
└── LICENSE
```

## Getting Started

### 1. Install dependencies

```bash
npm install
```

### 2. Configure BTP destination

Create a BTP Destination pointing to the CPI OData API:

| Destination | URL |
|-------------|-----|
| `CPI_DESTINATION` | `https://<tenant>.it-cpi0<xx>.cfapps.<region>.hana.ondemand.com` |

The destination should use OAuth2 client credentials authentication with the CPI service key credentials.

### 3. Local development

Create a `default-env.json` with your BTP service bindings (XSUAA, Destination, Connectivity) to run locally:

```bash
npm start
```

This runs `odata-mcp-proxy --config ci-api-config.json`.

### 4. Deploy to BTP

```bash
npm run build:btp     # Build MTA archive
npm run deploy:btp    # Deploy to Cloud Foundry
```

The MTA deployment provisions three service instances:
- **Destination** (lite) -- resolves the CPI API endpoint and manages OAuth2 tokens
- **Connectivity** (lite) -- enables secure backend connectivity
- **XSUAA** (application) -- handles OAuth2 authentication with role-based access control

## Security

The XSUAA configuration (`xs-security.json`) defines three role templates:

| Role | Scopes | Description |
|------|--------|-------------|
| `MCPViewer` | read | Read-only access to CPI data |
| `MCPEditor` | read, write | Read and modify CPI data |
| `MCPAdmin` | read, write, admin | Full administrative access |

OAuth2 redirect URIs are pre-configured for Claude.ai, Cursor, Microsoft Teams, and local development.

## License

MIT
