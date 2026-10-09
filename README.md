# ☁️ Cloud Security Journey

I'm **Talal Ahmed** — a BS Cybersecurity student at HITEC University, Pakistan.
This repository documents my hands-on path toward becoming a **Cloud Security
Engineer**, one real lab at a time: AWS security labs, web security practice
(PortSwigger), and the lessons learned from each mistake along the way.

> Goal: Cloud Security Engineer @ AWS (London) — apply 2027, after graduating in 2028.

## 🧪 Labs

| # | Lab | What it covers |
|---|-----|---------------|
| 01 | [First EC2 — Secure Launch & IMDSv2](aws-labs/01-first-ec2-secure-launch/) | Least-privilege security groups, EC2 Instance Connect, IMDSv2 token flow, the Capital One lesson |

More labs coming: IAM deep-dive, S3 security, PortSwigger SSRF series.

## 🔐 Rules I work by

- **Never** commit private keys (`.pem`), `.env` files, or credentials — see `.gitignore`.
- Least privilege by default: open the smallest access that works, nothing more.
- Practice resources get **terminated** when the lab ends — no idle attack surface, no wasted credits.
- Account IDs and personal IPs are redacted from screenshots before publishing.

## 🛠️ Also on my GitHub

- **AWS S3 Security Guarddog** — checks S3 buckets for public-access misconfigurations
- **Cisco Security Auditor** — audits network device configs for common weaknesses
