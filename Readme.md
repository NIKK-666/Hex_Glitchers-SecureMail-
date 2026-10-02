# SecureMailScope

**AI-Assisted Cryptographic Security Posture Assessment for Secure Email Communications**

> Smart India Hackathon 2026 · Problem Statement **26159** · Theme: Blockchain & Cybersecurity · Category: Software
> Team **DATA_HEX_GLITCHERS** (Team ID: 158434)

SecureMailScope passively analyses captured email traffic (PCAP files) from SMTP, IMAP and POP3, reconstructs the TLS handshakes and X.509 certificates inside it, and tells an analyst whether each mail server is configured securely, with evidence and a plain-language explanation for every finding. It runs fully offline.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Idea / Solution](#2-ideasolution)
3. [Unique Value Propositions](#3-unique-value-propositions)
4. [Technical Approach](#4-technical-approach)
5. [System Architecture](#5-system-architecture)
6. [Tech Stack](#6-tech-stack)
7. [Feasibility](#7-feasibility)
8. [Viability](#8-viability)
9. [Challenges and Solutions](#9-challenges-and-solutions)
10. [Impact and Benefits](#10-impact-and-benefits)
11. [Comparison: Existing vs Proposed](#11-comparison-existing-vs-proposed-solutions)
12. [Dashboard](#12-dashboard)
13. [Research and References](#13-research-and-references)
14. [Team](#14-team)

---

## 1. Problem Statement

Email travels from the sender's mail server through routers across the internet to the receiver's server, and then to an IMAP/POP3 server. Each hop relies on TLS (often negotiated via STARTTLS) to protect the message. Whether that protection is actually configured well is hard to see.

Current practice has these gaps:

- **Static rulebooks** flag known-bad ciphers and versions, but treat every session independently. They miss combination risks and repeated misconfigurations across an estate.
- **Session-by-session analysis** misses shared infrastructure risk.
- **Packet analyzers** decode protocols and show that a handshake happened, but never tell an analyst whether it was configured securely.

NTRO's concern is detecting **TLS downgrade attempts**, **MITM interception**, and **cryptographic non-compliance** on email infrastructure.

---

## 2. Idea/Solution

- **Passive analysis** of captured email traffic (PCAP files) from SMTP, IMAP and POP3 to assess cryptographic security.
- **STARTTLS detection**: finds the STARTTLS transition and reconstructs TLS handshakes and X.509 certificates for validation.
- **Rule engine first**: flags deprecated TLS versions, weak ciphers and certificate issues against known standards (NIST, RFC 8996).
- **Graph correlation**: links servers, certificates and sessions to catch shared misconfiguration across infrastructure.
- **ML risk ranking**: XGBoost fuses graph embeddings with an Isolation Forest anomaly score to rank risk.
- **Explainability**: SHAP and SubgraphX explain each finding with the exact contributing evidence, rewritten in plain language by a **local LLM**.

---

## 3. Unique Value Propositions

| UVP | What it means |
|---|---|
| **Infrastructure-wide correlation** | Models mail servers, certificates and sessions as a connected graph, surfacing shared misconfiguration that session-independent tools structurally cannot see. |
| **Scales without retraining** | New servers and certificates are scored the moment they appear (inductive GraphSAGE embeddings). |
| **Evidence-backed** | Every risk score is traceable to the exact handshake and certificate fields that produced it, with an optional plain-language explanation layered on top. |

---

## 4. Technical Approach

The processing pipeline has seven stages:

| # | Stage | What it does |
|---|---|---|
| 1 | **Data Ingestion** | Parses bulk PCAP captures containing SMTP/IMAP/POP3 traffic, offline. |
| 2 | **Correlation Engine** | Detects STARTTLS negotiation and reassembles TCP/TLS handshake streams. |
| 3 | **Neo4j and PostgreSQL** | Neo4j stores the entity graph (servers, certificates, sessions). PostgreSQL stores tabular features, `confidence_tier` and the audit trail. |
| 4 | **GraphSAGE Embedding** | Generates inductive node embeddings for servers and certificates. |
| 5 | **XGBoost + Isolation Forest** | Combines GraphSAGE embeddings with handshake/certificate features to score cryptographic risk. Isolation Forest provides an unsupervised anomaly score with no labels needed. |
| 6 | **SHAP Explanation** | Produces feature-level explanations for every flagged session, rewritten in plain language. |
| 7 | **Dashboard** | Displays ranked, explained alerts and an interactive session/certificate link-analysis graph. |

Supporting components:

- **Entity clustering**: groups servers by shared certificates to reveal common infrastructure.
- **Structural pattern matching**: flags downgrade and MITM behaviour.
- **Risk propagation**: spreads risk across the graph.
- **Visibility tagging**: each session is tagged full, partial or resumed, instead of assuming uniform visibility.

---

## 5. System Architecture

PCAP → STARTTLS/TCP Rebuild → X.509 + Handshake Extract → PostgreSQL + Neo4j
     → Entity Clustering → GraphSAGE + Isolation Forest + Pattern Matching
     → XGBoost Risk Score → Risk Propagation → SHAP/SubgraphX + Local LLM
     → React Dashboard (ranked, explained alerts)

<img width="952" height="968" alt="26159 pptx (3)" src="https://github.com/user-attachments/assets/adf37484-1547-46e3-9df2-4fcbef6f9477" />


**Security and deployment:** fully offline, Linux-hosted, zero external API calls. Models, rules and the Docker image are bundled for air-gapped installation.

---

## 6. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, Tailwind CSS |
| Backend / API | Python, FastAPI |
| Packet and TLS parsing | Scapy, X.509 parsing |
| Graph ML | PyTorch-Geometric (GraphSAGE) |
| Risk scoring | XGBoost, Isolation Forest |
| Explainability | SHAP, SubgraphX, local LLM |
| Databases | Neo4j (graph), PostgreSQL (tabular and audit) |
| Deployment | Docker, Linux, air-gapped |

---

## 7. Feasibility

### Technical Feasibility
- **Offline-ready:** PCAP parsing, TLS/X.509 parsing and inference run CPU-only on a single mid-range server.
- **Air-gapped packaging:** dependencies, model artifacts and the Docker image are all bundled offline.
- **Mature, proven open-source stack:** Scapy, PyTorch-Geometric, GraphSAGE, Isolation Forest, XGBoost, Neo4j, PostgreSQL, SHAP.

### Economic Feasibility
- **Fully open-source stack:** zero licensing cost.
- **No cloud/GPU spend:** runs on standard on-prem hardware.
- **Low upkeep:** no recurring API or threat-feed subscription fees.

---

## 8. Viability

### Operational Viability
- Offline model/rule updates are moved through **signed, reviewed artifacts** on approved media.
- **Synthetic lab PCAPs** (known-good plus deliberately misconfigured TLS) generate labelled training data without touching real sensitive traffic.
- Scales from a single-node pilot to a multi-node LAN cluster and then full NTRO integration.

### Maintenance and Scalability
- **Rollout path:** pilot → LAN cluster → full NTRO integration.
- The **rule-based baseline stays live** as a fallback while the ML layer is tuned.
- GraphSAGE's inductive design cuts how often full retraining is needed as the mail estate grows.

---

## 9. Challenges and Solutions

| Challenge | Solution |
|---|---|
| **Scale limitations:** the full mail-estate graph is too large to reprocess on every update. | **Neighbor-sampled batches:** training and inference run on neighbor-sampled mini-batches. |
| **Scoring unseen infrastructure:** new servers/certificates lack prior graph history. | **Inductive design:** GraphSAGE generalizes scoring to unseen nodes without full retraining. |
| **Integrity verification:** model/rule updates must be authentic and untampered. | **Cryptographic signing:** every package is verified against signed artifacts before load. |
| **Explainability gap:** black-box risk scores are hard to justify to analysts. | **Local SHAP + LLM rationale:** per-session feature attribution computed and explained locally. |
| **Encrypted handshake metadata:** TLS 1.3 hides more handshake fields than 1.2. | **Pre-encryption feature reliance:** ClientHello, JA3 fingerprint and certificate data stay visible pre-encryption. |

---

## 10. Impact and Benefits

### Core Benefits
- **Multi-layer correlation:** fuses protocol-layer session data with cryptographic-layer handshake/certificate data into a unified graph, removing manual packet-by-packet inspection.
- **Infrastructure-level visibility:** surfaces shared misconfiguration across servers and certificates instead of treating sessions as independent.
- **Explainable, not a black box:** each finding comes with triggering evidence and an LLM-articulated rationale grounded in SHAP values.
- **Multiple signals fused:** rule-based detection, learned anomaly scoring and graph-structural signal, so no single fragile method is relied on.

### Key Impacts
- **Faster lead generation:** converts bulk PCAP captures into a ranked, prioritized worklist, reducing manual triage.
- **Targets NTRO's mandate:** oriented toward TLS downgrade attempts, MITM interception and cryptographic non-compliance on email infrastructure.
- **Investigation- and evidence-ready output:** a defensible, attributable starting point for further analysis rather than an unexplained alert.
- **Closes a known blind spot:** continuous visibility into email cryptographic posture, which existing packet-decoding tools address inadequately.

> Estimated reduction in analyst triage time: **60–75%** on parsing, extraction and first-draft explanation. Final judgment stays human.

---

## 11. Comparison: Existing vs Proposed Solutions

| Existing approach | SecureMailScope |
|---|---|
| Decodes protocols; does not score cryptographic risk | AI-scored cryptographic risk per session |
| Static bad-cipher / bad-version blacklists | Rules + unsupervised anomaly detection for unseen patterns |
| Cannot score unseen servers without manual review | Inductive embeddings score new entities instantly |
| Manual, sample-based analyst review | Automated, exportable JSON/PDF/HTML reports |
| Assumes uniform visibility across sessions | Tags visibility (full / partial / resumed) per session |

---

## 12. Dashboard

The React Investigator Dashboard presents:

- Ranked, explained risk findings (risk score, severity, protocol, TLS version, cipher, certificate status, visibility tag, confidence tier)
- Session detail with a STARTTLS timeline, handshake and certificate panels, SHAP feature attribution, and a plain-language rationale
- An interactive session/certificate **link-analysis graph** highlighting shared-misconfiguration clusters
- Downgrade and MITM pattern views
- Exportable JSON/PDF/HTML audit reports and an audit trail
- Dark and light modes

---

## 13. Research and References

- [Neither Snow Nor Rain Nor MITM: An Empirical Analysis of Email Delivery Security](https://dl.acm.org/doi/pdf/10.1145/2815675.2815695)
- [IETF RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/info/rfc8446)
- [IETF RFC 3207: SMTP Service Extension for Secure SMTP over TLS](https://www.rfc-editor.org/info/rfc3207/)
- [IETF RFC 8996: Deprecating TLS 1.0 and TLS 1.1](https://www.rfc-editor.org/info/rfc8996)
- [TLS Fingerprinting with JA3 and JA3S](https://engineering.salesforce.com/tls-fingerprinting-with-ja3-and-ja3s-247362855967/)
- [Evaluation of Parsing Behavior using Real-world Out-in-the-wild X.509 Certificates](https://arxiv.org/pdf/2405.18993v1)
- [ChequeMark: An Ensemble Machine Learning Framework for After-Hours Business Deposit Fraud Detection](https://share.google/AgwrEzUBQFnKvxU4b)

---

## 14. Team

**DATA_HEX_GLITCHERS** · Team ID 158434 · Smart India Hackathon 2026

Repository: https://github.com/NIKK-666/Hex_Glitchers-SecureMail-
