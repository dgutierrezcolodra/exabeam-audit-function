# OCI Audit Logs to Exabeam Webhook Collector

Sample OCI Function reference for forwarding Service Connector Hub audit log batches to an Exabeam JSON Webhook Collector.

## Disclaimer

This repository is a sample implementation and reference template for an OCI Audit Logs to Exabeam Webhook Collector integration. It is intended to show one possible approach and should not be treated as production-ready code as-is. Before any production use, review, test, harden, and adapt it to the target tenancy, network, IAM, Vault, logging, security, operational, and Exabeam collector requirements.

## Behavior

```text
OCI Audit Logs -> Service Connector Hub -> OCI Function -> Exabeam JSON Webhook Collector
```

The deployable OCI Function files are under:

```text
function/
```

Use an Exabeam JSON collector, not a RAW collector. The Function defaults to:

```text
https://api2.us-east.exabeam.cloud/cloud-collectors/v1/logs/json
```

## Required configuration

- `VAULT_SECRET_OCID` or `BEARER_TOKEN`: required bearer token used to authenticate with the Exabeam JSON collector.
  - Prefer `VAULT_SECRET_OCID` for deployed environments. It must point to an OCI Vault secret whose plaintext value is the Exabeam collector bearer token.
  - Use `BEARER_TOKEN` only for direct tests or simple deployments. Set it to the token value only, without quotes, spaces, newlines, or an extra `Bearer` prefix.

## Optional configuration

- `WEBHOOK_URL`: Exabeam JSON endpoint override. Default: `https://api2.us-east.exabeam.cloud/cloud-collectors/v1/logs/json`.
- `WEBHOOK_GZIP`: set to `true` to gzip the request body before sending it. Default: `false`.
- `WEBHOOK_MAX_ATTEMPTS`: total delivery attempts. Default: `2`.
- `WEBHOOK_CONNECT_TIMEOUT_SECONDS`: TCP/TLS connect timeout. Default: `3.05`.
- `WEBHOOK_READ_TIMEOUT_SECONDS`: response read timeout. Default: `20.0`.
- `WEBHOOK_RETRY_DELAY_SECONDS`: base delay before retry. Default: `2.0`.
- `TOKEN_CACHE_TTL_SECONDS`: local bearer token cache TTL when using Vault. Default: `300`.
- `LOG_LEVEL`: Python logging level. Default: `INFO`.

## Notes

For OCI Audit Logs, use an Exabeam JSON collector with Exabeam's `/logs/json` endpoint. This function does not support `/logs/raw`.

For private subnets, the function subnet needs outbound internet access, typically through a NAT Gateway with egress TCP 443 allowed by the route table and NSG/security list.

## Test the Function

Deploy from the Function directory:

```bash
cd function
fn deploy --app audit-forwarder-app
```

The Function name is defined in `function/func.yaml`:

```text
audit-webhook-forwarder
```

Configure the Function. For a direct token test:

```bash
fn config function audit-forwarder-app audit-webhook-forwarder WEBHOOK_URL \
  https://api2.us-east.exabeam.cloud/cloud-collectors/v1/logs/json

fn config function audit-forwarder-app audit-webhook-forwarder BEARER_TOKEN \
  '<JWT_TOKEN_ONLY>'
```

For Vault-based configuration:

```bash
fn config function audit-forwarder-app audit-webhook-forwarder WEBHOOK_URL \
  https://api2.us-east.exabeam.cloud/cloud-collectors/v1/logs/json

fn config function audit-forwarder-app audit-webhook-forwarder VAULT_SECRET_OCID \
  '<secret_ocid>'
```

Inspect the Function configuration:

```bash
fn inspect function audit-forwarder-app audit-webhook-forwarder
```

Create a sample payload:

```bash
cat > payload.json <<'EOF'
[
  {
    "datetime": "2026-05-29T10:00:00Z",
    "logContent": {
      "data": {
        "eventName": "TestAuditEvent",
        "resourceName": "cloud-shell-test",
        "status": "200"
      },
      "id": "test-event-001",
      "source": "com.oraclecloud.audit",
      "type": "com.oraclecloud.audit"
    }
  }
]
EOF
```

Invoke the Function:

```bash
fn invoke audit-forwarder-app audit-webhook-forwarder < payload.json
```

Check the Function invocation logs for `Webhook delivery succeeded: status=200`.

## Troubleshooting

Connector Hub only reports that the Function invocation failed. Check the OCI Function invocation logs to find the real cause.

In the OCI Console:

1. Go to **Developer Services**.
2. Open **Functions**.
3. Select the application and function.
4. Open **Logs** or the linked **OCI Logging** log.
5. Filter around the same timestamp as the Service Connector Hub run.

Useful messages from this function:

- `Forwarding N audit event(s) to Exabeam webhook`
- `Webhook delivery attempt ... endpoint=... payload_type=... event_count=... bytes=... content_type=...`
- `Webhook delivery failed ... HTTP 400/401/403/415/...`
- `Webhook transient exception ...` for connection or timeout problems

The logs never print the bearer token.

Common causes:

- `HTTP 400`: payload or content-type issue.
- `HTTP 401` or `HTTP 403`: invalid token, wrong collector token, or malformed JWT.
- Timeout or connection errors: Function subnet egress, NAT Gateway, NSG/security list, route table, or DNS.
