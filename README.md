# Malcure Bot & Brute-Force Shield for ModSecurity

[![License: Malcure](https://img.shields.io/badge/License-Malcure-264059)](https://malcure.com/about-malcure/malcure-license-template/)

A suite of two ModSecurity plugins that work together to block bots, brute-force attacks, and exploitation attempts — entirely within the WAF, without Fail2Ban or any external tools.

| Plugin | File pair | Rule ID range | What it blocks |
|---|---|---|---|
| **Malcure Intrusion Prevention** | `malcure-intrusion-prevention-*.conf` | 9,600,000–9,600,999 | IPs generating excessive 403/404 errors (scanners, probers) |
| **Malcure WP-Login Protection** | `malcure-wplogin-protection-*.conf` | 9,601,000–9,601,999 | IPs hammering `wp-login.php`, `xmlrpc.php`, or any configured endpoint |

Both plugins share the same architecture: phase 2 early block, sliding expiry window, and a progressive linear penalty that grows the block window on every probe received while an IP is already blocked.

---

## Plugin 1 — Malcure Intrusion Prevention (v1.3.0)

Monitors HTTP response codes. When an IP accumulates more than `tx.mip_error_limit` responses with status 403 or 404 within the tracking window, it is blocked. Subsequent requests are denied in phase 2 before the request ever reaches the backend.

### How it works

1. **Count** (phase 3, rule 9600030) — increments `ip.mip_error_counter` on every 403 or 404 response; refreshes the tracking window expiry.
2. **Flag** (phase 3, rule 9600040) — sets `ip.mip_block_ip=1` when `ip.mip_error_counter > tx.mip_error_limit`.
3. **Deny (first trip)** (phase 3, rule 9600050) — blocks the request that just crossed the threshold.
4. **Deny (all subsequent)** (phase 2, rule 9600020) — blocks every future request from that IP before the backend sees it; refreshes and extends the block expiry.
5. **Penalty** (phase 2, rule 9600016) — adds `tx.mip_block_penalty_per_hit` to `ip.mip_block_penalty` on every blocked probe.
6. **Expire** (phase 2, rule 9600015) — clears all state when the tracking/block window has genuinely elapsed.

### Tunable variables

Override any of these in your site config **before** the plugin loads. Do not edit the plugin files directly.

| Variable | Default | Description |
|---|---|---|
| `tx.mip_error_limit` | `5` | Errors before blocking triggers (blocks on the 6th) |
| `tx.mip_ec_expire_period` | `30` | Seconds to track errors per IP |
| `tx.mip_base_block_duration` | `30` | Minimum block window in seconds |
| `tx.mip_block_penalty_per_hit` | `2` | Seconds added to the block window per probe while blocked |

Block window formula: `tx.mip_base_block_duration + (blocked_probes × tx.mip_block_penalty_per_hit)`

### Configuration example

```nginx
# Tighten to 3 errors, 60s base block, 10s penalty per probe
SecAction \
    "id:9600011,\
    phase:1,\
    nolog,\
    pass,\
    setvar:tx.mip_error_limit=3,\
    setvar:tx.mip_ec_expire_period=60,\
    setvar:tx.mip_base_block_duration=60,\
    setvar:tx.mip_block_penalty_per_hit=10"
```

---

## Plugin 2 — Malcure WP-Login Protection (v1.1.0)

Monitors requests to sensitive WordPress endpoints. When an IP exceeds `tx.wpl_request_limit` requests to any protected endpoint within the tracking window, all traffic from that IP is blocked — regardless of HTTP method (GET, POST, XMLRPC, etc.).

### How it works

1. **Count** (phase 2, rule 9601030) — increments `ip.wpl_counter` on every request matching `tx.wpl_endpoint_pattern`; refreshes the tracking window expiry.
2. **Flag** (phase 2, rule 9601040) — sets `ip.wpl_block_ip=1` when `ip.wpl_counter > tx.wpl_request_limit`; overwrites expiry with the full block duration.
3. **Deny (first trip)** (phase 2, rule 9601050) — blocks the request that just crossed the threshold.
4. **Deny (all subsequent)** (phase 2, rule 9601020) — blocks every future request from that IP before the backend sees it; refreshes and extends the block expiry.
5. **Penalty** (phase 2, rule 9601016) — adds `tx.wpl_block_penalty_per_hit` to `ip.wpl_block_penalty` on every blocked probe.
6. **Expire** (phase 2, rule 9601015) — clears all state when the tracking/block window has genuinely elapsed.

### Tunable variables

| Variable | Default | Description |
|---|---|---|
| `tx.wpl_request_limit` | `5` | Requests to protected endpoints allowed per IP before blocking |
| `tx.wpl_window` | `30` | Tracking window in seconds; resets on every matching request |
| `tx.wpl_base_block_duration` | `30` | Minimum block window in seconds |
| `tx.wpl_block_penalty_per_hit` | `2` | Seconds added to the block window per probe while blocked |
| `tx.wpl_endpoint_pattern` | `(?:wp-login\|xmlrpc)\.php$` | Regex matched against `REQUEST_FILENAME` |

Block window formula: `tx.wpl_base_block_duration + (blocked_probes × tx.wpl_block_penalty_per_hit)`

### Configuration example

```nginx
# 3 attempts, 24-hour base block, 5-minute penalty per probe, also protect wp-cron.php
SecAction \
    "id:9601011,\
    phase:1,\
    nolog,\
    pass,\
    setvar:tx.wpl_request_limit=3,\
    setvar:tx.wpl_window=30,\
    setvar:tx.wpl_base_block_duration=86400,\
    setvar:tx.wpl_block_penalty_per_hit=300,\
    setvar:'tx.wpl_endpoint_pattern=(?:wp-login|xmlrpc|wp-cron)\.php$'"
```

---

## Requirements

- **libModSecurity 3.x** compiled with `--with-lmdb` for persistent IP collection storage across requests.
- **OWASP CRS** installed and configured (plugins load around CRS using the standard `-before.conf` / `-config.conf` convention).
- `ENABLE_DEFAULT_COLLECTIONS` set to `1` in `crs-setup.conf` (rule 900130).
- **Nginx** or **Apache HTTP Server**.

---

## Installation

Follow the [OWASP CRS plugin installation guide](https://coreruleset.org/docs/concepts/plugins/#how-to-install-a-plugin) for general context.

### 1. Copy the plugin files

```bash
cp plugins/malcure-intrusion-prevention-before.conf  /path/to/modsec/plugins/
cp plugins/malcure-intrusion-prevention-config.conf  /path/to/modsec/plugins/
cp plugins/malcure-wplogin-protection-before.conf    /path/to/modsec/plugins/
cp plugins/malcure-wplogin-protection-config.conf    /path/to/modsec/plugins/
```

Typical plugin directory locations:
- Nginx + ModSecurity v3: `/etc/nginx/modsec/plugins/`
- Apache + ModSecurity: `/etc/modsecurity.d/plugins/`

### 2. Verify your ModSecurity include order

The plugin files must be loaded in the correct position relative to CRS:

```nginx
Include /path/to/modsec/plugins/*-config.conf
Include /path/to/modsec/plugins/*-before.conf
Include /path/to/modsec/coreruleset/rules/*.conf
Include /path/to/modsec/plugins/*-after.conf
```

### 3. Tune (optional)

Add `SecAction` overrides in your site vhost config or a dedicated setup file loaded **before** the plugin config. See the configuration examples above.

### 4. Restart the web server

```bash
sudo systemctl restart nginx    # or apache2
```

### 5. Verify

Check the error log to confirm both plugins loaded without errors:

```bash
grep -E "malcure_intrusion_prevention|malcure_wplogin_protection" /var/log/nginx/error.log
```

---

## Disabling a plugin

To disable either plugin without removing its `Include` directives, uncomment the `SecRule` disable block in the relevant `-config.conf` file, then restart the web server.

---

## Private IP exemption

Both plugins automatically exempt all private and loopback addresses from tracking, so WordPress Cron, health checks, and local services are never blocked:

- IPv4: `127.x`, `10.x`, `192.168.x`, `172.16–31.x`
- IPv6: `::1`, `fe80::`, `fd::`

---

## Testing

### Intrusion Prevention

From a non-production IP, request a non-existent path more than 5 times within 30 seconds to trigger 404 errors. Confirm you receive a 403 after the threshold and that subsequent requests are also blocked immediately. Check the logs:

```bash
grep "malcure_intrusion_prevention" /var/log/nginx/error.log
# Key rule events: 9600030 (COUNT), 9600040 (BLOCK), 9600020 (DENY), 9600015 (EXPIRE)
```

### WP-Login Protection

From a non-production IP, send more than 5 requests to `wp-login.php` within 30 seconds. Confirm blocking triggers. Check the logs:

```bash
grep "malcure_wplogin_protection" /var/log/nginx/error.log
# Key rule events: 9601030 (COUNT), 9601040 (BLOCK), 9601020 (DENY), 9601015 (EXPIRE)
```

---

## Reporting Issues

Open a GitHub issue and include:

1. **ModSecurity version** and whether it was compiled with `--with-lmdb`.
2. **Web server** (Nginx or Apache) and version.
3. **Plugin version** (see the `ver` tag in the relevant `-before.conf` file).
4. **Relevant log lines** from the ModSecurity / web server error log.
5. **Description** of the issue and steps to reproduce.

---

## License

Licensed under the [Malcure License](MALCURE-LICENSE.md) — free to use, modify, and distribute. See the license file for the full terms.

---

**Author**: [Malcure](https://malcure.com/?utm_source=github&utm_medium=readme&utm_campaign=malcure-bot-brute-force-shield)
