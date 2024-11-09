# Malcure's Intrusion Prevention Plugin for ModSecurity

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Proactively block malicious IPs to prevent brute-force attacks and exploitation attempts by monitoring error responses in ModSecurity without relying on external tools like Fail2Ban.**

Malcure's Intrusion Prevention Plugin enhances your web server's security by integrating directly with ModSecurity to detect and block malicious IP addresses exhibiting suspicious behavior. By monitoring specific HTTP error responses, the plugin identifies potential brute-force attacks and proactively blocks offenders.

## Features

- **Proactive IP Blocking**: Automatically blocks IPs that exceed a configurable error threshold.
- **Error-Based Detection**: Monitors 403 and 404 HTTP response codes to identify malicious activity.
- **Customizable Settings**:
  - Configure error limits (default: 5 errors).
  - Set error tracking period (default: 30 seconds).
- **Private IP Exemption**: Excludes internal/private IP addresses to prevent false positives from legitimate internal services.
- **No External Dependencies**: Operates entirely within ModSecurity without the need for external tools like Fail2Ban.
- **Detailed Logging**: Provides comprehensive logs for auditing and monitoring purposes.
- **Easy Integration**: Simple installation and configuration with ModSecurity.


## Requirements

- **ModSecurity Web Application Firewall**:
  - **Version**: ModSecurity 3.x (recomended).
- **Web Server**:
  - **Nginx** (Recommended).
  - **Apache HTTP Server** .
- **Support for Persistent Storage and Variable Collections**:
- **IP Collection Support**: Required for tracking IP-specific data.

## How to Install the Plugin

Please refer to the [OWASP ModSecurity Core Rule Set (CRS) documentation](https://coreruleset.org/docs/concepts/plugins/#how-to-install-a-plugin) for general guidance on installing plugins.

### Installation Steps:

1. **Download the Plugin**:
   - Obtain the files from the official repository or release package.

2. **Place the Plugin Files**:
   - Copy the files into your ModSecurity plugins directory.
     - Common locations:
       - For Apache: `/etc/modsecurity.d/`
       - For Nginx with ModSecurity: `/etc/nginx/modsec/`

3. **Include the Plugin in ModSecurity Configuration**:
   - Add the following lines to your ModSecurity configuration file (e.g., `modsecurity.conf`):

     ```nginx
     Include /etc/nginx/conf.d/malcure/modsec/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
     Include /etc/nginx/conf.d/malcure/modsec/plugins/*-config.conf
     Include /etc/nginx/conf.d/malcure/modsec/plugins/*-before.conf
     Include /etc/nginx/conf.d/malcure/modsec/coreruleset/rules/*.conf
     Include /etc/nginx/conf.d/malcure/modsec/plugins/*-after.conf
     ```

     - Replace `/path/to/` with the actual path to the plugin file.

4. **Adjust Configuration if Necessary**:
   - Open the plugin file and adjust the `tx.error_limit` and `tx.error_counter_expire_period` variables if you wish to change the default settings.

5. **Restart the Web Server**:
   - Restart Nginx to apply the changes:

     ```bash
     # For Nginx
     sudo systemctl restart nginx

     # For Apache
     sudo systemctl restart apache2
     ```

6. **Verify the Installation**:
   - Check the web server and ModSecurity logs to ensure the plugin is loaded without errors.

## Disabling the Plugin

The plugin is enabled by default when included in the ModSecurity configuration. See Rule ID 9600001 in the plugin for disabling the plugin.plugin.

**Restart the Web Server**:
   - Restart Apache or Nginx to apply the changes.

## Testing That the Plugin Works

To verify that the plugin is functioning correctly:

1. **Simulate Malicious Activity**:
   - From a test client, intentionally generate 403 or 404 errors by requesting non-existent resources multiple times, exceeding the error limit.

2. **Observe Blocking Behavior**:
   - Confirm that your IP address is blocked after exceeding the error threshold.

3. **Check the Logs**:
   - Review the ModSecurity logs to see entries related to the plugin's actions.

## Reporting Issues and False Positives

If you encounter any issues, false positives, or have suggestions for improvement, please open a new issue or pull request in the official repository. Include the following details:

1. **ModSecurity Version**: Specify the version of ModSecurity you are using.
2. **Web Server**: Indicate whether you are using Apache or Nginx.
3. **Plugin Version**: Confirm the version of the plugin.
4. **Logs**: Provide relevant ModSecurity audit logs that demonstrate the issue.
5. **Description**: Explain what caused the issue or false positive and any steps to reproduce it.

## License

This plugin is licensed under the [MIT License](https://opensource.org/licenses/MIT). See the `LICENSE` file for more details.

---

**Author**: Malcure  
**Website**: [https://www.malcure.com/](https://www.malcure.com/)  
**Support Email**: [support@malcure.com](mailto:shiv@malcure.com)
