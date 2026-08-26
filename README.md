# homelab-iac

Reproducible, version-controlled home lab infrastructure — built and documented as part of a hands-on Cyber Defense portfolio (CompTIA Network+/Security+, Roane State Community College, AAS expected May 2027).

This repo tracks the configuration for a self-hosted home lab running on a single always-on Ubuntu server, with the goal of making the entire environment reproducible from scratch, monitored, and eventually triaged with the help of a locally-run LLM. It's built and documented in three stages:

1. **Infrastructure as Code** (this stage) — core services defined as Docker Compose, secrets excluded from version control, host provisioning via Ansible (in progress)
2. **SIEM / log monitoring** (planned) — Wazuh, evaluated against Security Onion, ingesting logs from the services below
3. **AI-assisted alert triage** (planned) — a locally-run LLM (via Ollama) that summarizes and prioritizes SIEM alerts in plain English

---

## Why this exists

I'm building toward a Cyber Defense career without formal IT work experience yet. This lab is my substitute for that experience — a real, running environment where I make real infrastructure decisions, break things, fix them, and document the reasoning, not just the commands. Every major decision below (tool choices, trade-offs, things I got wrong the first time) is documented on purpose, because the reasoning is the actual skill being demonstrated.

---

## Architecture

```
                        ┌─────────────────────────┐
                        │   AT&T BGW320-500        │
                        │   192.168.1.254 (router) │
                        └────────────┬─────────────┘
                                     │
                     ┌───────────────┴───────────────┐
                     │   Ubuntu Server (11elenovo)     │
                     │   192.168.1.10                  │
                     │   Intel i3-7100U / 8GB RAM       │
                     └───────────────┬───────────────┘
                                     │
        ┌────────────┬──────────────┼──────────────┬────────────┐
        │             │              │              │            │
   ┌────▼────┐  ┌─────▼─────┐  ┌────▼─────┐  ┌──────▼──────┐    │
   │ Pi-hole │  │  Unbound  │  │Portainer │  │Nginx Proxy  │    │
   │ DNS/DHCP│─▶│ Recursive │  │  Docker  │  │  Manager    │    │
   │ (host   │  │ Resolver  │  │   Mgmt   │  │ (reverse    │    │
   │ network)│  │  :5335    │  │   UI     │  │  proxy/SSL) │    │
   └─────────┘  └───────────┘  └──────────┘  └─────────────┘    │
                                                                  │
                                                          ┌───────▼───────┐
                                                          │  security/     │
                                                          │  WireGuard VPN │
                                                          │ (remote access)│
                                                          └────────────────┘

   lab/  → isolated DVWA + MariaDB (deliberately vulnerable app for
           hands-on security practice — not exposed outside the lab network)

   secplus/ → self-built Security+ SY0-701 flashcard study tool (static HTML)
```

**DNS query flow:** client → Pi-hole (:53) → Unbound (:5335) → root DNS servers → answer. Pi-hole handles ad-blocking/DHCP; Unbound does the actual recursive resolution so no third-party DNS provider ever sees lookups.

**HTTPS access flow:** external request → Nginx Proxy Manager (SSL termination) → internal service. NPM lets services be reached at clean HTTPS URLs (e.g. `https://portainer.lan`) instead of raw IP:port with self-signed cert warnings.

---

## Stack

| Component | Image | Purpose |
|---|---|---|
| Pi-hole | `pihole/pihole:latest` | DNS sinkhole, ad-blocking, DHCP server |
| Unbound | `mvance/unbound:latest` | Recursive DNS resolver (privacy — no upstream resolver logging queries) |
| Portainer CE | `portainer/portainer-ce:latest` | Docker management UI |
| Nginx Proxy Manager | `jc21/nginx-proxy-manager:latest` | Reverse proxy + automated SSL |
| WireGuard | (native, not yet containerized) | Remote VPN access to the lab network |
| DVWA + MariaDB | (isolated lab network) | Deliberately vulnerable app for hands-on offensive/defensive practice |

**Host:** Lenovo ThinkPad 11e, Intel i3-7100U (2C/4T), 8GB RAM, 237GB storage, no GPU — a deliberate constraint that shaped later architecture decisions (see Project 2 and 3 notes below).

---

## Repo structure

```
homelab-iac/
├── core/           # Pi-hole, Unbound, Nginx Proxy Manager — always-on core services
│   └── unbound/    # Unbound resolver config
├── lab/            # DVWA + database — isolated practice environment
├── secplus/        # Security+ SY0-701 flashcard study tool
├── security/       # WireGuard VPN (config/keys excluded — see Secrets, below)
└── .gitignore      # Excludes all secrets, keys, and certs from version control
```

---

## Secrets handling

No real credentials, API keys, or private keys are committed to this repo. Every service that needs a secret reads it from a local `.env` file, which is excluded via `.gitignore` and never pushed. Where a `.env.example` exists (e.g. `lab/.env.example`), it documents which variables are required without exposing real values — copy it to `.env` and fill in your own values to reproduce this environment.

Before this repo was ever initialized with `git init`, the full directory tree was audited for hardcoded secrets and private keys. Everything sensitive found in that audit (a Pi-hole admin password, two database passwords, WireGuard private keys, a TLS private key) was scoped out via `.gitignore` *before* the first commit, not after — so nothing sensitive ever entered Git history, even temporarily.

Git commits are authenticated via SSH key, not HTTPS token, from the host server to GitHub.

---

## Design decisions worth knowing about

- **Unbound behind Pi-hole, not a third-party DNS provider** — keeps DNS resolution fully recursive and local; no upstream resolver (Google, Cloudflare, etc.) sees raw query logs.
- **Pi-hole runs in `network_mode: host`** — required for DHCP to broadcast correctly; documented trade-off is that Compose `ports:` mappings are ignored for this service.
- **NPM runs HTTPS-only, no port 80** — Pi-hole already owns port 80 on this host for its admin UI and DNS block pages, so NPM was deliberately configured to skip HTTP entirely rather than fight for the port.
- **`.gitignore` scoped narrowly, not broadly** — e.g., `security/wireguard/*.conf` is excluded specifically (because WireGuard configs embed private keys inline), while `core/unbound/unbound.conf` is intentionally tracked, since it's meaningful configuration with no embedded secrets. A blanket `*.conf` rule would have hidden useful IaC along with the real risk.
- **No network segmentation yet** — the home network sits behind a stock AT&T BGW320-500 with no VLAN support. All lab traffic currently shares one broadcast domain with the rest of the home network. Documented here deliberately as a known limitation, not something to quietly ignore — a managed switch with VLAN support is a planned future improvement.

---

## Roadmap

- [x] **Project 1 — IaC:** Git version control, secrets audit, `.gitignore` hygiene, SSH-based push access
- [ ] **Project 1 — IaC:** Ansible playbook for base host provisioning (Docker install, firewall baseline)
- [ ] **Project 1 — IaC:** WireGuard converted from standalone config to a tracked Compose service
- [ ] **Project 1 — IaC:** Full teardown/rebuild verified on a throwaway VM
- [ ] **Project 2 — SIEM:** Wazuh deployed (chosen over Security Onion — lighter resource footprint fits this host's 8GB RAM ceiling; Security Onion's network-tap-centric model doesn't fit a host-only, non-mirrored network)
- [ ] **Project 2 — SIEM:** Log ingestion from Pi-hole, Unbound, NPM, Portainer; custom detection rules tuned against real traffic
- [ ] **Project 3 — AI Triage:** Local LLM (Ollama, run on a separate GPU-equipped machine due to this host's hardware constraints) summarizing and prioritizing Wazuh alerts in plain English

---

## Reproducing this environment

Currently: clone the repo, copy each `.env.example` to `.env`, fill in your own values, and run `docker compose up -d` from each stack directory (`core/`, `lab/`).

Full one-command reproducibility (bare Ubuntu → running lab) is the goal of the in-progress Ansible playbook — this section will be updated once that's complete.

---

## About the author

River Garren — Cyber Defense AAS student (Roane State Community College, expected May 2027), CompTIA Network+ and Security+ certified. This lab is my hands-on lab notebook, built to demonstrate real infrastructure, monitoring, and security-engineering decisions rather than just certifications on paper.
