---
title: "SCCM Post-Installation Configuration"
date: 2025-12-16
categories:
  - Infrastructure
tags:
  - SCCM
  - Active Directory
  - Endpoint Management
  - Device Management
  - Group Policy
  - Firewall
  - Windows
---

This document covers post-installation setup and configuration to make SCCM fully functional. It follows my previous project where I performed end-to-end SCCM server deployment: [Access Here](https://alifarhanfayyaz.github.io/infrastructure/End-to-End-SCCM-Server-Deployment-and-Configuration/)

I have divided the configuration tasks in 6 manageable parts:

1. Install and Configure Site System Roles  
2. Configure Boundaries and Boundary groups  
3. Configure Discovery  
4. Configure Client Push  
5. Deploy Custom Client Settings (Lab Optimization)  
6. Validation Tests

## Infrastructure Components

| Virtual Component | Description | IP Address |
| :---- | :---- | :---- |
| Address space | Address space used in this lab | 172.16.171.0/24 |
| FA-DC01 | Domain Controller | 172.16.171.5 |
| FA-SCCM | SCCM Site Server \+ SQL Server | 172.16.171.10 |
| FA-Client01 | Windows 11 Client | 172.16.171.15 |

<br>

# 1\. Install and Configure Site System Roles

## Configure Report Server

I configured the Reporting Services Point for this lab; reporting services point integrates with SQL report server to deploy configuration manager reports.

Configured SQL Report Server first.

Launch Report Server Configuration Manager and Configure Web Service URL

![](/assets/4/remote_server_0.png)

Change Database → Create a new report server database

![](/assets/4/remote_server_1.png)

![](/assets/4/remote_server_2.png)

![](/assets/4/remote_server_3.png)![](/assets/4/remote_server_4.png)

![](/assets/4/remote_server_5.png)

## Install SCCM Site System Roles

SCCM (Microsoft Configuration Manager) Site System Roles are specific server functions that provide management capabilities. I will configure following roles for this lab:

* **Reporting Services Point**: Integrates with SQL Reporting Services for monitoring and reporting.  
* **Endpoint Protection Point:** Enables centralized management of Microsoft Defender Antivirus for client devices.  
* **Fallback Status Point:** Receives error codes from clients during failed installations.  
* **Software Update Point (SUP):** Manages Windows updates for devices.

Launch Microsoft Configuration Manager  
**SCCM Console → Administration → Site Configuration → Servers and Site System Roles**

![](/assets/4/sitesystemrole_1.png)

Select required roles.

![](/assets/4/sitesystemrole_2.png)

WSUS is configured to use ports **8530** and **8531**.

![](/assets/4/sitesystemrole_3.png)

Enabled synchronization on a schedule: Daily at 12:00AM

![](/assets/4/sitesystemrole_4png.png)

Set Supersedence behavior to immediate for the lab environment.

![](/assets/4/sitesystemrole_5png.png)

Selecting the option to download full files for all approved updates however, for enterprise environment express installation files can provide faster installation on computers because only the necessary files are downloaded.

![](/assets/4/sitesystemrole_6png.png)

Verify connectivity for the site database server, select **SCCM\_ SQLService** (service account) as reporting services point account.

![](/assets/4/sitesystemrole_7.png)  

Verify installation.

![](/assets/4/sitesystemrole_8.png)  

Review **sitecomp.log** 

![](/assets/4/sitesystemrole_9.png)  

<br>

# 2\. Configure Boundaries and Boundary Groups

A boundary defines where a client is on the network, and a boundary group tells SCCM which site and content sources that client should use.

## Create Boundary

I am creating a boundary for my lab’s IP address range.

![](/assets/4/boundary_1.png)

![](/assets/4/boundary_2.png)

## Create Boundary Group for Content Location

Configured a boundary group to point to the distribution point (FA-SCCM).

![](/assets/4/boundary_3.png)

![](/assets/4/boundary_4.png)

## Create Boundary Group for Site Assignment

Configured a boundary group for automatic site assignment.

![](/assets/4/boundary_5.png)

Check the option to use this boundary group for site assignment.

![](/assets/4/boundary_6.png)

<br>

# 3\. Configure Discovery

SCCM Discovery is the process where SCCM finds computers, users, and network resources in your organization, populating its database with this information.

Navigate to Discovery Methods → Active Directory System Discovery

![](/assets/4/discovery_method.png)

Enable Active Directory System Discovery

![](/assets/4/discovery_method_2.png)

Select **Workstations** OU

![](/assets/4/discovery_method_3.png) 
![](/assets/4/discovery_method_4.png)
![](/assets/4/discovery_method_5.png)

 Set polling schedule to 1 day just for the lab environment 

![](/assets/4/discovery_method_6.png)

<br>

# 4\. Configure Client Push

SCCM Client Push is a deployment method where the SCCM site server remotely initiates the installation of the Configuration Manager client agent onto Windows computers.

## Configure Client Installation Settings

Navigate to: Administration → Sites → Primary Site \-\> Client Installation Settings

![](/assets/4/client_installation_1.png)

Select **SCCM\_ClientPush** (service account) as a client push installation account.  
![](/assets/4/client_installation_2.png)  
![](/assets/4/client_installation_3.png)

## Configure Group Policy for Client Push

Client push installation account must be a local administrator on all target clients. I will enforce this using a group policy in this lab.

Navigate to the Group Policy Management Editor in the domain controller (FA-DC01) and create a new GPO linked to the **Workstations** OU called **SCCM-Settings**.

Edit **SCCM Settings** \-\> Create a new local group

![](/assets/4/client_install_gpo_1.png)

**Configuration Values:**  
**Action:** Update  
**Group Name:**  Administrators (built-in)  
**Add Account:** SCCM\_ClientPush

![](/assets/4/client_install_gpo_2.png)

Enforce GPO.

![](/assets/4/client_install_gpo_3.png)

Verify that **SCCM\_ClientPush** is added in the local Administrators account on a client machine (FA-Client01).

![](/assets/4/client_install_gpo_4.png)

## Configure Firewall Rules for Client Push

To allow SCCM to remotely install and manage clients, firewall exceptions are required on all clients.

Edit **SCCM-Settings** GPO

Navigate to: Computer Configuration \-\> Policies \-\>Windows Settings \-\> Security Settings \-\> Windows Firewall and Advanced Security

Create inbound firewall rules:

Predefined rule: Windows Management Instrumentation (WMI)

![](/assets/4/firewall_1.png)

Create inbound firewall rules:

Predefined rule: File and Printer Sharing

![](/assets/4/firewall_2.png)
![](/assets/4/firewall_3.png)

<br>

# 5\. Deploy Custom Client Settings (optional)

I will configure custom client settings so it’s automatically applied to new clients for consistent client behavior overriding default client settings.

Navigate to **Client Settings** \-\> Create a new **Custom Client Device Settings**

![](/assets/4/custom_client_settings.png)

![](/assets/4/custom_client_settings_2.png)

Enabled setting to Manage Endpoint Protection client on client computers.

![](/assets/4/custom_client_settings_3.png)
![](/assets/4/custom_client_settings_4.png)

Specified company name and created a custom logo.

![](/assets/4/custom_client_settings_5.png)

Deployed Custom Client Settings to All Systems.

![](/assets/4/custom_client_settings_6.png)

<br>

# 6\. Verify Client Installation

Navigate to: Assets and Compliance \-\> Right Click **FA-Client01** \-\> Install Client

![](/assets/4/final_client_install_1.png)

Keep the defaults while installing configuration manager client.

![](/assets/4/final_client_install_2.png)

Review **ccm.log** file for client installation logs or review the client push report in configuration manager for detailed status.

![](/assets/4/final_client_install_3.png)

![](/assets/4/final_client_install_4.png)

<br>

# Related Microsoft Documentation

1. **Client Push Installation Account:** [https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/accounts\#client-push-installation-account](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/accounts#client-push-installation-account)  
2. **Network Access Account:** [https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/accounts\#client-push-installation-account](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/accounts#client-push-installation-account)  
3. **Ports Used During Configuration Manager Client Deployment:** [https://learn.microsoft.com/en-us/intune/configmgr/core/clients/deploy/windows-firewall-and-port-settings-for-clients](https://learn.microsoft.com/en-us/intune/configmgr/core/clients/deploy/windows-firewall-and-port-settings-for-clients)  
4. **Learn how clients find site resources and services for System Center Configuration Manager:** [https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/understand-how-clients-find-site-resources-and-services](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/understand-how-clients-find-site-resources-and-services)  
5. **Define site boundaries and boundary groups for System Center Configuration Manager:** [https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/configure/define-site-boundaries-and-boundary-groups](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/configure/define-site-boundaries-and-boundary-groups)  
6. **About discovery methods for System Center Configuration Manager:** [https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/configure/about-discovery-methods](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/configure/about-discovery-methods)  
7. **Reporting Services Point Account:** [https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/accounts\#client-push-installation-account](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/accounts#client-push-installation-account)  
8. **Client installation methods in System Center Configuration Manager:** [https://learn.microsoft.com/en-us/intune/configmgr/core/clients/deploy/plan/client-installation-methods](https://learn.microsoft.com/en-us/intune/configmgr/core/clients/deploy/plan/client-installation-methods)  
9. **How To Install Clients With Client Push:** [https://learn.microsoft.com/en-us/intune/configmgr/core/clients/deploy/deploy-clients-to-windows-computers\#BKMK\_ClientPush](https://learn.microsoft.com/en-us/intune/configmgr/core/clients/deploy/deploy-clients-to-windows-computers#BKMK_ClientPush)

<br>

# Other References

Shout out to my guy **Justin Chalfant** at Patch My PC, couldn’t have done it without his detailed guide: [https://youtu.be/amrg\_mlFvuk?si=KSHFS0FwOEqUkIzP](https://youtu.be/amrg_mlFvuk?si=KSHFS0FwOEqUkIzP)