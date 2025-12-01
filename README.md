# Secure LiteLLM Access via Tailscale + GitHub Actions

A hardened proof-of-concept for accessing an on-premises LiteLLM instance securely from GitHub Actions runners using Tailscale with network hardening.

## Overview

This workflow demonstrates how to:
- Connect a GitHub Actions runner to your private Tailscale network using OAuth
- Access an on-premises LiteLLM instance without exposing it to the public internet
- Harden the CI/CD pipeline with network egress monitoring (StepSecurity)
- Keep all infrastructure details (host, port) secret from public logs

**Architecture:**
```
GitHub Actions Runner → Tailscale (ephemeral tag:ci node) → Private LiteLLM Instance
```

## Prerequisites

### 1. Tailscale Network
- [Tailscale account](https://tailscale.com) with a private network (tailnet)
- Tailscale CLI installed on the machine running LiteLLM
- HTTPS enabled in your tailnet ([Tailscale HTTPS setup](https://tailscale.com/kb/1153/enabling-https)) [web:145]

### 2. LiteLLM Server
- LiteLLM running on your local machine or on-premises
- Accessible via `localhost:<port>` (e.g., `localhost:4000`)

### 3. Tailscale OAuth Client
Create an OAuth client in [Tailscale admin console](https://login.tailscale.com/admin) to allow automated joiners:
- Navigate to **Settings → OAuth Clients**
- Create a new OAuth client
- Save the **Client ID** and **Client Secret** for GitHub Actions secrets

### 4. GitHub Secrets
Add the following secrets to your repository (Settings → Secrets and variables → Actions):
- `TS_OAUTH_CLIENT_ID` – Tailscale OAuth client ID
- `TS_OAUTH_SECRET` – Tailscale OAuth client secret
- `LITELLM_HOST` – Tailscale hostname (e.g., `litellm.your-tailnet-name.ts.net`)
- `LITELLM_PORT` – HTTPS port (e.g., `8443`)

> **Security tip:** Use repository-level secrets rather than organization-level to limit blast radius. [web:147]

## Setup Steps

### Step 1: Enable Tailscale Serve on LiteLLM Host

Expose your LiteLLM service through Tailscale on a dedicated HTTPS port (avoid port 443 if other services use it): [web:107][web:144]

```
# Replace 4000 with your actual LiteLLM port
tailscale serve --https=8443 localhost:4000
```

Verify it's running:
```
tailscale serve get-config --all
```

### Step 2: Configure Tailscale ACLs

Update your Tailscale ACL to restrict `tag:ci` access to only the LiteLLM service (principle of least privilege):

In [Tailscale admin console](https://login.tailscale.com/admin), go to **Settings → ACLs** and add:

```
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:ci"],
      "dst": ["litellm:8443"]
    }
  ]
}
```

This ensures GitHub Actions can only reach LiteLLM on port 8443, nothing else on that host. [web:89][web:103]

### Step 3: Add GitHub Secrets

In your repository:
1. Go to **Settings → Secrets and variables → Actions**
2. Create the four secrets listed above

GitHub automatically masks secret values in logs. [web:147][web:153]

### Step 4: Add the Workflow

Copy `.github/workflows/test-litellm.yml` (see below) to your repository.

## Workflow Details

The workflow includes three layers of security and clarity:

### Layer 1: StepSecurity Harden-Runner
- **Mode:** `egress-policy: audit` (monitors all outbound traffic, does not block)
- **Purpose:** Detects unexpected network connections; provides visibility into what your workflow accesses [web:130][web:128]
- Logs available in StepSecurity dashboard for future tuning to `egress-policy: block`

### Layer 2: Tailscale Ephemeral Node
- Runs the [Tailscale GitHub Action](https://github.com/tailscale/github-action) with OAuth credentials [web:89]
- Creates a temporary node tagged `tag:ci` that lives for the duration of the workflow
- Node is automatically cleaned up after the job completes
- ACLs enforce that `tag:ci` can only reach LiteLLM

### Layer 3: Secret Management
- HTTPS host and port stored as GitHub secrets (masked in logs)
- OAuth credentials only used during workflow execution, never logged
- No infrastructure details in public GitHub logs

## Workflow File

```
name: Test LiteLLM via Tailscale (hardened)

on:
  workflow_dispatch:  # Manual trigger for testing

jobs:
  test-litellm-access:
    runs-on: ubuntu-latest

    steps:
    - name: Harden the runner (egress audit only)
      uses: step-security/harden-runner@c6295a65d1254861815972266d5933fd6e532bdf # v2.11.1
      with:
        egress-policy: audit

    - name: Connect to Tailscale
      uses: tailscale/github-action@a392da0a182bba0e9613b6243ebd69529b1878aa # v4.1.0
      with:
        oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
        oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
        tags: tag:ci

    - name: Show Tailscale status
      run: |
        echo "🔗 Tailscale status:"
        tailscale status

    - name: Test LiteLLM health endpoint
      env:
        LITELLM_HOST: ${{ secrets.LITELLM_HOST }}
        LITELLM_PORT: ${{ secrets.LITELLM_PORT }}
      run: |
        set -e
        echo "🧪 Testing LiteLLM health ..."
        RESPONSE="$(curl -ksS -w '\n%{http_code}' https://$LITELLM_HOST:$LITELLM_PORT/health || true)"
        HTTP_CODE="$(echo "$RESPONSE" | tail -n1)"
        BODY="$(echo "$RESPONSE" | head -n-1)"

        echo "HTTP status: $HTTP_CODE"
        echo "Body: $BODY"

        if [ "$HTTP_CODE" != "200" ]; then
          echo "❌ Health check failed"
          exit 1
        fi
        echo "✅ Health check OK"

    - name: Test simple LiteLLM chat completion
      env:
        LITELLM_HOST: ${{ secrets.LITELLM_HOST }}
        LITELLM_PORT: ${{ secrets.LITELLM_PORT }}
      run: |
        set -e
        echo "🚀 Testing LiteLLM /v1/chat/completions ..."
        curl -ksS -X POST "https://$LITELLM_HOST:$LITELLM_PORT/v1/chat/completions" \
          -H "Content-Type: application/json" \
          -d '{
            "model": "gpt-4",
            "messages": [
              {"role": "user", "content": "Say hello in one short sentence."}
            ],
            "max_tokens": 16
          }' | tee response.json

        if grep -q '"choices"' response.json; then
          echo "✅ LiteLLM responded successfully"
        else
          echo "❌ LiteLLM response did not contain choices"
          cat response.json
          exit 1
        fi

    - name: Summary
      if: always()
      run: |
        echo "## 🔐 Hardened LiteLLM Connectivity Test" >> $GITHUB_STEP_SUMMARY
        echo "- StepSecurity Harden-Runner: enabled (egress audit)" >> $GITHUB_STEP_SUMMARY
        echo "- Tailscale connection: see logs above" >> $GITHUB_STEP_SUMMARY
        echo "- LiteLLM health + simple chat: see logs above" >> $GITHUB_STEP_SUMMARY
```

## Running the Workflow

1. Go to your repository **Actions** tab
2. Select **"Test LiteLLM via Tailscale (hardened)"**
3. Click **"Run workflow"** → **"Run workflow"**
4. Monitor the logs in real time

**Expected output:**
- ✅ Harden-Runner initialized with audit egress policy
- ✅ Tailscale connected (node shown in `tailscale status`)
- ✅ LiteLLM health endpoint responds (HTTP 200)
- ✅ LiteLLM API responds with chat completion (contains `"choices"`)

## Troubleshooting

| Issue | Solution |
| --- | --- |
| **"Connection refused"** | Verify `tailscale serve --https=8443 localhost:4000` is running on your LiteLLM host; check port is correct in secrets |
| **"Peer not found in tailnet"** | Verify the Tailscale hostname in `LITELLM_HOST` secret matches your actual node name; run `tailscale status` on host to confirm |
| **"403 Forbidden" from Tailscale** | Check ACL: `tag:ci` must be allowed to reach `litellm:8443`; verify tag matches workflow config |
| **"404 Not Found" on health endpoint** | Confirm your LiteLLM `/health` endpoint path; adjust curl URL if needed |
| **Secrets appear in logs** | GitHub automatically masks secrets; if you see plaintext, check that you're using `${{ secrets.VARIABLE_NAME }}` syntax, not `${{ env.VARIABLE_NAME }}` [web:153] |

## Next Steps

### Option 1: Integrate into Production Workflow
Once this PoC passes, add the Tailscale and LiteLLM health check to your main digest workflow. This isolates connectivity issues from application logic.

### Option 2: Rotate to Block Mode
After several runs, review StepSecurity's **Network Events** tab to build an allowed egress list. Switch from `egress-policy: audit` to `egress-policy: block` with explicit allowed endpoints. [web:130]

### Option 3: Implement Secret Rotation
Plan to rotate `TS_OAUTH_SECRET` and `LITELLM_PORT` (if you change it) regularly. Consider Bitwarden Secrets Manager for automated rotation. [web:147]

## Security Considerations

- **Ephemeral Nodes:** Tailscale nodes created via GitHub Action are temporary and cleaned up after each run, reducing long-lived exposure. [web:89]
- **Least Privilege ACLs:** The ACL restricts `tag:ci` to only the LiteLLM port on the target host—nothing else. [web:103]
- **Network Segmentation:** Your LiteLLM server never touches the public internet; it only connects to Tailscale via outbound connection. [web:107]
- **Secret Masking:** GitHub automatically redacts secret values from logs. [web:147][web:153]
- **Egress Monitoring:** StepSecurity captures all outbound DNS and network traffic for audit. [web:128][web:130]

## References

- [Tailscale GitHub Action Docs](https://tailscale.com/kb/1276/tailscale-github-action) [web:89]
- [Tailscale Serve Docs](https://tailscale.com/kb/1242/tailscale-serve) [web:107]
- [Tailscale ACL Examples](https://tailscale.com/kb/1192/acl-samples) [web:103]
- [GitHub Actions Secrets Best Practices](https://docs.github.com/actions/security-guides/using-secrets-in-github-actions) [web:153]
- [StepSecurity Harden-Runner Docs](https://docs.stepsecurity.io/harden-runner) [web:129][web:128]

## License

This is a proof-of-concept for secure CI/CD access to private infrastructure. Use as reference and adapt to your security requirements.
