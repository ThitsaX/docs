---
title: Home
---

# 📘 ThitsaWorks Platform Documentation

Welcome to the technical documentation for **platform security and architecture** within the **Mojaloop ecosystem**.

This documentation describes how the platform is **actually implemented, secured, deployed, and operated** in real-world environments.

It focuses on:

- Practical architecture
- Production security controls
- Deployment models
- Operational behavior
- Integration patterns

This is **implementation-driven documentation**, not theoretical reference material.

---

## 🚀 Getting Started

Choose the deployment model that matches your environment:

- **[Deploy Mojaloop Locally (MicroK8s)](deploy-mojaloop-local-microk8s)**
- **[Deploy Mojaloop Locally (Without Kubernetes)](deploy-mojaloop-local-withoutk8s)**
- **[Deploy Mojaloop Payment Manager (On-Premise)](deploy-payment-manager-on-premise)**
- **[PM4ML Deployment Guide (Helm Installation)](deploy-payment-manager-helm)**
---

## 🏗️ Architecture

- **[Platform Architecture Overview](platform-architecture-overview)**  
  Provides a high-level reference architecture covering:
  - Mojaloop Hub, PM4ML, and Tazama
  - Cloud, on-premise, and hybrid deployments
  - Primary–Standby topology
  - Platform responsibilities and trust boundaries

---

## 🔐 Security Architecture

- **[Hub ↔ PM4ML Security Architecture](hub-pm4ml-security-architecture)**  
  Describes the end-to-end security model between the Hub and PM4ML, including:
  - Mutual TLS (mTLS) for transport-level security  
  - JWS for application-level message signing  
  - Certificate lifecycle and rotation  
  - Trust boundaries and ownership  

---

## ⚙️ Operations & Reliability

The following sections are being developed and will reflect validated production behavior:

- 🔁 Certificate & Key Management  
- 🚨 Incident & Failure Scenarios  
- 📐 Environment & Capacity Model  
- 📊 Monitoring & Alerting Architecture  
- 🧾 Operational Runbooks  

---

## 🌐 More Information

For general information about **ThitsaWorks**, including services and platform offerings:

👉 https://thitsaworks.com

---

## 📌 About This Site

This documentation reflects **real-world implementation and operational security practices** used in Mojaloop-based deployments.

It is intended for:

- Platform Engineers  
- Security Engineers  
- Infrastructure Engineers  
- DFSP Integration Teams  
- Operations Teams  

The goal is to provide a **clear, practical, and production-aligned understanding** of how the system is secured, deployed, and operated.