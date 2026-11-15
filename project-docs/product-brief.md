---
title: mcp-causeway Product Brief
status: draft
created: 2026-09-12
updated: 2026-09-12
revision: 2
---

# Product Brief: mcp-causeway

> **Revision note.** Revision 2 removes public exposure from the product entirely. The 2026-09-12 draft was built on publishing MCP services to cloud-hosted AI clients over a real domain with publicly trusted TLS. That is now ruled out: the causeway runs inside the home network and is addressed at `192.168.2.201` and `localhost`. Reach is no longer the value proposition — **normalization is**. The superseded revision is kept at `project-docs/planning-artifacts/prds/prd-mcp-causeway-2026-09-06/superseded-product-brief-2026-09-12.md`.

## Summary

mcp-causeway is a LAN-internal normalization layer that makes the homelab's existing HTTP MCP servers present themselves identically to any MCP client on the network.

Today Soniq, MQTT MCP and Hindsight each sit on their own port with their own authorization story and their own degree of MCP spec conformance. A client that wants to use all three has to know three sets of details, and any client that expects standards-compliant authorization discovery gets inconsistent answers. mcp-causeway puts one uniform, conformant, documented surface in front of them.

It is not an MCP server implementation, and it is not a publication or tunnelling layer. It is a routing, conformance and authorization-normalization pattern for MCP servers that already run inside the network — plus the documentation that lets another homelab operator reproduce it.

## Problem

Paul runs several MCP-capable services inside a private k3s cluster. They work, but they are not uniform:

- **Addressing is per-service.** Services sit on `:8000` and `:8888` behind nginx-ingress. There is no single MCP surface; every client is configured per service.
- **Authorization is inconsistent.** Soniq can behave as an OIDC resource server (it accepts issuer, audience, JWKS and resource-URL configuration). MQTT MCP and Hindsight are bearer-token-only. There is no common model.
- **Authorization discovery is largely absent.** The MCP spec expects servers to implement RFC 9728 Protected Resource Metadata — a `401` carrying `WWW-Authenticate` with a `resource_metadata` URL, and a metadata document naming the authorization servers. Soniq is plausibly capable of this; the other two are not.
- **Transport conformance is unverified.** Streamable HTTP is the transport modern MCP clients expect. Whether each service conforms, and how it behaves on session handling and error paths, has not been established.
- **Onboarding is undocumented.** Adding a fourth MCP service means rediscovering the same routing, auth and metadata decisions from scratch.

The cost is per-service bespoke work that compounds with every service added, and clients that must be told about implementation differences they should never have to know about.

### What the problem is *not*

Earlier revisions framed the problem as vendor reach — getting cloud-hosted assistants to call into the homelab. That framing is retired. The services are not unreachable by the clients Paul actually uses; they are *inconsistent*. See **Cloud Client Reality** below for why the vendor framing was dropped rather than solved.

## Vision

mcp-causeway turns a set of individually-configured internal MCP services into one coherent MCP surface for the home network.

A client on the LAN points at the causeway, discovers what is available through standards-compliant means, authorizes once against one model, and speaks conformant Streamable HTTP to any service behind it — without knowing which port that service happens to listen on or whether it natively speaks OAuth.

The product should be useful in two ways:

1. **As working infrastructure for Paul's homelab** — good enough that real assistants use real internal tools day to day, without per-service client configuration.
2. **As documentation-as-product** — clear enough that another technical homelab operator can understand the trade-offs, reproduce the pattern, and adapt it to their own small cluster. Homelab-specific values stay isolated and named.

## Target Users

### Primary user

Paul, operating a private homelab k3s cluster with several internal MCP services, using LAN-local AI clients — Claude Desktop, Claude Code, and locally-run clients.

### Secondary users

Technical homelab operators who:

- run several MCP services on a private LAN or small k3s cluster;
- want one consistent, spec-conformant MCP surface rather than per-service configuration;
- care about a defensible internal authorization story and a minimal footprint;
- prefer a documented, portable pattern over one-off fixes.

### Explicit non-users

Anyone needing MCP services reachable from outside their own network. That is a different product and this brief does not serve it.

## Network and Addressing Model

This section is load-bearing; it constrains nearly every requirement below.

- The causeway is reachable **only from inside the home network**. There is no public endpoint, no domain name, no tunnel, and no inbound port-forward.
- Addressing is **`192.168.2.201`** (the k3s ingress host, `rpi4-1`) and **`localhost`**.
- **Avahi/mDNS** exists in the environment and resolves some names. It is available but not required — nothing in the MVP may *depend* on mDNS resolution.
- Existing internal domains (`home.lab`, `k3s.home.arpa`) and the internal HTTPS NodePort `30443` remain part of the environment and may be used, but are not required by the design.
- TLS, where used, uses the **existing homelab CA** (`homelab-ca-issuer` → `homelab-tls`). This is acceptable precisely because Paul controls every client's trust store. No publicly trusted certificate authority is involved.

## Current System Context

Known environment facts from discovery:

- The k3s ingress host is `192.168.2.201` / `rpi4-1`.
- Cluster ingress is **nginx-ingress**; Traefik is disabled. (An earlier draft incorrectly assumed Traefik.)
- Internal domains `home.lab` and `k3s.home.arpa` exist; internal HTTPS is on NodePort `30443`.
- TLS uses a homelab CA / self-signed trust model, with clients trusting the CA.
- Existing MCP services are **already HTTP-based**. There are no day-one stdio MCP servers, so `mcp-proxy` is not required by anything shipping in the MVP.
- Soniq already accepts OIDC resource-server configuration (`oidc_issuer`, `audience`, `jwks_uri`, `resource_url`).
- MQTT MCP and Hindsight are bearer-token-only and would need gateway-level handling to present a uniform authorization model.
- Authelia is already deployed with OIDC configured (HMAC secret, JWKS key, a client). It does **not** support Dynamic Client Registration — which no longer matters much; see **Open Questions**.
- Infrastructure lives in the `iot-edge-k3s` repo as Terragrunt/Terraform IaC.

## Product Goals

1. **Present one uniform MCP surface**
   - A single addressable surface routing to selected internal MCP services.
   - Clients configured once, not once per service.

2. **Normalize authorization discovery**
   - Standards-compliant RFC 9728 Protected Resource Metadata for every published service, including services that cannot emit it themselves.
   - Correct `401` / `WWW-Authenticate` behaviour on the unauthorized path.

3. **Guarantee transport conformance**
   - Streamable HTTP as the surface every published service presents.
   - Verified session and error behaviour, not assumed.

4. **Provide a consistent authorization model**
   - One model across services whose native capabilities differ.
   - Scope and MVP status of this goal are an open decision — see **Open Questions Q2**.

5. **Keep the footprint minimal**
   - No public exposure, no domain, no third-party tunnel, no new dependency where an existing cluster component will do.
   - Prefer the smallest mechanism that satisfies the requirement.

6. **Keep the design portable**
   - Homelab-specific values isolated and documented.
   - Adaptable to other small k3s / nginx-ingress environments.

7. **Document the pattern thoroughly**
   - Architecture, request flow, internal threat model, client setup, onboarding a new service, trade-offs, and honest limitations.

## Non-Goals

mcp-causeway will not:

- expose any MCP service outside the home network;
- acquire, configure or depend on a public domain name;
- obtain publicly trusted TLS certificates, or use Let's Encrypt / ACME / DNS-01;
- require Cloudflare Tunnel or any third-party tunnel;
- support cloud-hosted AI clients that must call in from vendor infrastructure;
- depend on Avahi/mDNS resolution for correctness;
- rewrite the existing MCP servers;
- replace the homelab ingress stack;
- publish internal tools automatically or by default;
- prioritize stdio MCP onboarding, since no day-one service needs it;
- become a general-purpose or commercial MCP gateway.

## Cloud Client Reality

Recorded so the constraint is not rediscovered, and because the secondary audience will ask.

With no public endpoint, cloud-origin clients cannot reach the causeway. This is structural, not a gap to close:

| Client | Reachable at `192.168.2.201` / `localhost` | Why |
|---|---|---|
| Claude Desktop | Yes | Runs on a LAN machine |
| Claude Code | Yes | Runs locally |
| Locally-run / self-hosted clients | Yes | On-network |
| claude.ai web connectors | No | Call originates from vendor cloud |
| Gemini custom MCP | No | Vendor cloud origin; also requires a publicly trusted CA, and is a Business/Enterprise admin feature |
| ChatGPT hosted connectors | No | Cannot reach private networks without OpenAI's tunnel |
| OpenAI Agents API, `connection_origin: "service"` | No | Vendor cloud origin |
| OpenAI Agents API, `connection_origin: "environment"` | Conditionally | Works only if the session executor itself sits on the LAN |

Deliverable: a short honest note explaining the above. Not a compatibility programme.

## MVP Scope

The MVP delivers one uniform, conformant, documented MCP surface in front of a small number of existing internal HTTP MCP services.

### In scope

- A single LAN-internal MCP surface on `192.168.2.201`, routing to selected internal MCP services.
- nginx-ingress configuration consistent with the existing cluster.
- Protected Resource Metadata emitted or forwarded for every published service, including those that cannot produce it themselves.
- Correct unauthorized-path behaviour (`401` + `WWW-Authenticate` + discoverable metadata).
- Streamable HTTP conformance for every published service.
- TLS via the existing homelab CA where a client requires HTTPS — scope depends on **Q3**.
- At least one real MCP service published end to end. **Soniq first**, because its existing OIDC configuration hooks make it the cheapest proof.
- At least one real client working end to end against it — Claude Desktop or Claude Code.
- Documentation: architecture, request flow, internal threat model, add-a-service workflow, client setup, cloud-client limitations.

### Out of scope for MVP

- Anything public: endpoints, domains, certificates, tunnels.
- Replacing Authelia or introducing a different authorization server.
- A generic stdio-to-HTTP MCP proxy as a centerpiece.
- Publishing all internal services — selection is deliberate and default-deny.
- Vendor desktop automation beyond written client setup steps.

## Key Design Decisions

### mcp-causeway is a normalization layer

Discovery killed two successive framings. The first centred on `mcp-proxy` converting stdio servers to HTTP — abandoned when a search of the infrastructure repo found **zero stdio MCP servers**; all day-one services were already HTTP. The second centred on secure *publication* to cloud AI clients — abandoned when public exposure was ruled out entirely.

What remains, and what the product actually is: the services are reachable but **inconsistent**, and normalizing them is the work. `mcp-proxy` may return later as an optional on-ramp if a stdio server ever appears.

### The network boundary is the primary security control

With no inbound path from the internet, the trust boundary is the home network itself. The threat model is LAN peers, IoT devices with questionable firmware, and guest-network devices — real, but categorically weaker than internet exposure. Controls should be proportionate to that model rather than inherited from the public-exposure design.

### Homelab CA is sufficient

Publicly trusted certificates were only ever required by cloud vendors, all now out of scope. Paul controls every client, so the existing `homelab-ca-issuer` is the correct answer, and CA trust distribution to client machines is ordinary setup work rather than a design problem.

### Dynamic Client Registration is no longer a blocking risk

Authelia's lack of DCR (RFC 7591) was the largest named risk in the superseded revision, with three mitigation candidates including swapping authorization servers. DCR exists so that *unknown* clients can self-register. With a handful of clients that Paul configures by hand, static registration is simply the normal path. The risk is downgraded, not mitigated — it stopped applying.

### Capability before mechanism

This brief states what the surface must do. Whether that is satisfied by nginx-ingress configuration alone, an auth proxy, or a small purpose-built component with code in it is an **architecture** decision, deliberately not made here. It is the first thing architecture must settle — see **Q1**.

## Functional Requirements

Renumbered for this revision. Superseded-revision IDs noted for traceability.

### FR1: Uniform MCP surface
The system must present a single LAN-internal surface that routes to selected internal MCP services, so that clients need no per-service addressing. *(was FR6)*

### FR2: Protected Resource Metadata normalization
The system must make RFC 9728 Protected Resource Metadata discoverable for every published service — including services that cannot emit it natively — with correct `401` and `WWW-Authenticate` behaviour on the unauthorized path. *(was FR5)*

### FR3: Streamable HTTP conformance
The system must present conformant Streamable HTTP for every published service, with session and error behaviour verified rather than assumed. *(was FR7)*

### FR4: Consistent authorization model
The system must present one authorization model across services whose native capabilities differ, so that a client authorizes the same way regardless of which service it is reaching. **MVP status open — see Q2.** *(was FR4; absorbs FR9, since static client registration is now simply the normal path)*

### FR5: Internal TLS where required
Where a client requires HTTPS, the system must serve it using the existing homelab CA, with documented client-side CA trust steps. **Scope conditional on Q3.** *(replaces FR3, radically reduced)*

### FR6: Add-a-service workflow
The product must provide a repeatable, documented process for publishing an additional internal MCP service — required configuration, routing, authorization, metadata, and validation checks. *(was FR10)*

### FR7: Client setup documentation
The product must document how to configure each supported LAN client against the causeway. *(was FR8, narrowed to supported clients)*

### FR8: Cloud-client limitation note
The product must state plainly which cloud-hosted clients cannot work and why, so neither Paul nor a secondary reader rediscovers it. *(new; the honest residue of old FR8)*

**Dropped:** old FR1 (public endpoint publication), old FR2 (split-horizon DNS), old FR3 (ACME/DNS-01 automation).

## Non-Functional Requirements

### Security
- Default-deny publication: services are exposed by explicit selection only.
- No unauthenticated access from outside the network — satisfied structurally, since there is no outside path.
- Controls proportionate to the LAN threat model: LAN peers, IoT devices, guest network.
- No credentials in command lines, reusable documentation, or logs.
- Clear separation between the ingress surface, the authorization decision, and internal service targets.

### Reliability
- Routing observable and testable.
- Internal certificate issuance and renewal automatic via the existing cluster issuer.
- Failure modes documented — particularly authorization failures and client initialization failures, which are the hardest to diagnose from a client UI.

### Portability
- Configuration comprehensible outside Paul's environment.
- Homelab-specific values (`192.168.2.201`, internal domains, CA names) isolated and clearly marked.
- Pattern adaptable to other k3s / nginx-ingress homelabs.

### Maintainability
- Gateway-level solutions over per-service bespoke patches.
- Declarative IaC in the existing `iot-edge-k3s` repo over manual cluster changes.
- Brief, PRD, architecture notes and runbooks kept in sync.

### Footprint
- No new infrastructure dependency where an existing cluster component suffices.
- Adding the causeway must not require changes to the existing MCP services themselves.

## Key Risks and Open Questions

### Q1 — Gateway responsibility boundary *(blocks architecture)*
Should the causeway normalize authorization and metadata **uniformly** for all services, or **preserve per-service differences** and only route? This decides whether the product is configuration or code. FR2 in particular is hard to satisfy with ingress rules alone, since emitting Protected Resource Metadata for a service that lacks it means serving a JSON document and crafting headers on that service's behalf.

### Q2 — Does authorization stay in the MVP? *(blocks scope)*
With no public exposure, is OAuth/OIDC a day-one requirement, or a documented upgrade path behind a simpler LAN-internal MVP? Keeping it makes FR4 central and pulls Authelia in. Deferring it shrinks the MVP substantially and leaves network-level controls as the answer. **Unanswered — Paul's decision.**

### Q3 — Does Claude Desktop accept a plain `http://` LAN URL? *(blocks FR5 sizing)*
If a plain HTTP LAN endpoint is accepted, FR5 is near-zero work. If HTTPS with a trusted certificate is required, homelab CA trust distribution to every client machine becomes MVP work. **Unverified, and vendor behaviour here changes — must be tested, not assumed.**

### Q4 — What actually annoys Paul today? *(blocks requirement validation)*
Every requirement here is derived from observed inconsistency rather than from stated friction. The concrete irritation when pointing a client at these three services today has not been captured, and it is the strongest available evidence for which requirements matter most. **Unanswered.**

### Q5 — Is this PRD-sized?
If the answer to Q1 is "ingress configuration and three config edits," the work may not warrant a full PRD → architecture → epics chain, and `bmad-build` would be the proportionate path. If FR2 and FR4 need a real component, a PRD earns its keep. Depends on Q1 and Q2.

### Q6 — Per-service conformance is unmeasured
Streamable HTTP conformance, session handling and error behaviour for Soniq, MQTT MCP and Hindsight are assumed, not tested. FR3 may be larger than it looks.

## Suggested MVP Milestones

Revised; the superseded TLS/DNS foundation milestone is gone.

### Milestone 1: Architecture and internal threat model
- Settle **Q1** — ingress-only, auth proxy, or purpose-built component.
- Settle **Q2** — authorization in or out of the MVP.
- Document request flow and trust boundaries for the LAN-only model.

### Milestone 2: Conformance baseline
- Test each service's actual Streamable HTTP behaviour, session handling and error paths.
- Answer **Q3** by testing a real client against a plain HTTP endpoint.
- Record what normalization each service actually needs — closing **Q6**.

### Milestone 3: First published service
- Publish Soniq through the causeway end to end.
- Verify MCP initialization and tool listing from another machine on the LAN.
- Verify Protected Resource Metadata discovery.
- Verify the unauthorized path fails correctly (scope per Q2).

### Milestone 4: Second and third services
- Publish MQTT MCP and Hindsight — the services that need gateway-level normalization, and therefore the real test of the pattern.

### Milestone 5: Client validation
- One real client working end to end: Claude Desktop or Claude Code.
- Capture setup steps as documentation.

### Milestone 6: Onboarding documentation
- Repeatable instructions for adding another service: config fields, routing, authorization, metadata, validation.
- Cloud-client limitation note.

## Success Criteria

The MVP is successful when:

- Selected internal MCP services are reachable through **one** LAN surface, with no per-service client configuration.
- A real client on the LAN completes MCP initialization and tool listing through the causeway.
- Protected Resource Metadata is discoverable for every published service, including those that cannot emit it themselves.
- Streamable HTTP conformance is verified by test, not assumed.
- Where authorization is in scope, the unauthorized path fails correctly and discoverably.
- Adding a further MCP service follows written instructions without rediscovering the architecture.
- Limitations — cloud clients, untested vendors, deferred authorization — are documented honestly rather than omitted.
- Nothing is reachable from outside the home network.

## Initial Documentation Set

Under `project-docs/`:

- `product-brief.md` — this document.
- `architecture.md` — request flow, trust boundaries, the Q1 decision and its rationale.
- `threat-model.md` — LAN threat model, exposed internal surfaces, credentials, failure modes.
- `client-setup.md` — configuring each supported LAN client; includes the cloud-client limitation note.
- `service-onboarding.md` — publishing a further internal MCP service.
- `operations-runbook.md` — verification, internal certificate checks, authorization troubleshooting.

*(Dropped: `vendor-compatibility.md` — a compatibility programme is no longer warranted; its honest residue folds into `client-setup.md`.)*

## Working Definition

mcp-causeway is the causeway between a home network's own AI clients and its own MCP services: one surface, one authorization model, conformant transport, documented and portable — and reachable from nowhere else.
