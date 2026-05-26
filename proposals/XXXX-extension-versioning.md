# SEP-XXXX: Extension Versioning

- **Status**: Draft
- **Type**: Standards Track
- **Created**: 2026-05-26
- **Author(s)**: Luca Chang (@LucaButBoring)
- **Sponsor**: None
- **PR**: TBD

## Abstract

This document defines a versioning convention for MCP extensions. It specifies how extension versions are declared, negotiated, and reported in errors. The mechanism mirrors standard MCP protocol version negotiation, adjusted only where the extension model imposes structurally different constraints.

## Motivation

Why is this change needed? Why is the current protocol specification inadequate to address the problem that this SEP solves?

The motivation is critical for SEPs that want to change the Model Context Protocol. SEP submissions without sufficient motivation may be rejected outright.

## Specification

### Capability Negotiation

Clients **SHOULD** declare the extension version they are targeting in the `_meta` field of each request, scoped under the extension's namespace within their client capabilities as a `version` field. For backwards-compatibility with clients published before this specification, a missing extension `version` **MUST** be interpreted by the server as the client supporting the initial version of the extension.

**Example Extension Request:**

```jsonc
{
  "method": "tasks/get",
  "params": {
    "taskId": "abc-123",
    "_meta": {
      // other metadata...
      "io.modelcontextprotocol/protocolVersion": "DRAFT-2026-v1",
      "io.modelcontextprotocol/clientCapabilities": {
        // other capabilities...
        "extensions": {
          "io.modelcontextprotocol/tasks": {
            "version": "2026-06-30"
          }
        }
      }
    }
  }
}
```

Servers **MUST** advertise supported extension versions in the `server/discover` response, scoped under the extension's namespace within their server capabilities as a `versions` array. The `versions` array lists every version of the extension the server supports. For backwards-compatibility with servers published before this specification, a missing `versions` list **MUST** be interpreted by the client as the server supporting the initial version of the extension.

**Example `server/discover` Response:**

```jsonc
{
  "supportedVersions": ["2025-06-18", "2025-11-25", "DRAFT-2026-v1"],
  "capabilities": {
    // other capabilities...
    "extensions": {
      "io.modelcontextprotocol/tasks": {
        "versions": ["2026-06-30", "2026-10-15"]
      }
    }
  }
}
```

### Extension Version Negotiation

Extension version negotiation mirrors [protocol version negotiation](https://modelcontextprotocol.io/specification/draft/basic/lifecycle#protocol-version-negotiation):

1. The client sends a request with its preferred extension version in its client capabilities, under `extensions.<id>.version`.
2. If the server supports that version, it processes the request normally.
3. If the server does not support the requested version, it **MUST** return an [UnsupportedExtensionVersionError](#unsupported-extension-version) (`-32005`) containing the server's supported versions for that extension.
4. Upon processing the error, the client **MUST** select a mutually-supported version from the list and retry the request. If it is unable to support any version advertised by the server, the client **MUST** abort the request.

Alternatively, a client **MAY** call `server/discover` first to learn the server's supported extension versions before sending any extension-targeted request.

Servers **SHOULD** permit extension version mismatches if the operation requested by the client does not leverage the violating extension capability.

If the request fails both extension version negotiation and protocol version negotiation, the server **MUST** return `UnsupportedProtocolVersionError` (`-32004`). Protocol version errors take precedence because extension behavior is undefined under unsupported protocol versions.

#### Unsupported Extension Version

The server **MUST NOT** rely on extension behaviors introduced or removed via breaking changes between extension versions. If processing a request requires an extension version the client did not declare in its extension capabilities, the server **MUST** return a JSON-RPC error specifying the supported extension versions. For HTTP, the response status code **MUST** be `400 Bad Request`.

```ts
export const UNSUPPORTED_EXTENSION_VERSION = -32005;

export interface UnsupportedExtensionVersionError extends Omit<
  JSONRPCErrorResponse,
  "error"
> {
  error: Error & {
    code: typeof UNSUPPORTED_EXTENSION_VERSION;
    data: {
      /**
       * The extension versions supported by the server for
       * servicing this request, namespaced by extension ID.
       */
      extensions: Record<string, {
        /**
         * The extension version included in the client request
         */
        requested: string[];
        /**
         * The extension versions supported by the server for this extension.
         */
        supported: string[];
      }>;
    };
  };
}
```

A single request may target functionality from multiple extensions (e.g., a request that uses capabilities from both Tasks and another extension). Each extension's version is declared independently within the client capabilities. If multiple extensions fail version negotiation simultaneously, the server **SHOULD** report all failures in a single error response:

```jsonc
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32005, // UnsupportedExtensionVersionError
    "message": "Extension version(s) not supported",
    "data": {
      "extensions": {
        "io.modelcontextprotocol/tasks": {
          "requested": "1900-01-01",
          "supported": ["2026-06-30", "2026-10-15"]
        },
        "io.modelcontextprotocol/ui": {
          "requested": "1900-01-01",
          "supported": ["2026-01-26", "2026-10-15"]
        }
      }
    }
  }
}
```

### Subscription Version Pinning

When a client issues a `subscriptions/listen` request that includes extension-scoped notification types, the extension version declared in `extensions.<id>.version` at the time of the `subscriptions/listen` request governs the notification schema for the lifetime of that subscription.

The server **MUST NOT** send notifications that do not conform to the extension version the client declared. If the server drops support for the pinned version while the subscription is active, it **MUST** [close the subscription](https://modelcontextprotocol.io/seps/2575-stateless-mcp#stopping-a-subscription).

### Error Handling

Servers **MUST** return standard JSON-RPC errors for the following protocol error cases:

- One or more unsupported extension versions in client request: `-32005` (Unsupported Extension Version)

Servers **SHOULD** provide informative error messages to describe the cause of errors.

## Rationale

### Undefined Version Identifier Format

Following the convention set by the base specification, this proposal does not define a particular format for extension versions. This proposal prioritizes maintaining identical semantics between protocol versions and extension versions over introducing a bespoke versioning scheme such as semantic versions. The author of this proposal recommends updating the extension version convention introduced in this proposal if the protocol versioning scheme is changed.

### Dedicated Error Code for Extension Version Mismatches

A dedicated error code allows application-level error handling without ambiguities. `-32003` (Missing Required Client Capability) was considered for reuse, but its payload describes specific client capabilities that must be satisfied for processing a request, and extension version errors can be resolved by using any supported version of a particular extension rather than requiring one particular version.

## Backward Compatibility

This proposal is backwards-compatible with the existing specification. Extensions that do not declare a version are assumed to be the initial extension version for backwards-compatibility purposes. Servers that do declare extension versions negotiate to an explicit error with clients that do not, ensuring that updated servers do not return malformed responses to base-version clients.

The guidance in this proposal does not supersede SEP-2133's existing [backwards-compatibility](https://modelcontextprotocol.io/seps/2133-extensions#backward-compatibility) requirements, although it does interpret them generously: While "breaking changes within an extension **MUST** use a new extension identifier," breaking changes between extension _versions_ are gated on the version negotiation flow introduced by this proposal. A versioned extension only makes breaking changes [as defined by SEP-2133](https://modelcontextprotocol.io/seps/2133-extensions#definition) if it cannot gate those changes as described in this proposal.

## Security Implications

This proposal introduces no new security implications. Security implications associated with cross-version compatibility gaps are owned by individual extensions.

## Reference Implementation

To be provided.
