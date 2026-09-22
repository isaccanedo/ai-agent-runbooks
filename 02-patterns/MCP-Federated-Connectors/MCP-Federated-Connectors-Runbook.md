# Runbook - MCP Federated Connectors

> Use this for **Microsoft 365 Copilot federated connectors**, not for registering a direct
> MCP tool in Copilot Studio or another agent. Start with the
> [pattern and qualification gates](MCP-Federated-Connectors.md).

**Baseline:** September 22, 2026. Follow the current
[custom setup guide](https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/set-up-custom-federated-connectors)
and provider-specific instructions; labels and availability can vary.

---

## Prerequisites

| Requirement | Owner |
|---|---|
| Read-only use case, target Copilot experience, and business acceptance prompts | Delivery lead |
| Gallery connector or compatible MCP server with curated read-only tools | Source/platform owner |
| Global Administrator or AI Administrator for the documented custom creation path | Microsoft 365 admin |
| Appropriate Entra role and Teams Developer Portal access for authentication setup | Identity admin |
| Approved network endpoint, source permission model, and data-processing boundary | Network/security/source owners |
| Source accounts and any required host/source entitlements | Source and Microsoft 365 admins |
| Pilot Entra group and two users with different source permissions | Delivery lead |
| Developer name, documentation, privacy-policy and terms URLs for custom onboarding | Service owner |

Do not request blanket admin privileges for end users or give a service identity broad source
access to avoid user authentication.

---

## Step 1 - Qualify the Retrieval and Host

Record:

| Decision | Evidence to capture |
|---|---|
| Target experience | Copilot Chat, Copilot in Excel, Researcher, or Cowork; exact tenant availability |
| Required data | Fields, content containers, freshness, source links, and result-size bounds |
| Read vs write | Which needs are retrieval; which remain separate actions |
| Gallery vs custom | Provider availability or owner of the organisation-built MCP server |
| User access | Broad and restricted-user expected results, including known denied records |
| Retention/residency | Approval for returned content to be processed by Copilot, despite no source index |

**Stop** if the plan relies on automatic support in a different agent/channel. Qualify that
surface independently or retain the existing integration.

**Deliverable:** scope and host decision with a small set of answerable pilot questions.

---

## Step 2 - Prove Reachability Before Onboarding

1. Identify the **connector-facing** MCP URL separately from any private backend URL.
2. Confirm trusted HTTPS, DNS, the correct MCP path, and transport behaviour.
3. For a private-only on-premises server, record the blocker: the documented federation setup
   has no private-network attachment or arbitrary MCP relay.
4. If approved, design a public **authenticated** gateway with private outbound connectivity
   to the MCP server. For Azure this may use a VNet and S2S VPN; an AWS equivalent uses a
   suitable VPC-connected runtime and private backhaul.
5. Validate gateway-to-server DNS, forward/return routes, firewalls, backend certificates,
   streaming, and identity-service egress from the gateway runtime.
6. Reserve the actual federated-host test for Step 5; laptop reachability is insufficient.

If public ingress is prohibited, stop this route. Consider supported synced ingestion if an
index is acceptable, or an explicitly supported private-network agent architecture. Neither
the Graph connector agent nor a user's VPN is a documented federated MCP relay.

**Deliverable:** approved network path or an explicit no-go, not a private URL in a wizard.

---

## Step 3 - Configure and Qualify Authentication

### Entra SSO

Follow the custom guide's linked steps to update the Entra app registration, configure the
required token audience at the API, and register an SSO client in Teams Developer Portal.
Use the resulting **SSO registration ID**, not an app/client ID, in connector creation.
Preserve issuer, audience, client, scope, lifetime, and user authorization checks.

### OAuth 2.0

1. Register the app with the source OAuth provider, using the documented callback:
   `https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`.
2. In Teams Developer Portal, use **Tools > OAuth Client Registration > New OAuth connection**.
3. Configure client credentials, authorization/token/refresh endpoints, and least-privilege
   scopes. Enable PKCE when supported by the provider.
4. Save and capture the **OAuth client registration ID** for connector creation.
5. Store and rotate credentials through approved controls; do not paste secrets into runbooks.

This is not the Copilot Studio callback/connection procedure. Do not reuse a different host's
registration settings without checking the federation requirements.

### No Auth

Use only for intentionally public, non-sensitive sources approved for anonymous retrieval.
Never use it to route around enterprise OAuth failures.

### Complex-auth spike

For each hop, record issuer, expected audience, allowed client, delegated user/scopes, token
validator, and any exchange component. Use a gateway adapter only if it is approved and
preserves the user-level authorization contract.

| Test | Expected result |
|---|---|
| Normal sign-in and consent | Real user connects and receives only permitted content |
| Expiry and refresh | Correct renewal or explicit reauthentication, not a loop or service-account fallback |
| Revocation or consent denial | Access stops or fails explicitly as designed |
| Conditional Access/MFA challenge | Supported challenge completes or a clear blocker is recorded |
| Wrong issuer/audience/client/scope or expired token | Protected MCP route rejects the call |
| Gateway token exchange, if used | Downstream token has the correct audience and user context |
| Two concurrent users | No token, cache, response, or permission leakage between users |

**Deliverable:** authentication design and test evidence. A desktop MCP client or Copilot
Studio success is not federation acceptance.

---

## Step 4 - Create and Stage the Connector

For a **gallery connector**, use its provider-specific onboarding and required approval.

For an **organisation-created connector**:

1. In Microsoft 365 admin center, go to **Copilot > Connectors > Gallery**.
2. Under **Created by your org**, select **Create a new connector > Add**.
3. Under **Connect to MCP server**, select **Add**.
4. Enter the display name and exact connector-facing MCP base URL.
5. Enter the matching SSO or OAuth registration ID when authentication is required.
6. Save, then manage the connection under **Your connections**.
7. Restrict rollout to the pilot Entra group before making the source broadly available.
8. Check tenant **Allowed agent types** and per-connector scope together. Do not use the
   retired federation CLI toggle for a new deployment.

The custom setup guide notes that connector changes can take up to 15 minutes. Recheck the
actual user experience after propagation rather than repeatedly recreating the connection.

**Deliverable:** connector record, audience, authentication registration reference, and owner.

---

## Step 5 - Connect as a User and Validate in the Host

In supported Copilot Chat, users can use **Settings > Sources > Connect** and complete
authentication. Researcher exposes source selection; verify other target experiences
individually. Do not interpret admin enablement as completed user connection.

| Test | Pass criteria |
|---|---|
| Signed-in discovery and invocation | Actual host selects and successfully calls the expected read-only tools |
| Known-record accuracy | Returned facts match source truth and identify the correct record/version |
| Freshness | A controlled source update is reflected within the agreed target; record timestamps |
| Restricted user | Denied content is absent from results, snippets, citations, and subsequent fetches |
| Permission change | Source revocation is honoured; document token/cache/session effects |
| Read-only enforcement | No exposed tool or indirect call mutates source state |
| Citation usability | Evidence links open the correct source item for an authorised user |
| Completeness | Bounded results/pagination are disclosed; no invented "all records" claim |
| Source failure, throttle, or timeout | Explicit failure, not a fabricated answer or false empty result |
| Private backhaul failure, if used | Fail closed; no fallback to public backend or anonymous access |
| Concurrent identities | Separate users retain separate permissions and results |
| Disconnect and admin disable | Access behaves as configured after propagation |

Set measurable thresholds before testing: accepted latency, freshness, answer accuracy,
permission-denial success, result limits, and outage behaviour. Record observed results, not
just checkmarks. Never equate a health endpoint with user retrieval.

**Deliverable:** host-level acceptance matrix with evidence and unresolved blockers.

---

## Step 6 - Release, Monitor, and Roll Back

- Obtain source-owner, identity, network, and compliance acceptance.
- Retest when provider tools, MCP protocol/runtime, authentication, or gateway policies change.
- Track tool failures, latency, throttles, sign-in failures, and source availability using
  available connector, gateway, and source telemetry. Verify Purview audit coverage.
- Keep credentials and returned sensitive content out of logs.
- Document connector disablement as the normal rollback; deletion removes configuration.
  Record expected propagation and verify rollback from a pilot user's session.
- Keep synced and federated sources distinguishable if they coexist; avoid conflicting or
  stale duplicates being presented as current.
- Assign ownership for credentials/certificates, source permissions, network/VPN operation,
  runtime upgrades, and support escalation.

**Release gate:** no unresolved authorization or reachability gap; signed-in host tests pass.

## Handover

| Artifact | Required content |
|---|---|
| Connector record | Provider/custom owner, host, tenant audience, endpoint, non-secret registration references |
| Auth/network evidence | Supported flow and routing, negative tests, complex-flow decisions, private-backhaul limits |
| Quality results | Representative questions, source truth, freshness/latency measurements, citations and bounds |
| Operations | Telemetry, audit, rotation, upgrade tests, incident owner, disable/rollback procedure |

See the [pattern references](MCP-Federated-Connectors.md#reference-documentation) for the source
documentation and the separate MCP Server Integration pattern for server/gateway engineering.
