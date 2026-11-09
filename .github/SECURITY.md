# Security Policy

> 适用版本：v2.10.35 · 最近校对：2026-11-09

## Supported Versions

Only the latest released version of LLM API Sentinel receives security updates.
We follow [Semantic Versioning](https://semver.org/) and track the current
version in `package.json` (`2.10.35`).

| Version      | Supported          |
| ------------ | ------------------ |
| 2.10.x (latest) | :white_check_mark: |
| < 2.10       | :x:                |

## Reporting a Vulnerability

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, report them privately using **GitHub Private Vulnerability Reporting**:
open the **Security** tab of this repository and use the "Report a vulnerability"
button. If that channel is unavailable, open a private request via GitHub
Discussions or contact the repository owner through their GitHub profile
(https://github.com/Tanox).

> `package.json` has no `author` field and the repository does not publish a
> dedicated security mailbox; do not rely on either as a contact channel.

Please include the following in your report:

- A description of the vulnerability and its impact
- Steps to reproduce or a proof-of-concept
- Affected version(s)
- Any suggested remediation, if known

We will acknowledge receipt within **3 business days** and aim to provide a
remediation timeline within **7 business days**. Once fixed, we will coordinate
disclosure with you and credit the reporter (unless anonymity is requested).

## Security Best Practices

For the project's threat model, data protection, and deployment hardening
guidance, see:

- [docs/security.md](../docs/security.md) — security architecture and best practices
- [docs/env.md](../docs/env.md) — environment variable and secret handling

> The previously referenced `security_best_practices_report.md` does not exist
> in this repository; no external audit report is currently published.
