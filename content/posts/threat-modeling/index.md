+++
title = 'A Practical Guide to Threat Modeling'
date = 2025-08-20T13:03:51-04:00
draft = false
+++

## Introduction

As application security continues to be a critical concern in modern software
development, threat modeling has emerged as an essential proactive approach to
identifying and mitigating security vulnerabilities before they become
exploitable weaknesses. 

But what exactly is threat modeling, and how does it fit into the development
lifecycle? Threat modeling is a structured process for identifying potential
threats, vulnerabilities, and countermeasures early in the development
lifecycle. It’s about shifting from a reactive "patch-when-hacked" stance to a
proactive "secure-by-design" mindset. To put theory into practice, I embarked
on a personal project to apply threat modeling to a realistic application.
Multiple distinct threat modeling frameworks exist, each of which can be
applied to different scenarios.

## The STRIDE Framework

Among these, I chose Microsoft’s STRIDE framework because it is widely adopted,
easy to map to system elements, and well-suited for analyzing application-level
threats. STRIDE framework categorizes threats into six distinct categories:

- **S**poofing: Identity impersonation attacks
- **T**ampering: Unauthorized data modification
- **R**epudiation: Denial of actions performed
- **I**nformation Disclosure: Exposure of sensitive information
- **D**enial of Service: Making resources unavailable
- **E**levation of Privilege: Gaining unauthorized access levels

Now that we understand STRIDE's six categories, let's see how they apply to a
real-world scenario.

## Case Study: "SecureDoc" Document Management System

To illustrate how STRIDE can be applied in practice, let’s walk through a case
study of a hypothetical system: SecureDoc. This application allows
organizations to store, share, and collaborate on sensitive documents.

### Current Architecture

**Core Components:**
- **Frontend**: React SPA with JWT authentication
- **API Gateway**: Node.js/Express with rate limiting
- **Application Server**: Java Spring Boot microservices
- **PostgreSQL**: Stores structured metadata like user permissions, document ownership, and folder hierarchies.
- **MongoDB**: Stores the unstructured document content itself.
- **AWS S3**: Stores associated file blobs that are part of a document (e.g., uploaded images, PDFs). Files are encrypted.
- **Authentication Service**: OAuth 2.0 with LDAP integration
- **Audit Service**: Elasticsearch for compliance logging

With these components in place, the next step is to understand how they
interact. That’s where a *Data Flow Diagram (DFD)* is invaluable. A DFD helps
us visualize how data moves through the system, highlighting interactions and
boundaries where threats might arise.

### DFD Components for SecureDoc

Breaking it down, here are the main elements of SecureDoc’s DFD.

- **External Entities**: These are the actors that interact with our system but are outside of its control.
  - User via Browser: A person interacting with SecureDoc through their web browser to manage documents.
  - LDAP: External directory service for user authentication.

- **Processes**: These are the components within our system that handle or transform data.
  - API Gateway: Node.js/Express layer for routing, rate limiting, and request validation.
  - Authentication Service: OAuth 2.0 service that verifies users and integrates with LDAP.
  - Application Server: Java Spring Boot microservices for core business logic (e.g., document processing, collaboration features).
  - Audit Service: Handles logging for compliance.

- **Data Stores**: This is where data is held persistently.
  - PostgreSQL: Holds document metadata (e.g., titles, permissions, versions).
  - MongoDB: Stores document content (e.g., unstructured data like text or embeds).
  - AWS S3: Encrypted bucket for file blobs (e.g., PDFs, images).
  - Elasticsearch: Indexes and stores audit logs for search and compliance.

- **Trust Boundaries** (often represented as dotted lines): These are crucial for
  threat modeling. A trust boundary is a line we draw on our diagram to
separate zones with different security levels—for example, the boundary between
an untrusted user's browser and our trusted backend servers. We'll analyze
these in detail after the diagram.

![Data Flow Diagram](dfd.svg)

Trust Boundaries (red dotted line in the figure) in the SecureDoc DFD:

1. **Client-Server Boundary**: Separates the untrusted client-side (Frontend
React SPA) from the trusted server-side (API Gateway and beyond). Data
crossing: HTTP requests/responses, API calls with JWT. Risks: Untrusted inputs
from users.

2. **Internal-External Auth Boundary**: Between Authentication Service
(internal) and LDAP (external entity). Data crossing: LDAP queries/credentials.
Risks: Exposure to external directory services.

3. **Internal-Cloud Storage Boundary**: Between Application Server (internal)
and AWS S3 (external cloud provider). Data crossing: Encrypted file
uploads/downloads. Risks: Data leaving the organization's control to a
third-party service.

4. **Audit Logging Boundary**: Between Application Server/Audit Service (core
processes) and Elasticsearch (data store, potentially with separate access
controls). Data crossing: Event logs. Risks: Sensitive logs in a searchable
index.

### Applying STRIDE to SecureDoc Based on the DFD

Now that we’ve mapped out the system and its trust boundaries, we can begin
analyzing potential threats systematically using STRIDE. This is done by
asking: What could go wrong under each STRIDE category? Practical workflow: 

- For each node (processes/actors/services): check Tampering, Repudiation,
  Elevation of privilege, Spoofing (if a node can impersonate others), and
Information disclosure (if the node stores/processes sensitive data). Nodes are
where logic and privileges live.
- For each edge (data flows/communications): check Tampering, Information
  disclosure, and Replay‑like Spoofing (auth of messages). Consider
transport integrity/confidentiality and message authentication.
- For each trust boundary (between different privilege/ownership/trust zones):
  prioritize Spoofing, Elevation of privilege, and Tampering — boundaries are
where authentication, authorization, and input validation fail. Also treat them
as places to enforce controls (auth, ACLs, filtering, encryption).

Below is a table summarizing key threats, focused on practical development
scenarios. This isn't exhaustive but highlights common risks in a document
management system like SecureDoc.

| Element              | STRIDE Category | Potential Threat | Mitigation |
|----------------------|-----------------|------------------|------------|
| User via Browser (External Entity) | Repudiation | User denies performing an action (e.g., deleting a document). | Log all user actions via Audit Service to Elasticsearch with timestamps and IP details. |
| User via Browser (External Entity) | Tampering | Malicious script injection (e.g., XSS) alters UI or intercepts data. | Use React's built-in escaping; implement Content Security Policy (CSP); validate and sanitize all inputs server-side. |
| User via Browser (External Entity) | Information Disclosure | Sensitive data (e.g., JWT payload) exposed in browser storage. | Store JWTs securely (e.g., HttpOnly cookies if possible); avoid logging sensitive info. |
| API Gateway (Process) | Repudiation |Lack of logging/audit trails makes it impossible to prove actions taken via the API. | Implement append-only logging to the Audit Service for all state-changing API calls |
| API Gateway (Process) | Denial of Service | Flood of requests overwhelms the gateway, blocking access. | Already has rate limiting; add auto-scaling and Web Application Firewall (WAF). |
| API Gateway (Process) | Elevation of Privilege | Weak validation allows unauthorized API access. | Strict JWT validation; role-based access control (RBAC) enforced here. |
| API Gateway (Process) | Spoofing | Weak authentication allows attackers to impersonate legitimate API clients. | Strong API authentication (JWT validation) and proper CORS configuration. |
| API Gateway (Process) | Spoofing | Weak validation allows CSRF attacks. | Implement anti-CSRF protection, such as using the `SameSite=Strict` cookie attribute or synchronizer tokens. |
| Authentication Service (Process) | Spoofing | Attacker spoofs LDAP responses or intercepts OAuth flows. | Use TLS for all communications; validate LDAP certificates; monitor for anomalous auth attempts. |
| Application Server (Process) | Tampering | Input tampering alters document data during processing. | Input validation/sanitization in Spring Boot; use prepared statements for DB queries. |
| Authentication Service (Process) | Information Disclosure | Debug information or stack traces leaking sensitive data. | Disable debugging information. |
| Application Server (Process) | Denial of Service | Resource-intensive operations (e.g., large file processing, ReDoS exploiting inefficient regex patterns) exhaust server. | Implement timeouts and resource quotas in microservices; monitor with tools like Prometheus. |
| PostgreSQL (Data Store) | Information Disclosure | Unauthorized access to metadata exposes document details. | Use row-level security; encrypt sensitive fields; restrict access via VPC. |
| PostgreSQL (Data Store) | Tampering | SQL injection modifies metadata. | Use ORM (e.g., Hibernate in Spring) to prevent injections; regular backups. |
| MongoDB (Data Store) | Elevation of Privilege | Exploited vulnerability allows escalated DB access. | Role-based auth in MongoDB; least-privilege principles for app server connections. |
| AWS S3 (Data Store) | Tampering | Misconfigured bucket allows modification/deletion of files. | Apply least-privilege access policies (IAM roles) and use AWS Config rules to detect/remediate misconfigurations. |
| AWS S3 (Data Store) | Information Disclosure | Misconfigured bucket allows public access to encrypted files. | Server-side encryption (SSE); private buckets with signed URLs; regular audits. |
| AWS S3 (Data Store) | Information Disclosure | Overly permissive or long-lived pre-signed URLs grant excessive access to private documents. | Generate pre-signed URLs with the shortest possible expiration time and scoped to the specific user's permissions. |
| Audit Service (Process) | Repudiation | Logs tampered to hide actions. | Make logs append-only; use digital signatures for integrity. |
| Audit Service (Process) | Information Disclosure | Logs leak sensitive data (e.g., user PII). | Anonymize logs; access controls on Elasticsearch. |
| LDAP (External Entity) | Information Disclosure | LDAP queries or responses could be leaked if TLS is not enforced. | Secure LDAP (LDAPS) with TLS; mutual authentication. |
| LDAP (External Entity) | Tampering | LDAP queries or responses could be altered if TLS is not enforced. | Secure LDAP (LDAPS) with TLS; mutual authentication. |
| Elasticsearch (Data Store) | Information Disclosure | Insecure API exposes logs to unauthorized users. | API keys and RBAC in Elasticsearch; firewall rules. |

This STRIDE application focuses on the DFD elements, revealing hotspots like
authentication flows and data stores for sensitive documents. In practice,
prioritize threats by likelihood and impact, then iterate on mitigations during
development.

### Proposed New Features Analysis

To show how threat modeling can guide design decisions, let’s apply STRIDE to
some proposed features for SecureDoc.

#### Feature 1: External Image Profile Photos

**Proposed Implementation:**
A user can set their profile photo by providing a URL to an external image.
Instead of uploading files, the server fetches the image from the provided URL
and displays it as their profile picture.

```javascript
// Proposed endpoint
POST /api/profile/photo
{
  "imageUrl": "https://example.com/user-photo.jpg"
}
```

**STRIDE Analysis:**

| STRIDE Category | Potential Threat | Mitigation |
|----------------------|-----------------|----------------------------|
| Spoofing | Client-side request forgery (CSRF) that allows attacker to modify other user's photo |Implement anti-CSRF protection, such as using the `SameSite=Strict` cookie attribute or synchronizer tokens |
| Tampering | Server-side request forgery (SSRF) that allows attacker to access internal services | Whitelist allowed domains, block private IP ranges, use dedicated image proxy service|
| Tampering | Fetching an attacker-hosted PHP file | After fetching the content, validate that its `Content-Type` header and file magic bytes correspond to an allowed image format (e.g., JPEG, PNG) before storing or serving it|
| Repudiation | User denies performing such a request | Log user actions with timestamps and IP details |
| Information Disclosure | Server-side request forgery (SSRF) could expose internal resources | Whitelist allowed domains, block private IP ranges, use dedicated image proxy service|
| Denial of Service | Large image files could exhaust server resources | Set maximum file size limits, implement timeout controls, use asynchronous processing|

Recommendation to developer: Implement a secure image proxy service with domain whitelisting, file size limits, and content validation. Never directly fetch user-provided URLs from your application servers.

#### Feature 2: Document Auto-Save with Version History

**Proposed Implementation:**
As users type in a document editor, the system automatically saves drafts every 30 seconds and maintains a version history that users can browse and restore from.

```javascript
// Proposed implementation
setInterval(() => {
  fetch('/api/documents/' + docId + '/autosave', {
    method: 'POST',
    body: JSON.stringify({ content: editor.getContent() })
  });
}, 30000);
```

**STRIDE Analysis:**

| STRIDE Category | Potential Threat | Mitigation |
|----------------------|-----------------|----------------------------|
| Spoofing | Client-side request forgery (CSRF) that allows attacker to modify unauthorized documents | Implement anti-CSRF protection, such as using the `SameSite=Strict` cookie attribute or synchronizer tokens |
| Tampering | Race conditions between multiple users could corrupt document state |Consider concurrency control strategies like optimistic locking, version timestamps for collaborative edits. |
| Tampering | Vulnerable to Insecure Direct Object Reference (IDOR) attack due to lack of permission check | Check user's permission on particular `docID` |
| Information Disclosure | Version history could expose sensitive information that was later removed| Implement version history access controls, allow permanent deletion of specific versions, scan for sensitive data patterns|
| Denial of Service | Rapid automated changes to generate thousands of versions per document |Limit version creation frequency, implement storage quotas per user/document, automatically purge old versions |

Recommendation to developer: Implement proper concurrency controls with conflict resolution, add version history access controls that respect current document permissions, and include storage limits with automatic cleanup policies.

### My Key Takeaways

Having walked through both existing architecture and proposed features, here
are the key lessons I drew from the process.

1.  **Security is a Mindset, Not a Feature:** Threat modeling forces you to think like an attacker and view your application through a critical lens at every stage.
2.  **Proactive is Cheaper than Reactive:** Finding a design flaw on a whiteboard is infinitely cheaper and easier than fixing a vulnerability in production that has been exploited.
3.  **Structure is Your Friend:** Frameworks like STRIDE provide a systematic way to brainstorm threats, ensuring you don't miss entire categories of risk.
4. **Iterate on Threat Models as Systems Evolve:** Systems evolve continuously, and threat models must be updated regularly to remain effective. Schedule periodic reviews with major releases or architecture changes to identify new attack vectors that emerge as your system grows.
