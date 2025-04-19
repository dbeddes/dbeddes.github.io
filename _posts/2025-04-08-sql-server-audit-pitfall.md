---
layout: post
title: "SQL Server Audit: Overview and Potential Issue"
date: 2025-04-19
categories: 
  - Database Security
  - SQL Server
  - Performance Optimization
  - Lessons Learned
  - Technical Tips
tags:
  - SQL Server Audit
  - Database Security
  - Compliance
  - CIFS Storage
  - Database Administration
  - Audit Specifications
description: "Learn about SQL Server Audit's core functionality and discover a critical performance pitfall when storing audit logs on CIFS shares."
---

# SQL Server Audit: Overview and Potential Issue

## Introduction

SQL Server Audit is a built-in system that captures information about access and changes to your SQL Server environment. Built on the Extended Events system, it efficiently captures events as they happen and stores them for later review.

The purpose of this post isn't to cover all the options for configuring SQL Audit, nor is my goal to tell you exactly how to configure SQL Audit. Instead, I aim to provide a quick overview of what it can do and share one significant issue I've encountered in a real-world implementation.

Most organizations implement auditing to meet compliance requirements, which often mandate logging who accessed tables and when. This data can drive alerts or help determine what information may have been compromised during a security breach. It can also help identify how intruders infiltrated the system and what configuration settings were modified.

## SQL Server Audit Components

SQL Server Audit consists of two main parts:

### 1. The Audit

This defines where to store audit logs and what action SQL Server should take if it cannot persist audit information. When configuring the audit, you have three storage options:

- **Windows Event Log**
- **Windows Security Log**
- **File storage**

For Event Log or Security Log storage, ensure the SQL Server service account has proper write permissions ([Microsoft documentation](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/write-sql-server-audit-events-to-the-security-log?view=sql-server-ver16)). Similarly, when writing to a file on a local drive or CIFS share, ensure the service account has write permissions to that location. Since these are audit logs, you should lock down security for the destination folder.

Depending on your organization's requirements, you can specify what SQL Server should do if it cannot write audit logs:
- Continue running
- Stop the service (ensuring all activity is logged, no matter what)

### 2. The Audit Specification

There are two types of Audit Specifications:

- **Server Audit Specification**: Captures server-level events such as login changes, server configuration changes, and failed login attempts.
- **Database Audit Specification**: Captures database-level events such as table reads/updates, role changes, and configuration modifications. This must be configured separately for each database.

After defining your requirements, you assign an Audit Specification to an Audit, then enable the audit to begin capturing information.

## Performance Considerations

While SQL Server Audit does impact performance, the impact is generally minimal depending on how much information you're capturing.

## Real-World Experience

In one implementation, we made several key decisions:
- We couldn't tolerate downtime, so we configured SQL Server to continue running if it couldn't write logs
- We needed to capture all queries on specific databases
- To reduce log volume, we added filters to exclude routine operations like index rebuilds and statistics updates

Despite filtering, the audit generated substantial data. We decided to write logs to a CIFS folder on another server with sufficient storage capacity.

This worked well initially, but occasionally the SQL Server would become unresponsive. Investigation revealed that when the CIFS server was extremely busy and slow to process requests, SQL Server would wait for audit log write operations to complete. This dramatically slowed performance to the point where new connections were rejected and existing queries stalled.

The solution required shutting down SQL Server, starting it in single-user mode to disable the audit, and then bringing it back online.

## Recommendation

Although you can write audit logs to a CIFS folder, I do not recommend this approach. SQL Server audit performance depends on the file server responding promptly, and slow CIFS responses can cause SQL engine issues.

Even with the audit configured to continue running if it can't write logs, slow CIFS connections can still cause outages. A better approach is to write logs to a local drive and then use a scheduled PowerShell job to copy those logs to another server for permanent storage.

## References

- [SQL Server Audit (Database Engine)](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine?view=sql-server-ver16)
- [Create a Server Audit and Server Audit Specification](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/create-a-server-audit-and-server-audit-specification?view=sql-server-ver16)
- [SQL Server Auditing - Server & Database Audit Specifications](https://www.sqlshack.com/sql-server-auditing-server-database-audit-specifications/)
