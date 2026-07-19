<div align="center">

# 🛡️ Wazuh Docs Hub

### Documentation, Integrations & Open Source Security Tooling

A curated knowledge base for deploying, configuring, and extending **Wazuh** — covering core setup, third-party integrations, and complementary open-source security tools.

[![Wazuh](https://img.shields.io/badge/Wazuh-4.x-1E88E5?style=for-the-badge&logo=wazuh&logoColor=white)](https://wazuh.com/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)
[![Docs](https://img.shields.io/badge/Docs-Maintained-success?style=for-the-badge)](#-table-of-contents)
[![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-orange?style=for-the-badge)](#-contributing)
[![Made with Markdown](https://img.shields.io/badge/Made%20with-Markdown-1f425f?style=for-the-badge&logo=markdown)](#)

</div>

---

## 📖 About

This repository is a personal / community **documentation hub** for everything related to **Wazuh** — the open-source security platform for threat detection, integrity monitoring, incident response, and compliance.

It's organized to serve three purposes:

- 📘 **Core Wazuh documentation** — installation, architecture, rules, decoders, and hardening notes
- 🔗 **Integrations** — connecting Wazuh with SIEMs, ticketing systems, cloud providers, and messaging platforms
- 🧰 **Open source tooling** — notes, guides, and configs for complementary security tools used alongside Wazuh

Whether you're setting up your first Wazuh instance or building out a full detection pipeline, you'll find structured, practical notes here.

---

## 📑 Table of Contents

- [About](#-about)
- [Repository Contents](#-repository-contents)
- [Getting Started](#-getting-started)
- [Contributing](#-contributing)
- [Resources](#-resources)
- [License](#-license)

---

## 🗂️ Repository Contents

| Document | Description |
|---|---|
| 📄 [`complete_soc_automation_lab_installation_and...md`](complete_soc_automation_lab_installation_and....md) | End-to-end SOC automation lab — full install & setup walkthrough |
| 📄 [`Wazuh_custom_email_configuration.md`](Wazuh_custom_email_configuration.md) | Configuring custom SMTP email alerting in Wazuh |
| 📄 [`sudo_wazuh.md`](sudo_wazuh.md) | Monitoring and alerting on `sudo` command usage with Wazuh |
| 📄 [`Auditd with wazuh.md`](Auditd%20with%20wazuh.md) | Integrating Linux `auditd` logs with Wazuh for deep system auditing |
| 📄 [`Log, Track, and Alert Every Linux Command ....md`](<Log, Track, and Alert Every Linux Command with Wazuh.md>) | Capturing and alerting on every executed Linux shell command |
| 📄 [`misp-integration.md`](misp-integration.md) | Connecting Wazuh with MISP for threat intelligence enrichment |
| 📄 [`modsecurity-geoblocking-docs.md`](modsecurity-geoblocking-docs.md) | Geo-blocking with ModSecurity on Nginx |
| 📄 [`Fail2Ban + ModSecurity Setup for Nginx.md`](<Fail2Ban + ModSecurity Setup for Nginx.md>) | Hardening Nginx with Fail2Ban and ModSecurity together |
| 📄 [`wazuh_active_response_for_known_web_attacks.md`](wazuh_active_response_for_known_web_attacks.md) | Active response rules for common/known web attack patterns |
| 📄 [`wafcontrol with juiceshop.md`](<wafcontrol with juiceshop.md>) | Testing WAF rules and control against OWASP Juice Shop |
| 📄 [`suricata-ids-ips with wazuh.md`](<suricata-ids-ips with wazuh.md>) | Deploying Suricata IDS/IPS and forwarding alerts to Wazuh |
| 📄 [`nDPI_integration_with_wazuh.md`](nDPI_integration_with_wazuh.md) | Deep packet inspection with nDPI feeding into Wazuh |
| 📄 [`crowdsec-docs.md`](crowdsec-docs.md) | CrowdSec setup and integration alongside Wazuh |
| 📄 [`n8n-docs.md`](n8n-docs.md) | Automating security workflows with n8n |

> 📌 File names above mirror the repository exactly — click through to open each doc.

---

## 🚀 Getting Started

## 🤝 Contributing

Contributions, corrections, and new integration guides are welcome!

1. Fork the repository
2. Create a branch (`git checkout -b docs/your-topic`)
3. Add or edit documentation
4. Submit a pull request with a clear description

Please keep docs concise, tested, and formatted consistently with existing files.

---

## 📚 Resources

- [Official Wazuh Documentation](https://documentation.wazuh.com/)
- [Wazuh GitHub](https://github.com/wazuh/wazuh)
- [Wazuh Community Slack](https://wazuh.com/community/join-us-on-slack/)

---

## 📄 License

This repository is licensed under the [MIT License](LICENSE). Documentation content is provided as-is for educational and operational reference.

---

<div align="center">

**⭐ If this repository helps you, consider giving it a star!**

</div>
