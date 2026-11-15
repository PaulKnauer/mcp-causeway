---
title: mcp-causeway Product Brief
status: draft
created: 2026-09-12
updated: 2026-09-12
---

# Product Brief: mcp-causeway

## Summary

mcp-causeway is a homelab MCP publication layer that makes existing internal HTTP MCP servers safely usable by cloud-hosted AI clients such as Claude, ChatGPT/OpenAI Agents, and Gemini.

The product provides the missing bridge between private k3s-hosted MCP services and vendor-side MCP clients: public reachability, publicly trusted TLS, OAuth/OIDC-based authorization, Streamable HTTP conformance, protected resource metadata, and repeatable documentation/configuration for small multi-host homelabs.

It is not primarily an MCP server implementation. It is a packaging, routing, authentication, and publication pattern for already-running MCP servers.

## Problem

Paul already has multiple MCP-capable services running inside the homelab/k3s environment, including services such as Soniq, MQTT MCP, and Hindsight. These services are internal, HTTP-based, and reachable on the LAN or through existing ingress patterns.

Modern AI clients are moving toward MCP as the tool integration layer, but their connectivity requirements differ:

- Claude/claude.ai connectors require a publicly reachable MCP endpoint and vendor-side network access.
- Gemini custom MCP requires Streamable HTTP and publicly trusted TLS; self-signed certificates are explicitly unsuitable.
- ChatGPT/OpenAI hosted connectors cannot directly reach private homelab networks without a tunnel or environment-side executor.
- OpenAI Agents API now supports MCP directly, but the connection can originate either from OpenAI's service or from an attached execution environment, creating two valid but different deployment paths.

The current homelab state is not enough for broad vendor compatibility:

- Existing ingress is nginx-ingress, not Traefik.
- Current local domains such as `home.lab` / `k3s.home.arpa` use private DNS and self-signed/internal CA assumptions.
- HTTPS is exposed internally via NodePort `30443`, not normal public `443`.
- Some services already support OIDC resource-server behavior, but others are bearer-token-only or lack MCP Protected Resource Metadata.
- Authelia is already deployed with OIDC, but it does not currently support Dynamic Client Registration, which many MCP/OAuth client flows may expect.

Without mcp-causeway, every MCP service would need bespoke publication, TLS, auth, metadata, and vendor-compatibility work.

## Vision

mcp-causeway turns a private homelab MCP ecosystem into a secure, vendor-portable MCP surface.

The long-term vision is a documented reference pattern that other small homelab operators can adapt: take existing internal MCP services, publish them through a hardened gateway layer, protect them with standards-based OAuth/OIDC, and expose them to multiple AI clients without rewriting each MCP server.

The product should be useful in two ways:

1. **As working infrastructure for Paul's homelab** — production-grade enough to connect real assistants to real internal tools.
2. **As documentation-as-product** — clear enough that another technical user can understand the trade-offs, reproduce the pattern, and adapt it to their own k3s or small-cluster environment.

## Target Users

### Primary user

Paul, operating a private homelab/k3s environment with internal MCP services and multiple AI assistants across desktop, cloud, and local environments.

### Secondary users

Technical homelab operators who:

- run services on a private LAN or small k3s cluster;
- want to expose selected MCP tools to hosted AI clients;
- care about TLS, OAuth, and minimal public attack surface;
- prefer portable infrastructure and documented patterns over one-off tunnels.

## Current System Context

Known environment facts from discovery:

- The relevant k3s ingress host is `192.168.2.201` / `rpi4-1`.
- Existing cluster ingress uses nginx-ingress; Traefik is disabled.
- Existing local domains include `home.lab` and `k3s.home.arpa`.
- Existing TLS uses a homelab CA / self-signed trust model.
- Existing MCP services are already HTTP-based; there are currently no day-one stdio MCP servers requiring `mcp-proxy`.
- Soniq already appears capable of OIDC resource-server behavior via issuer/audience/JWKS/resource URL configuration.
- MQTT MCP and Hindsight may need gateway-level OAuth/resource-metadata handling if they remain bearer-token-only.
- Authelia is already deployed with OIDC support but lacks Dynamic Client Registration.

## Product Goals

1. **Publish existing internal MCP servers safely**
   - Expose selected MCP services through a controlled public endpoint.
   - Avoid making every internal service individually public.

2. **Use publicly trusted TLS**
   - Use a real owned domain.
   - Use Let's Encrypt certificates via cert-manager DNS-01.
   - Avoid HTTP-01 or inbound port-forwarding solely for certificate issuance.

3. **Support split-horizon DNS**
   - Keep LAN traffic local where possible.
   - Allow vendor cloud clients to resolve the public endpoint correctly.

4. **Standardize MCP transport compatibility**
   - Prefer Streamable HTTP as the primary exposed transport.
   - Treat SSE compatibility as optional or legacy depending on client needs.

5. **Provide OAuth/OIDC protection**
   - Use Authelia OIDC where viable.
   - Make MCP services behave like OAuth resource servers.
   - Emit or proxy standards-compliant Protected Resource Metadata.

6. **Handle vendor differences explicitly**
   - Document Claude, ChatGPT/OpenAI, and Gemini requirements separately.
   - Support the broadest common compatible path first.

7. **Keep the design portable**
   - Avoid depending on Cloudflare Tunnel as a required component.
   - Avoid product-specific hacks where a standards-based gateway pattern is possible.

8. **Document the pattern thoroughly**
   - Capture architecture, threat model, deployment steps, client setup, and trade-offs.

## Non-Goals

For the MVP, mcp-causeway should not attempt to:

- rewrite existing MCP servers;
- replace the whole homelab ingress stack;
- require Cloudflare Tunnel or another third-party tunnel as the default path;
- expose all internal tools automatically;
- support arbitrary public unauthenticated MCP access;
- prioritize stdio MCP onboarding, since the current day-one services are already HTTP-based;
- build a general-purpose commercial MCP gateway before validating Paul's own homelab use case.

## MVP Scope

The MVP should deliver a public, authenticated, standards-aligned publication path for a small set of existing HTTP MCP services.

### In scope

- Real owned domain for MCP endpoints.
- Let's Encrypt TLS using cert-manager DNS-01.
- Split-horizon DNS design.
- nginx-ingress configuration matching the existing k3s cluster.
- A gateway or ingress-layer pattern for:
  - routing to internal MCP services;
  - enforcing authentication;
  - normalizing MCP endpoint behavior where required;
  - emitting or forwarding Protected Resource Metadata.
- Authelia OIDC integration as the first preferred authorization-server path.
- Explicit handling of the Authelia Dynamic Client Registration gap.
- At least one real MCP service published end to end, preferably Soniq first because it already has OIDC-related configuration hooks.
- Documentation for adding additional MCP services.
- Compatibility notes for Claude, OpenAI/ChatGPT, and Gemini.

### Out of scope for MVP

- A full replacement authorization server unless Authelia proves incompatible.
- Public exposure before auth and TLS are working.
- A generic stdio-to-HTTP MCP proxy layer as the centerpiece.
- Vendor-specific desktop automation beyond connector setup documentation.

## Key Design Decisions

### mcp-causeway is a publication layer

Earlier assumptions centered on `mcp-proxy` and stdio-to-HTTP conversion. Discovery showed that the current day-one MCP services are already HTTP-based. Therefore, the core product is not a proxy for stdio servers; it is a secure publication layer for existing HTTP MCP services.

`mcp-proxy` may still be useful later as an optional on-ramp for future stdio servers.

### Public reachability is part of MVP

An earlier LAN-only / Claude Desktop-only MVP was superseded by the requirement to work with Gemini and ChatGPT where possible. Since these clients often operate from vendor cloud infrastructure and require publicly trusted TLS, public reachability is now part of the MVP.

### Use real TLS, not private CA trust

Self-signed or homelab CA certificates are not sufficient for broad cloud-client compatibility. The MVP should use Let's Encrypt via DNS-01 to avoid opening inbound HTTP solely for certificate issuance.

### Prefer standards over tunnels

Cloudflare Tunnel and similar tools remain possible fallbacks, but the preferred design is an owned-domain, DNS-01, split-horizon, authenticated ingress pattern. This avoids adding Cloudflare as a required dependency or placing it in the request path by default.

### Treat auth as a first-class product requirement

MCP publication without auth is not acceptable for this use case. The product should support OAuth/OIDC, resource-server behavior, JWT validation, and Protected Resource Metadata.

## OpenAI Agents API Relationship

OpenAI's Agents API now supports MCP tools directly. This is closely aligned with mcp-causeway, but it does not fully replace it.

OpenAI supports two important connection origins:

| OpenAI MCP mode | Connection originates from | Impact on mcp-causeway |
|---|---|---|
| `connection_origin: "service"` | OpenAI cloud | Needs a public, trusted, authenticated MCP endpoint. This matches mcp-causeway's publication-layer purpose. |
| `connection_origin: "environment"` | A session executor/environment | Can reach private MCP services if the executor is attached to the right network. This may be a useful OpenAI-only fast path. |

Therefore:

- For OpenAI Agents with an environment that can reach the homelab, mcp-causeway may be reduced or bypassed.
- For OpenAI service-origin connections, ChatGPT-style hosted connectors, Claude, and Gemini, mcp-causeway remains the vendor-portable solution.

The product should document both paths rather than pretending there is only one deployment model.

## Functional Requirements

### FR1: Public MCP endpoint publication

The system must publish one or more selected internal MCP services under a real domain with publicly trusted HTTPS.

### FR2: Split-horizon DNS

The system should allow internal clients to resolve MCP endpoints locally while public/vendor clients resolve them through the public route.

### FR3: TLS automation

The system must automate certificate issuance and renewal using Let's Encrypt and cert-manager DNS-01.

### FR4: OAuth/OIDC authorization

The system must require authenticated access to protected MCP services using OAuth/OIDC-compatible flows.

### FR5: Protected Resource Metadata

The system must support MCP authorization discovery through RFC 9728 Protected Resource Metadata, including appropriate `WWW-Authenticate` responses and metadata URLs.

### FR6: Service routing

The system must route requests to selected internal MCP services without requiring each service to own public TLS/auth concerns individually.

### FR7: Streamable HTTP compatibility

The system must expose or preserve Streamable HTTP MCP compatibility for clients that require it, especially Gemini and modern MCP clients.

### FR8: Vendor compatibility documentation

The system must document known setup constraints for Claude, ChatGPT/OpenAI, and Gemini.

### FR9: Static-client fallback

If Dynamic Client Registration is unavailable, the system must provide a documented static client registration path or a named fallback design.

### FR10: Add-a-service workflow

The product must include a repeatable process for publishing a new internal MCP service through the causeway.

## Non-Functional Requirements

### Security

- Default-deny publication model.
- No public unauthenticated MCP tools.
- Minimal exposed surface area.
- No credentials embedded in command lines, reusable docs, or logs.
- Clear separation between public ingress, authorization, and internal service targets.

### Reliability

- Certificate renewal should be automatic.
- Routing should be observable and testable.
- Failure modes should be documented, especially auth failures and vendor connector initialization failures.

### Portability

- Configuration should be understandable outside Paul's environment.
- Homelab-specific values should be isolated and documented.
- The pattern should be adaptable to other k3s/nginx-ingress deployments.

### Maintainability

- Avoid per-service bespoke patches where a gateway-level solution can handle common behavior.
- Prefer declarative IaC in the existing homelab repo over ad-hoc manual cluster changes.
- Keep the product brief, PRD, architecture notes, and runbooks in sync.

## Key Risks and Open Questions

### Dynamic Client Registration

Authelia does not currently support Dynamic Client Registration. MCP OAuth clients and SDKs may expect DCR. This is the largest known auth risk.

Potential mitigations:

1. Pre-register static OIDC clients per vendor.
2. Build a DCR shim in front of Authelia.
3. Replace or supplement Authelia with an authorization server that supports DCR, such as Zitadel or Keycloak.

### Vendor-specific connector behavior

Claude, ChatGPT/OpenAI, and Gemini may differ in:

- accepted transports;
- OAuth metadata discovery behavior;
- DCR expectations;
- IP allowlisting or network-origin requirements;
- whether private-network or environment-origin execution is supported.

The MVP must validate at least one vendor end to end and document gaps for the others.

### Gateway responsibility boundaries

Some MCP services may implement auth/resource metadata natively; others may not. The product must decide whether the gateway should normalize these behaviors uniformly or preserve per-service differences.

### Public exposure sequencing

The project must avoid exposing MCP endpoints publicly before authentication, authorization metadata, and TLS are verified.

## Suggested MVP Milestones

### Milestone 1: Architecture and threat model

- Confirm public domain and DNS strategy.
- Document request flow.
- Document trust boundaries.
- Decide whether the first gateway is ingress-only, auth-proxy-based, or a purpose-built MCP gateway.

### Milestone 2: TLS and DNS foundation

- Configure DNS-01 issuance.
- Validate publicly trusted certificate issuance.
- Establish split-horizon DNS behavior.

### Milestone 3: Auth foundation

- Configure Authelia OIDC path.
- Validate JWT/resource-server behavior against Soniq or a test MCP service.
- Decide static-client vs DCR-shim fallback.

### Milestone 4: First published MCP service

- Publish one selected service end to end.
- Verify MCP initialization and tool listing from outside the LAN.
- Verify unauthorized access fails correctly.

### Milestone 5: Vendor validation

- Test at least one hosted AI client.
- Capture Claude, OpenAI/ChatGPT, and Gemini compatibility notes.
- Document known limitations and next actions.

### Milestone 6: Add-a-service documentation

- Write repeatable instructions for adding another MCP service.
- Include required config fields, routing, auth, metadata, and validation checks.

## Success Criteria

The MVP is successful when:

- At least one internal MCP service is reachable through a public, trusted HTTPS endpoint.
- Unauthenticated access is rejected.
- Authenticated MCP client access succeeds.
- Protected Resource Metadata is discoverable where required.
- The setup works end to end with at least one target AI client.
- The repo contains enough documentation for Paul or another technical operator to add another service without rediscovering the architecture.
- Known incompatibilities with Claude, OpenAI/ChatGPT, and Gemini are documented honestly rather than hidden.

## Initial Documentation Set

Recommended documents under `project-docs/`:

- `product-brief.md` — this document.
- `architecture.md` — diagrams and request flows.
- `threat-model.md` — exposed surfaces, credentials, auth flows, failure modes.
- `vendor-compatibility.md` — Claude, OpenAI/ChatGPT, Gemini requirements and test results.
- `service-onboarding.md` — how to publish another MCP service.
- `operations-runbook.md` — verification, certificate renewal checks, auth troubleshooting.

## Working Definition

mcp-causeway is the secure causeway between private MCP services and public AI clients: narrow, controlled, authenticated, documented, and portable.
