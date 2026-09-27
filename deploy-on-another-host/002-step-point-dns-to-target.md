# 002 — Point DNS to Target

## Goal

Create the DNS record for the pre-prod hostname and let it propagate before
Caddy tries to request a TLS certificate for it. Doing this early avoids
blocking later on propagation delay.

## Placeholders used in this runbook

- `<TARGET_IP>` — public IP address of the new Debian Trixie server
- `<DNS_PROVIDER_CONSOLE>` — wherever the `poland-contabo.fyi` zone is
  managed (registrar or DNS host control panel)

## Steps

### 1. Create the A record

In `<DNS_PROVIDER_CONSOLE>`, add:

| Type | Host                | Value        | TTL         |
| ---- | ------------------- | ------------ | ----------- |
| A    | `preprod.solar`      | `<TARGET_IP>` | 300 (5 min) |

This resolves `preprod.solar.poland-contabo.fyi` to the target server. A
short TTL is used deliberately so that, if the target IP changes before this
record is finalized, corrections propagate quickly.

### 2. Wait for propagation

```bash
watch -n 15 \
    "dig +short preprod.solar.poland-contabo.fyi"
```

> What this does: `dig +short` queries DNS and prints just the resolved
> IP(s); `watch -n 15` re-runs that every 15 seconds so you can see the
> record go live without manually re-running the command.

## Verification

```bash
dig +short preprod.solar.poland-contabo.fyi
```

Expect the output to be exactly `<TARGET_IP>`. Confirm from an external
resolver as well (e.g. `dig @1.1.1.1 +short preprod.solar.poland-contabo.fyi`)
since local/ISP DNS caches can return stale results even after the
authoritative record is correct.

> Caddy's automatic HTTPS (see
> [009-step-install-configure-caddy.md](009-step-install-configure-caddy.md))
> will fail its ACME HTTP-01 challenge if this record isn't resolvable from
> the public internet, so don't proceed to that step until this verification
> passes.
