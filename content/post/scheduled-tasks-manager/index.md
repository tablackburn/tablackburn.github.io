---
title: "Announcing ScheduledTasksManager: A New PowerShell Module for Cluster-Aware Task Management"
date: 2025-08-06
draft: false
slug: "announcing-scheduledtasksmanager-powershell-module"
summary: "Introducing ScheduledTasksManager v0.1.0, a new PowerShell module that simplifies managing scheduled tasks in Windows Server Failover Clusters with comprehensive cluster-aware functionality."
tags: ["PowerShell", "ScheduledTasks", "Clustering", "Windows Server", "Automation"]
categories: ["PowerShell", "Windows Server", "DevOps"]
params:
  author1: "Trent Blackburn"
  microsoft_alias: "trblackburn"
  featured_image: ""
  ai_note: "AI was used to help structure and format this blog post announcement"
---

I'm excited to announce the release of my first PowerShell module:
**ScheduledTasksManager v0.1.0**! This module is now available on the
[PowerShell Gallery](https://www.powershellgallery.com/packages/ScheduledTasksManager/)
and addresses a real challenge I've encountered in enterprise environments.

## The Problem

Managing scheduled tasks in Windows Server Failover Clusters can be complex and
error-prone. The built-in `ScheduledTasks` module from Microsoft works great for
standalone systems, but when you're dealing with clustered environments, you need
additional functionality to handle tasks across multiple nodes effectively.

## The Solution

ScheduledTasksManager extends the capabilities of Microsoft's built-in module with
cluster-aware functionality. It provides comprehensive functions for managing
scheduled tasks in both standalone and clustered Windows environments.

### Key Features

- **Cluster-Aware Operations**: Register, enable, disable, start, and monitor
  scheduled tasks across failover cluster nodes
- **Advanced Monitoring**: Retrieve detailed task information, run history, and
  cluster node details
- **Configuration Management**: Export and import task configurations for backup
  and deployment scenarios
- **Smart Filtering**: Filter tasks by state, type, and ownership across cluster nodes
- **Secure Authentication**: Built-in support for credential management and CIM sessions

## Quick Start

Getting started is simple:

```powershell
# Install from PowerShell Gallery
Install-Module -Name ScheduledTasksManager -Repository PSGallery

# Import the module
Import-Module ScheduledTasksManager

# Get all clustered scheduled tasks
Get-StmClusteredScheduledTask -Cluster "MyCluster"

# Register a new clustered task
Register-StmClusteredScheduledTask -Cluster "MyCluster" -TaskName "BackupTask" -TaskType "ClusterWide"
```

## What's Included

The module includes 11 comprehensive functions:

- Task retrieval and information gathering
- Task registration and removal
- Task state management (enable/disable)
- Manual task execution and monitoring
- Configuration export capabilities

Each function includes detailed help documentation and practical examples.

## Enterprise Ready

ScheduledTasksManager is designed for enterprise environments with features like:

- Credential delegation support
- Comprehensive error handling
- Standardized PowerShell function patterns
- Built-in validation for cluster operations

## Looking Forward

This initial release (v0.1.0) establishes the foundation for cluster-aware scheduled
task management. I'm already planning additional features and improvements based on
community feedback.

## Get Involved

- **Download**: Available now on [PowerShell Gallery](https://www.powershellgallery.com/packages/ScheduledTasksManager/)
- **Documentation**: Complete documentation at [tablackburn.github.io/ScheduledTasksManager](https://tablackburn.github.io/ScheduledTasksManager/)
- **Source Code**: View the source and contribute on [GitHub](https://github.com/tablackburn/ScheduledTasksManager)
- **Issues & Feedback**: Report bugs or request features through [GitHub Issues](https://github.com/tablackburn/ScheduledTasksManager/issues)

I hope this module helps simplify your scheduled task management in clustered
environments. Give it a try and let me know what you think!
