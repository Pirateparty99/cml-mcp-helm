# cml-mcp Helm chart (HTTP mode) + Claude Desktop

Deploys [xorrkaz/cml-mcp](https://github.com/xorrkaz/cml-mcp) to Kubernetes in **HTTP transport mode** and connects Claude Desktop to it through `mcp-remote`.

This is an unofficial chart. It's based on the project's `INSTALLATION.md`, `Dockerfile`, and `entrypoint.sh` as of October 14, 2026, with image `xorrkaz/cml-mcp:0.32.2`.

```
Claude Desktop ── mcp-remote ──HTTPS──▶ Ingress ──▶ Service :9000 ──▶ cml-mcp pod ──HTTPS──▶ CML server
                 (adds auth headers)                                   (per-request creds)
```

## What the chart creates

| Resource | Purpose |
|---|---|
| Deployment | One pod running `cml-mcp` with `CML_MCP_TRANSPORT=http` on port 9000 |
| ConfigMap | Non-secret settings: `CML_URL`, `CML_VERIFY_SSL`, allowed URLs, session TTL |
| ConfigMap (optional) | ACL rules mounted at `/app/acl.yaml` |
| Service | ClusterIP on port 9000 |
| Ingress | TLS-terminating entry point for Claude Desktop |

No credentials are stored in the cluster. Each client sends its own CML and device credentials in request headers.

### Design choices

- **One replica, `Recreate` strategy.** MCP sessions and cached CML logins live in the pod's memory. More replicas would drop sessions without sticky routing.
- **TCP probes.** The server doesn't expose a health endpoint, so probes check that port 9000 is open.
- **Streaming-friendly Ingress.** The default annotations turn off buffering and raise timeouts for ingress-nginx. Change them if you use a different controller.
- **Pinned image tag.** Defaults to the chart's `appVersion` (0.32.2), not `latest`.

## 1. Prerequisites

- A Kubernetes cluster and `kubectl` / `helm` (v3+) access to it
- An Ingress controller (the defaults assume ingress-nginx) and a DNS name for it
- A TLS certificate for that name (cert-manager, or a Secret you create)
- **Network reachability from pods to your CML server.** This is often the real hurdle; check it first (step 2).
- On your laptop: Claude Desktop and Node.js 18+ (for `npx mcp-remote`)

## 2. Check that pods can reach CML

```bash
kubectl run cml-check --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sk -o /dev/null -w "%{http_code}\n" https://<CML_SERVER>/api/v0/system_information
```

A `200` means the network path works. A timeout means the cluster can't route to CML, so fix that before installing.

## 3. Configure values

Create `my-values.yaml`:

```yaml
cml:
  url: "https://10.10.20.50"      # your CML server
  verifySsl: false                # CML's cert is self-signed by default

ingress:
  className: nginx
  host: cml-mcp.lab.example.com
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    cert-manager.io/cluster-issuer: letsencrypt   # remove if you create the TLS Secret yourself
  tls:
    enabled: true
    secretName: cml-mcp-tls
```

**Optional: restrict tools per user**

```yaml
acl:
  enabled: true
  rules: |
    default_enabled: false
    users:
      admin: {}
      student:
        disabled_tools: [delete_cml_lab, delete_cml_node, delete_cml_user, delete_cml_group]
```

**Optional: verify CML's certificate instead of disabling verification**

```bash
openssl s_client -connect <CML_SERVER>:443 -showcerts </dev/null 2>/dev/null \
  | openssl x509 > cml-ca.crt
kubectl -n cml-mcp create secret generic cml-ca --from-file=ca.crt=cml-ca.crt
```

```yaml
cml:
  verifySsl: true
caBundle:
  enabled: true
  secretName: cml-ca
```

## 4. Install

From the folder containing the chart:

```bash
helm lint ./cml-mcp -f my-values.yaml
helm upgrade --install cml-mcp ./cml-mcp \
  --namespace cml-mcp --create-namespace \
  -f my-values.yaml
```

Check it:

```bash
kubectl -n cml-mcp get pods,svc,ingress
kubectl -n cml-mcp logs deploy/cml-mcp
```

The logs should show `All tools registered successfully` and Uvicorn listening on `0.0.0.0:9000`.

## 5. Encode your credentials

```bash
# macOS / Linux
echo -n "cml_user:cml_pass" | base64          # -> CML_AUTH_HEADER
echo -n "device_user:device_pass" | base64    # -> PYATS_AUTH_HEADER
echo -n "enable_password" | base64            # -> PYATS_ENABLE_HEADER (password only)
```

```powershell
# PowerShell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("cml_user:cml_pass"))
```

## 6. Configure Claude Desktop

Open the config file (**Settings → Developer → Edit Config**):

| OS | Path |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

Add this (also in `examples/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "Cisco Modeling Labs CML": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://cml-mcp.lab.example.com/mcp",
        "--header",
        "X-Authorization:${CML_AUTH_HEADER}",
        "--header",
        "X-PyATS-Authorization:${PYATS_AUTH_HEADER}",
        "--header",
        "X-PyATS-Enable:${PYATS_ENABLE_HEADER}"
      ],
      "env": {
        "CML_AUTH_HEADER": "Basic <base64 of cml_user:cml_pass>",
        "PYATS_AUTH_HEADER": "Basic <base64 of device_user:device_pass>",
        "PYATS_ENABLE_HEADER": "Basic <base64 of enable_password>"
      }
    }
  }
}
```

Notes:

- Keep the values in `env` and write `Header:${VAR}` with **no space after the colon**. This avoids a Windows bug that mangles spaces in `args`.
- The device headers are only needed for `send_cli_command`. If you leave out `X-PyATS-Enable`, check whether your devices need an enable password.
- If your Ingress uses a self-signed certificate, add `"NODE_TLS_REJECT_UNAUTHORIZED": "0"` to `env`.
- If you get `npx` not found, replace `"npx"` with its full path (`which npx` / `where npx`).

Then **fully quit and reopen** Claude Desktop, confirm "Cisco Modeling Labs CML" is listed in the tools menu, and try: *"Show me all my CML labs."*

## 7. Testing without an Ingress

Set `ingress.enabled: false`, then:

```bash
kubectl -n cml-mcp port-forward svc/cml-mcp 9000:9000
```

Use `http://localhost:9000/mcp` in the Desktop config and add `"--allow-http"` to `args`.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Pod `CrashLoopBackOff` | `kubectl logs` it. Usually a bad `CML_URL` value or ACL YAML syntax. |
| Desktop: server failed to start | Check Desktop's MCP logs. Add `"--debug"` to `mcp-remote` args; it logs to `~/.mcp-auth/`. |
| 401 / auth errors on tool calls | Wrong Base64, or `Basic ` prefix missing in `env`. Re-encode with `echo -n` (no trailing newline). |
| Tools list loads but calls fail with SSL errors | `cml.verifySsl: true` without a CA bundle. Set it to `false` or configure `caBundle`. |
| Calls hang or drop after ~60s | Ingress is buffering or timing out the stream. Check the annotations for your controller. |
| Timeouts reaching CML | Pod network can't route to CML. Re-run the check in step 2. |
| CLI commands fail, lab tools work | Device creds or enable header wrong, or the node isn't fully booted. |

## Security notes

- Credentials sit in plain text in `claude_desktop_config.json` and travel in headers. Always use TLS on the Ingress, and use a lab-only CML account.
- Don't set `CML_MCP_ALLOW_UNAUTHENTICATED=true` (via `extraEnv`) unless the endpoint is reachable only by you; it lets anyone who can reach it act as the configured identity.
- Consider a NetworkPolicy that limits egress from the pod to your CML server and the device management network.

## Uninstall

```bash
helm uninstall cml-mcp -n cml-mcp
```