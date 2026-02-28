# Threat Model Report: Online Payment Processing Application


## Task 1: System Definition and Architecture

### 1.1 System Overview

The system is an online payment processing platform that enables customers to make payments to merchants through a website. There is an admin portal which allows internal staff to manage the platform, onboard merchants, and monitor transactions.

### 1.2 Application Components

| # | Component | Description |
|---|-----------|-------------|
| 1 | **Web Frontend** | A single page application served over internet for customer-facing, built with Next.js. Provides login, account management, payment initiation, and transaction history. |
| 2 | **API Backend** | Exposes RESTful APIs consumed by the web frontend and admin portal. Handles authentication, authorization, payments, and data validation. |
| 3 | **Admin Portal** | Internal web application for platform managers. Used for merchant onboarding, user management, transaction monitoring and system configuration. |
| 4 | **User Database** | Stores customer profiles, credentials (hashed), addresses and account preferences. |
| 5 | **Merchant Database** | Stores merchant profiles, bank settlement details, API keys, fee structures, and onboarding records. |
| 6 | **Core Banking System** | External banking integration for fund transfers and settlement |
| 7 | **Notification Service** | Sends transactional emails and SMS (OTPs, payment receipts, alerts) via third-party services like firebase. |

### 1.3 Users and Roles

| Role | Description | Access Level |
|------|-------------|--------------|
| **Customer** | End user who registers, links payment methods, and initiates payments to merchants. | Public facing frontend. Personal account data and transactions only. |
| **Merchant** | Business entity that receives payments. Accesses merchant dashboard for transaction reports and  information. | Merchant scoped API access. Own merchant data only. |
| **Platform Admin** | Internal staff responsible for system configuration and merchant onboarding/offboarding. | Admin portal. Have read/write access to all platform data. |
| **Support Agent** | Internal staff handling customer queries and refund requests. | Admin portal. Have read access to customer/transaction data, only limited write (initiate refunds). |

### 1.4 Data Types Handled

| Data Category | Examples | Sensitivity |
|---------------|----------|-------------|
| **Authentication Credentials** | Passwords (hashed + salted), session tokens, OAuth tokens | Critical |
| **Payment Card Data** | Card numbers, expiry, CVV  | Critical  |
| **Personally Information** | Full name, email, phone, address, national ID | High |
| **Transaction Data** | Transaction amounts, timestamps, statuses, entries | High |
| **Merchant Business Data** | Merchant profiles, bank account details for settlement, fee structures, API keys | High |
| **API Keys & Secrets** | Payment gateway credentials, banking API keys, internal service tokens | Critical |
| **Logs** | Login events, payment events, admin actions, API call logs | High |

### 1.5 External Dependencies

| Dependency | Purpose | Trust Level |
|------------|---------|-------------|
| **Payment Gateway (Visa/Mastercard networks)** | Card authorization | Third-party |
| **Email / SMS Provider** | Transactional notifications, OTP sending | Third-party (Firebase), moderate trust |
| **Identity Provider (OAuth)** | Optional social login for customers | Third-party, moderate trust |
| **Deployment** | For website hosting and deployment | Third-party (Vercel or AWS) ,moderate trust |

### 1.6 Trust Boundaries

| # | Boundary | From → To | Description |
|---|----------|-----------|-------------|
| TB-1 | **Internet → Web Frontend** | Public internet → Frontend | Untrusted users reach the frontend, all input is untrusted. |
| TB-2 | **Web Frontend → API Backend** | Frontend → Application Tier | Frontend passes user requests to the backend, requires authentication tokens. |
| TB-3 | **API Backend → Data Stores** | Application Tier → Data Tier | Backend queries databases, requires DB credentials and input sanitization. |
| TB-4 | **API Backend → Payment Gateway** | Internal Network → External Third-Party | Outbound calls carrying sensitive card data, requires API credentials. |

---

## Task 2: Asset Identification and Security Objectives

### 2.1 Asset Inventory

| # | Asset | Description | Primary Security Objective |
|---|-------|-------------|---------------------------|
| A1 | **User Credentials** | Hashed passwords, seeds, session tokens | Confidentiality |
| A2 | **Payment Card Data** | PAN, expiry  | Confidentiality |
| A3 | **Customer** | Name, email, phone, address, KYC documents | Confidentiality |
| A4 | **Transaction Records** | Payment amounts, timestamps, statuses, reference IDs | Integrity |
| A5 | **Merchant Data** | Profiles, settlement bank details, fee configs | Confidentiality |
| A6 | **API Keys & Secrets** | Gateway credentials, banking API keys, service tokens | Confidentiality |
| A7 | **Transaction Logs** | Login events, payment events, admin actions | Integrity |
| A8 | **Core Banking Credentials** | API keys and certificates for banking integration | Confidentiality |

---

## Task 3: Threat Modeling

| ID | STRIDE Category | Threat | Affected Component | Impact | Risk Level | Risk Reasoning |
|----|----------------|--------|-------------------|--------|:---:|----------------|
| T1 | Spoofing | Brute force attack on customer or merchant login endpoints. | API Backend (Auth module) | Unauthorized account access → financial loss, data breach. | **High** | For internet facing login endpoint, large cale automated attacks are common. |
| T2 | Spoofing | Session hijacking via stolen or leaked session/JWT tokens (XSS). | Web Frontend, API Backend | Attacker impersonates legitimate user, can initiate payments or exfiltrate data. | **High** | Tokens are bearer credentials. If stolen, full account takeover is possible until expiry. |
| T3 | Spoofing | Phishing attacks targeting customers to steal credentials or card data via fake login pages. | External (end-user device) | Credential theft, fraudulent transactions. | **Medium** | Outside direct system control, but platform reputation and user funds are at risk. |
| T4 | Unauthorization |  Regular user gains admin level access through broken role checks or browsing to admin endpoints. | API Backend, Admin Portal | Full platform compromise, attacker can modify merchant configs, view all data. | **High** | If role enforcement is bypassed. |
| T5 | Information Disclosure | SQL injection, attacker extracts database contents through unsanitized input in API queries. | API Backend, User DB, Merchant DB | Mass data breach of PII, credentials, financial data. | **High** | Single successful injection can dump entire database. |
| T6 | Information Disclosure | Database backup exposure, backups stored in insecure locations or without encryption. | Backup Storage | Full data breach from leaked backups. | **Medium** | Backups are often overlooked. If compromised, impact equals a full DB breach. |
| T7 | Information Disclosure | Attacker intercepts API traffic between frontend and backend, or between backend and external systems. | All network paths | Credential theft, card data leakage,. | **Medium** | Low verification measures. |
| T8 | Tampering | API parameter tampering, attacker modifies payment amounts, merchant IDs, or other parameters in API requests. | API Backend | Financial fraud, paying less than owed. | **High** | Client side values must never be trusted, server side validation is essential. |
| T9 | Denial of Service | API rate limit / DDoS, attacker floods API endpoints with requests to degrade or deny service. | API Backend, Web Frontend | Payment processing downtime, revenue loss, customer dissatisfaction. | **Medium** | Internet facing system is always a DDoS target. |
| T10 | Information Disclosure | Sensitive data leaking into logs. PII, card numbers, or secrets accidentally written to log files. | Logging Stack | Data breach through log access. | **Medium** | Common developer mistake, log files often have broader access than databases. |
| T11 | Spoofing | Compromised admin credentials, admin account taken over via phishing, credential reuse, or weak password. | Admin Portal, API Backend | Full platform compromise, read/write access to all data and configurations. | **High** | Admin accounts are high value targets. |

---

## Task 4: Secure Architecture Design


### 4.1 Access Management

| Control | Description | Justification |
|---------|-------------|---------------|
| **OAuth 2.0** | All authentication flows use OAuth 2.0. | Standardized protocol; eliminates custom auth vulnerabilities. Addresses T1, T2. |
| **Multi-Factor Authentication (MFA)** | MFA mandatory for admin and merchant accounts; for customers on high value transactions. | Prevents credential only attacks (T1, T11). Even if passwords are compromised, MFA blocks account takeover. Reduces impact of T3 (phishing). |
| **Role-Based Access Control** | Strict role definitions (Customer, Merchant, Support Agent, Admin) enforced at the API gateway and backend. | Directly prevents T4 (unauthorized admin access via broken role checks). Each role has minimum necessary permissions. |

### 4.2 Network Segmentation

| Control | Description | Justification |
|---------|-------------|---------------|
| **API Gateway as Single Entry Point** | All external traffic routes through an API gateway that handles rate limiting, authentication, input validation, and routing. | Centralizes security enforcement, no direct backend access possible. Primary defence against T8 (API parameter tampering) and T9 (DDoS). |

### 4.3 Data Protection

| Control | Description | Justification |
|---------|-------------|---------------|
| **Field-Level Encryption** | Critical fields (national ID, KYC docs) are encrypted at the application layer before database storage. | Adds a second layer of protection against T5 (SQL injection). |

### 4.4 Secrets Management

| Control | Description | Justification |
|---------|-------------|---------------|
| **Centralized Secrets Vault** | All secrets (DB credentials, API keys, encryption keys) stored in a dedicated vault (e.g., .env files) with access policies. | Eliminates hardcoded secrets in code or config files (assets A6, A9). |
| **No Secrets in Code or Logs** | CI/CD pipelines inject secrets at deploy time. | Prevents accidental credential exposure through source code or T10 (sensitive data leaking into logs). |

### 4.5 Monitoring and Logging

| Control | Description | Justification |
|---------|-------------|---------------|
| **Log Sanitization** | Automated filters remove card numbers, and secrets from log entries before storage. | Directly fixes T10 (sensitive data leaking into logs). |

---

## Task 5: Risk Treatment and Residual Risk


| Threat ID | Threat | Risk Level | Treatment | Treatment Details |
|-----------|--------|:---:|-----------|-------------------|
| T1 | Brute force | High | **Mitigate** | Rate limiting at API gateway, account lockout after repeated failures, CAPTCHA on suspicious activity. |
| T2 | Session hijacking via stolen JWT tokens | High | **Mitigate** | Short lived tokens (15 min), refresh rotation. |
| T3 | Phishing attacks targeting customer credentials | Medium | **Mitigate + Accept** | MFA reduces impact even if credentials are stolen. Educating user. |
| T5 | SQL injection — mass data exfiltration from databases | High | **Mitigate** | Parameterized queries. |
| T7 | Interception of API traffic | Medium | **Mitigate** | TLS on all channels. |
| T8 | API parameter tampering — modified payment amounts or merchant IDs | High | **Mitigate** | Server-side validation of all inputs, signed payment requests, API gateway input schema enforcement. |
| T9 | DDoS / API rate abuse causing service downtime | Medium | **Mitigate + Transfer** | CDN-based DDoS absorption, API rate limiting at gateway, DDoS protection services. |
| T10 | Sensitive data leaking into log files | Medium | **Mitigate** | Automated log sanitization filters with PII and card number |
### 5.2 Residual Risk Explanation

- **Phishing:** No technical control can stop a user from entering credentials on a convincing fake site. MFA limits the damage even when credentials are stolen, but the risk cannot be fully eliminated and is accepted with ongoing user awareness efforts.

- **Insider Threat (T4, T11):** A legitimate admin with valid credentials can misuse access in ways that initially look normal. Behavioral monitoring and separation of duties reduce the likelihood and blast radius, but complete elimination would require removing human access entirely, which is not feasible.

- **Third-Party Dependency Risk:** Outages or breaches at the payment gateway or core banking system are outside the platform's control.

---

## Task 6: Final Report Summary

### System Overview

This report presents a complete threat model for an **Online Payment Processing Application**, an online platform that processes payments between customers and merchants via payment gateway and core banking integrations. The system consists of 9 major components spanning a web frontend, API backend, admin portal, two databases, and integrations with external payment and banking systems.

### Architecture Diagram



### Asset Inventory

10 critical assets identified, including credentials, PII, financial data, secrets, business logic, and system configuration. Each mapped to Confidentiality, Integrity, Availability, and Accountability objectives.

### Threat Model

11 threats identified across the required threat areas using the STRIDE methodology. 6 classified as **High** risk (T1, T2, T4, T5, T8, T11) and 5 as **Medium** risk (T3, T6, T7, T9, T10).

### Security Controls

20 architectural controls across 6 categories: Identity & Access Management, Network Segmentation, Data Protection, Secrets Management, Monitoring & Logging, and Secure Deployment. Each control is justified against specific threat IDs.

### Residual Risks

2 residual risks acknowledged: phishing and third party dependency risk.

### Assumptions and Limitations

| # | Assumption / Limitation |
|---|------------------------|
| 1 | The core banking system and payment gateway are treated as black boxes, their internal security is not considered in this model. |
| 2 | Physical security of data centers and endpoint security of user devices are out of scope. |
| 3 | The threat model covers the production environment. Development and staging environments are not modeled separately. |
