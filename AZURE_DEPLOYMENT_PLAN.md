# OpenClaw → Azure Container Apps: Full E2E Deployment Plan

> ⚠️ **SECURITY POLICY**: Any change that weakens the security posture described in this
> document — including but not limited to enabling public access on locked-down resources,
> adding plain-text secrets, broadening RBAC roles, disabling diagnostics/alerts, or
> opening additional network paths — **MUST be explicitly approved before implementation**.
> No security regression is acceptable as a shortcut to get things working.

## Goal
Deploy a fully functional, security-hardened OpenClaw AI messaging gateway on Azure
Container Apps — from zero to a working deployment with WhatsApp/Telegram/Discord
integration, persistent storage, and centralized monitoring.

## What is OpenClaw?
OpenClaw is an AI-powered messaging gateway that connects messaging platforms
(WhatsApp, Telegram, Discord, Slack, SMS) with AI providers (Anthropic Claude,
OpenAI GPT). It acts as a personal AI assistant accessible from any chat app.

---

## Architecture (As Deployed)

```
  WhatsApp / Telegram / Discord / Browser
                    │
                    ▼ (HTTPS only, port 443, 48-char token auth)
┌──────────────────────────────────────────────────────────┐
│              Azure Resource Group: openclaw-rg            │
│              Region: westus2                              │
│              Lock: CanNotDelete                           │
│                                                          │
│  ┌─── VNET: openclaw-vnet (10.0.0.0/16) ────────────┐  │
│  │                                                    │  │
│  │  ┌─ container-apps subnet (10.0.0.0/23) ───────┐  │  │
│  │  │  NSG: openclaw-capp-nsg (defaults only)      │  │  │
│  │  │  Delegation: Microsoft.App/environments      │  │  │
│  │  │                                              │  │  │
│  │  │  Container Apps Environment: openclaw-env    │  │  │
│  │  │  • VNET-integrated                           │  │  │
│  │  │  • mTLS enabled                              │  │  │
│  │  │  • Peer traffic encryption enabled           │  │  │
│  │  │  • Log Analytics integrated                  │  │  │
│  │  │                                              │  │  │
│  │  │  ┌──────────────────────────────────────┐    │  │  │
│  │  │  │  openclaw-gateway                    │    │  │  │
│  │  │  │  • 0.25 vCPU / 0.5 GiB              │    │  │  │
│  │  │  │  • Image: digest-pinned (immutable)  │    │  │  │
│  │  │  │  • User-assigned Managed Identity    │    │  │  │
│  │  │  │  • Non-root (node user, UID 1000)    │    │  │  │
│  │  │  │  • Secrets from Key Vault refs only  │    │  │  │
│  │  │  │  • Health: /healthz + /readyz        │    │  │  │
│  │  │  │  • HTTPS only, allowInsecure=false   │    │  │  │
│  │  │  │  • maxInactiveRevisions: 3           │    │  │  │
│  │  │  └──────────────────────────────────────┘    │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │                                                    │  │
│  │  ┌─ private-endpoints subnet (10.0.2.0/24) ────┐  │  │
│  │  │  NSG: openclaw-pe-nsg                        │  │  │
│  │  │  • Allow TCP/443 from 10.0.0.0/23 only      │  │  │
│  │  │  • Allow TCP/445 from 10.0.0.0/23 only      │  │  │
│  │  │  • Deny all other inbound                    │  │  │
│  │  │                                              │  │  │
│  │  │  ● pe-keyvault  → openclaw-kv5898  (10.0.2.4)│  │  │
│  │  │  ● pe-storage   → openclawstcb2fbc06(10.0.2.5)│  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │                                                    │  │
│  │  Private DNS Zones:                                │  │
│  │  • privatelink.vaultcore.azure.net → 10.0.2.4     │  │
│  │  • privatelink.file.core.windows.net → 10.0.2.5   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌─ Monitoring & Alerting ───────────────────────────┐  │
│  │  Log Analytics: openclaw-logs                      │  │
│  │  • Public ingestion: Disabled                      │  │
│  │  • Public query: Disabled                          │  │
│  │  Diagnostics flowing from:                         │  │
│  │  ← Key Vault (AuditEvent)                         │  │
│  │  ← ACR (RepositoryEvents, LoginEvents)            │  │
│  │  ← Storage (file read/write/delete + metrics)     │  │
│  │  ← Container App (AllMetrics)                      │  │
│  │  ← Container App Environment (app console logs)    │  │
│  │                                                    │  │
│  │  Defender for Containers: Standard (CVE scanning)  │  │
│  │                                                    │  │
│  │  Alert Rules → coletaylor788@gmail.com:            │  │
│  │  • Container restarts                              │  │
│  │  • Resource deletions                              │  │
│  │  • NSG rule changes                                │  │
│  │  • RBAC role assignment changes                    │  │
│  └───────────────────────────────────────────────────┘  │
│                                                          │
│  ┌─ Patching Strategy ───────────────────────────────┐  │
│  │  • Defender continuously scans ACR image for CVEs  │  │
│  │  • Alert emails when vulnerabilities detected      │  │
│  │  • Manual rebuild: git pull → az acr build → update│  │
│  │  • Watch openclaw/openclaw releases for app updates│  │
│  └───────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## Security Posture (Verified by 3 Independent Audits)

### Network
| Control | Status |
|---------|--------|
| Only port 443 (HTTPS) exposed externally | ✅ Verified |
| Key Vault public access disabled | ✅ Verified (403 from internet) |
| Storage public access disabled, default action Deny | ✅ Verified (403 from internet) |
| Log Analytics public ingestion + query disabled | ✅ Verified |
| Private endpoints: KV + Storage with DNS resolution verified | ✅ Verified |
| PE subnet NSG: only 443/445 from container subnet, explicit deny-all | ✅ Verified |
| mTLS between containers enabled | ✅ Verified |
| Peer traffic encryption enabled | ✅ Verified |
| ACR public access (Basic SKU — no PE support) | ⚠️ Accepted risk ($45/mo for Premium) |

### Identity & Secrets
| Control | Status |
|---------|--------|
| Only 1 secret exists: gateway-token in Key Vault | ✅ Verified |
| All secrets via Key Vault references (versionless URL) | ✅ Verified |
| Zero hardcoded credentials in container config | ✅ Verified |
| Managed Identity: AcrPull, KV Secrets User, Storage File Data Contributor | ✅ Verified (least-privilege, resource-scoped) |
| ACR admin disabled | ✅ Verified |
| Storage shared key access disabled (AAD-only) | ✅ Verified |
| Key Vault: RBAC mode, soft-delete, purge protection | ✅ Verified |
| No rogue service principals or app registrations | ✅ Verified |
| Azure Files mount uses cached storage key | ⚠️ Platform limitation (not fixable) |

### Container
| Control | Status |
|---------|--------|
| Image pinned by SHA256 digest (immutable) | ✅ Verified |
| Defender scan: 0 CVEs | ✅ Verified |
| Non-root user (node, UID 1000) | ✅ Verified |
| Health probes: /healthz (liveness) + /readyz (readiness) | ✅ Verified |
| Container healthy, 0 restarts | ✅ Verified |
| No writable volume mounts or elevated capabilities | ✅ Verified |
| maxInactiveRevisions: 3 | ✅ Verified |

### Monitoring
| Control | Status |
|---------|--------|
| Diagnostics on KV, ACR, Storage, Container App | ✅ Configured |
| Alert rules: restarts, deletions, NSG changes, RBAC changes | ✅ Configured |
| Action group: email to coletaylor788@gmail.com | ✅ Configured |
| Defender for Containers: Standard tier | ✅ Enabled (30-day trial, ~$2/mo after) |
| Resource lock: CanNotDelete on RG | ✅ Verified |

### Accepted Risks
| Risk | Reason | Mitigation |
|------|--------|------------|
| ACR has public network access | Basic SKU ($5/mo) doesn't support PE. Premium is $50/mo. | MI auth required, admin disabled, no anonymous pull |
| KV soft-delete retention 7 days | Cannot change after creation. Would need vault recreation. | Purge protection enabled. Low probability event. |
| Storage Standard_LRS (no geo-redundancy) | Cost doubles for ZRS. Data is WhatsApp sessions (re-pairable). | Periodic backups recommended |
| No IP restrictions on ingress | WhatsApp/Telegram webhooks use dynamic IPs. | 48-char random token auth on all requests |
| Defender on free trial (29 days) | Will cost ~$2/mo after trial. | Budget allocated. |

---

## Deployed Resources

| Resource | Name | Key Config |
|----------|------|------------|
| Resource Group | `openclaw-rg` | westus2, CanNotDelete lock |
| VNET | `openclaw-vnet` | 10.0.0.0/16, 2 subnets |
| Container Apps Environment | `openclaw-env` | VNET-integrated, mTLS, peer encryption |
| Container App | `openclaw-gateway` | 0.25 vCPU/0.5GiB, digest-pinned, MI auth |
| ACR | `openclawacr7101` | Basic, admin disabled |
| Storage Account | `openclawstcb2fbc06` | Standard_LRS, private endpoint, shared keys disabled |
| Key Vault | `openclaw-kv5898` | RBAC, purge protection, private endpoint |
| Managed Identity | `openclaw-identity` | AcrPull, KV Secrets User, Storage Contributor |
| Log Analytics | `openclaw-logs` | Public access disabled |
| Private Endpoints | `pe-keyvault`, `pe-storage` | Approved, DNS verified |

Gateway URL: `https://openclaw-gateway.agreeablepond-b63b74e9.westus2.azurecontainerapps.io`

---

## Remaining Steps (E2E Configuration)

### Phase 4: Application Configuration
| # | Step | Details | Status |
|---|------|---------|--------|
| 14 | Mount persistent storage | Azure Files volume at /root/.clawdbot/devices for WhatsApp sessions | 🔲 Pending |
| 15 | Configure AI provider | Add Anthropic/OpenAI API key to Key Vault, update container env vars | 🔲 Pending |
| 16 | Device pairing | Open gateway URL → trigger pairing → approve via CLI | 🔲 Pending |
| 17 | Configure local OpenClaw | ~/.openclaw/openclaw.json with remote gateway config | 🔲 Pending |
| 18 | Add messaging channels | WhatsApp, Telegram, Discord — each stores sessions in Azure Files | 🔲 Pending |
| 19 | End-to-end test | Send test messages, verify AI responses, confirm persistence | 🔲 Pending |

---

## Maintenance Runbook

### Rebuild after Defender CVE alert
```bash
# ACR cloud build (no local Docker needed)
az acr build --registry openclawacr7101 --image openclaw:latest \
  --platform linux/amd64 https://github.com/openclaw/openclaw.git

# Get new digest
NEW_DIGEST=$(az acr manifest list-metadata --registry openclawacr7101 --name openclaw \
  --query "[0].digest" --output tsv)

# Update container app
az containerapp update --name openclaw-gateway --resource-group openclaw-rg \
  --image "openclawacr7101.azurecr.io/openclaw@$NEW_DIGEST"
```

### Rotate gateway token
```bash
NEW_TOKEN=$(openssl rand -hex 24)
az keyvault secret set --vault-name openclaw-kv5898 --name gateway-token --value $NEW_TOKEN
# Restart to pick up new secret (Container Apps don't hot-reload KV secrets)
az containerapp revision restart --name openclaw-gateway --resource-group openclaw-rg
```

### Scale up if needed
```bash
az containerapp update --name openclaw-gateway --resource-group openclaw-rg \
  --cpu 0.5 --memory 1.0Gi
```

### Backup device data
```bash
# Note: shared key access is disabled. Use az login identity or SAS token.
az storage file download-batch --destination ./backup --source openclaw \
  --account-name openclawstcb2fbc06 --auth-mode login
```

### View logs
```bash
az containerapp logs show --name openclaw-gateway --resource-group openclaw-rg --follow
```

---

## Cost Estimate (~$62/month)

| Resource | Monthly Cost |
|----------|-------------|
| Container App (0.25 vCPU, 0.5Gi, always-on) | ~$20 |
| ACR Basic | ~$5 |
| Key Vault Standard | ~$3 |
| Storage (1 GB Standard_LRS) | ~$0.06 |
| Log Analytics (< 5 GB free) | $0 |
| Private Endpoints (×2) | ~$14 |
| Defender for Containers | ~$2 |
| Alerts | $0 |
| **Total** | **~$44-62** |

---

## Troubleshooting

| Problem | Check |
|---------|-------|
| Container keeps restarting | `az containerapp logs show --tail 100` — likely missing API key or OOM |
| Can't access web UI | Verify container is running, token is correct, HTTPS URL |
| Pairing request not showing | Requests expire in minutes — retry. Check logs. |
| Sessions not persisting | Verify Azure Files volume mount is attached |
| Image pull fails | Verify MI has AcrPull role, ACR is accessible |
| KV secret not loading | Verify private DNS resolves, MI has KV Secrets User role |
