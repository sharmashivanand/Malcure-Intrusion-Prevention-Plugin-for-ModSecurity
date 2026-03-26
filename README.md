# Malcure's Intrusion Prevention Plugin for ModSecurity

[![License: Malcure](https://img.shields.io/badge/License-Malcure-264059)](https://malcure.com/about-malcure/malcure-license-template/)

**Proactively block malicious IPs to prevent brute-force attacks and exploitation attempts by monitoring error responses in ModSecurity without relying on external tools like Fail2Ban.**

Malcure's Intrusion Prevention Plugin enhances your web server's security by integrating directly with ModSecurity to detect and block malicious IP addresses exhibiting suspicious behavior. By monitoring 403 and 404 HTTP error responses, the plugin identifies potential brute-force attacks and scanners, blocks them in phase 2 before the request ever reaches the backend, and applies a progressive linear penalty that grows the block window the longer the attacker keeps probing.

## Features

- **Proactive IP Blocking**: Automatically blocks IPs that exceed a configurable error threshold, entirely within ModSecurity — no Fail2Ban or external tools required.
- **Phase 2 Early Block**: Once blocked, every subsequent request from the offending IP is denied in phase 2, before it reaches the backend (PHP, upstream, etc.), protecting server resources.
- **Progressive Linear Penalty**: The block window grows linearly with each probe the attacker sends while already blocked. An active scanner that keeps probing accumulates an ever-increasing required silence period before it is unblocked.
  - Block window formula: `base_block_duration + (blocked_probes × block_penalty_per_hit)`
- **Sliding Window**: The block expiry timer resets on every blocked request. The attacker must go completely silent and wait out the full current window to get unblocked.
- **Error-Based Detection**: Monitors 403 and 404 HTTP response codes to identify malicious activity.
- **Customizable Settings** — all tunable via `tx.*` variables (override in your site config, not in the plugin file):
  - `tx.error_limit` — errors before blocking triggers (default: `5`, blocks on the 6th).
  - `tx.error_counter_expire_period` — seconds to track errors per IP (default: `30`).
  - `tx.base_block_duration` — minimum block window in seconds (default: `30`).
  - `tx.block_penalty_per_hit` — seconds added to the block window per probe while blocked (default: `2`).
- **Private IP Exemption**: Excludes internal/private IP addresses (IPv4 and IPv6) to prevent false positives from legitimate internal services, WordPress cron, local health checks, etc.
- **Correct Expiry Reset**: Expired state is reset for any IP whose tracking window has elapsed — including IPs that accumulated errors without reaching the block threshold — ensuring stale counters never persist indefinitely.
- **Detailed Logging**: Comprehensive log messages for every rule action including current penalty, error counter, and expiry timestamp.
- **Easy Integration**: Follows the standard [OWASP CRS plugin convention](https://coreruleset.org/docs/concepts/plugins/#how-to-install-a-plugin) with `-config.conf` and `-before.conf` files.

## Requirements

- **ModSecurity Web Application Firewall**:
  - **Version**: ModSecurity 3.x (recommended). Requires libModSecurity with LMDB support for persistent IP collections.
- **Web Server**:
  - **Nginx** (recommended).
  - **Apache HTTP Server**.
- **Persistent Storage**: The plugin uses the `IP` collection to track state across requests. LMDB (or another persistent backend) must be configured in `modsecurity.conf` via `SecCollectionTimeout` and `SecDataDir` / LMDB equivalent.

## How to Install the Plugin

Please refer to the [OWASP ModSecurity Core Rule Set (CRS) documentation](https://coreruleset.org/docs/concepts/plugins/#how-to-install-a-plugin) for general guidance on installing plugins.

### Installation Steps:

1. **Download the Plugin**:
   - Clone or download this repository and copy the contents of the `plugins/` directory into your ModSecurity plugins directory.

2. **Place the Plugin Files**:
   - Copy `malcure-intrusion-prevention-before.conf` and `malcure-intrusion-prevention-config.conf` into your ModSecurity plugins directory. Typical locations:
     - Nginx + ModSecurity v3: `/etc/nginx/modsec/plugins/`
     - Apache + ModSecurity: `/etc/modsecurity.d/plugins/`

3. **Include the Plugin in ModSecurity Configuration**:
   - The plugin must be loaded in the correct order relative to the CRS. Add the following to your ModSecurity configuration file (e.g., `modsecurity.conf` or a dedicated include file), replacing `/path/to/modsec` with your actual ModSecurity directory:

     ```nginx
     # Plugin configs (loaded before CRS)
     Include /path/to/modsec/plugins/*-config.conf

     # Plugin before-rules (loaded before CRS rules)
     Include /path/to/modsec/plugins/*-before.conf

     # CRS rules
     Include /path/to/modsec/coreruleset/rules/*.conf

     # Plugin after-rules (loaded after CRS rules)
     Include /path/to/modsec/plugins/*-after.conf
     ```

4. **Tune Configuration if Necessary**:
   - The plugin ships with safe defaults. To override any setting, add `SecAction` directives in your site-specific configuration **after** the plugin config is loaded — do not edit the plugin files directly so you can update cleanly:

     ```nginx
     # Example: tighten to 3 errors, 60s base block, 5s penalty per probe
     SecAction \
         "id:9600011,\
         phase:1,\
         nolog,\
         setvar:tx.error_limit=3,\
         setvar:tx.base_block_duration=60,\
         setvar:tx.block_penalty_per_hit=5"
     ```

5. **Restart the Web Server**:

     ```bash
     # For Nginx
     sudo systemctl restart nginx

     # For Apache
     sudo systemctl restart apache2
     ```

6. **Verify the Installation**:
   - Check the web server and ModSecurity error logs to confirm the plugin loaded without errors.

## Disabling the Plugin

The plugin is enabled by default when included in the ModSecurity configuration. To disable it without removing the `Include` directives, uncomment the disable rule in `malcure-intrusion-prevention-config.conf`.

After making changes, restart the web server:

```bash
sudo systemctl restart nginx   # or apache2
```

## Testing That the Plugin Works

To verify that the plugin is functioning correctly:

1. **Simulate Malicious Activity**:
   - From a test client (not your production IP), intentionally generate 403 or 404 errors by requesting non-existent resources more than 5 times within 30 seconds.

2. **Observe Blocking Behavior**:
   - Confirm that your IP address receives 403 responses after exceeding the error threshold, and that subsequent requests are blocked immediately in phase 2.

3. **Check the Logs**:
   - Review the ModSecurity / nginx error log for entries from rules `9600030` (error counting), `9600040` (block flag set), `9600020` (phase 2 early block), and `9600015` (state reset on expiry).

     ```bash
     grep "malcure_intrusion_prevention" /var/log/nginx/error.log
     ```

## Reporting Issues and False Positives

If you encounter any issues, false positives, or have suggestions for improvement, please open a new issue or pull request in the official repository. Include the following details:

1. **ModSecurity Version**: Specify the version of ModSecurity you are using.
2. **Web Server**: Indicate whether you are using Apache or Nginx.
3. **Plugin Version**: Confirm the version of the plugin (see the `ver` tag in `malcure-intrusion-prevention-before.conf`).
4. **Logs**: Provide relevant ModSecurity / web server error log entries that demonstrate the issue.
5. **Description**: Explain what caused the issue or false positive and any steps to reproduce it.

## License

This plugin is licensed under the [Malcure License](https://malcure.com/about-malcure/malcure-license-template/). See the `LICENSE` file for more details.

---

**Author**: Malcure  
**Website**: [https://www.malcure.com/](https://www.malcure.com/?utm_source=github&utm_medium=readme&utm_campaign=malcure-intrusion-prevention-plugin)
