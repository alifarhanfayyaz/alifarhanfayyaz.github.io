---
title: "Deploying Devices Using Windows Autopilot Device Preparation"
date: 2026-01-19
categories:
  - Infrastructure
tags:
  - Endpoint Management
  - Windows Autopilot
  - Microsoft Intune
  - Microsoft Entra ID
  - Device Provisioning
  - Zero-Touch Deployment
  - Modern Device Management
  - Conditional Access
  - Role-Based Access Control (RBAC)
  - Windows Autopilot Device Preparation
  - Intune Policies
---

Windows Autopilot device preparation automates the device provisioning and configuration process, hence streamlining the device onboarding process for IT teams. I have configured and documented the Windows Autopilot device preparation user-driven deployment, which can perform the following tasks during the deployment:

* Joins the device to Microsoft Entra ID.  
* Enrolls the device in Intune.  
* Installs up to 10 essential applications.  
* Runs up to 10 essential PowerShell scripts.

The amazing thing about Windows Autopilot device preparation is that it provides the ability to configure deployments from a single policy that bundles deployment settings and tracks essential policies, apps, and scripts throughout the provisioning process. In addition to that, both LOB and Win32 apps can be deployed simultaneously.

Once it is configured and deployed, that’s it \- there is nothing else for IT to do, as compared to the Windows Autopilot user-driven method where IT has to ensure that the device hash is uploaded to Intune, especially if the OEM is not handling the Windows Autopilot registration. The downside of Windows Autopilot device preparation, however, is that it only supports strictly Entra ID joined scenarios and is limited to Windows 11 only.

Microsoft has organized the requirements for Windows Autopilot device preparation into five categories. I will follow this structure to ensure all requirements are met.

1. Software  
2. Networking  
3. Licensing  
4. Configuration  
5. RBAC

<br>

# Requirements

## Software

For this lab, I am using a virtual machine running Windows 11 Pro (24H2), which meets the operating system requirements.

## Networking

I am using my home network for this lab so I basically just had to make sure that the device is able to communicate with Microsoft services without any restriction and outbound access to certain ports are allowed:

* Ensure DNS name resolution for internet DNS names.  
* Confirmed outbound access is allowed for all hosts via port 80 (HTTP), 443 (HTTPS), and 123 (UDP/NTP).

In corporate networks with restrictive internet access, you would need to ensure that devices are able to communicate directly and securely to Microsoft during the autopilot setup by making required changes in the firewall such as:

* Bypass SSL inspection for Microsoft endpoints  
* Allow outbound 443 without interception

I can skip this step for my lab but it’s important to note that Windows Autopilot device preparation relies on several different types of services to function properly so it’s important to ensure proper network configuration for these services: [Windows Autopilot device preparation requirements](https://learn.microsoft.com/en-us/autopilot/device-preparation/requirements?utm_source=chatgpt.com&tabs=networking)

## Licensing

I am using a combination of Microsoft Entra ID P1 and Microsoft Intune licenses, which satisfies the licensing requirements for the setup.

## Configuration

Now I will cover the required configuration to prepare the infrastructure for a Windows Autopilot device preparation deployment. I have divided this into six clear steps:

1. Configure MDM User Scope  
2. Allow users to join devices to Microsoft Entra ID  
3. Create an assigned device group  
4. Create a user group  
5. Assign applications and PowerShell scripts to device group  
6. Create an intune policy for remote desktop users (optional)

### Configure MDM User Scope

MDM user scope defines which users can automatically enroll their devices into Intune. Generally, you would want to limit MDM scope to corporate users to avoid auto enrollment for personal devices or contractors. I created a security group called **Corporate Users** for this purpose.

Navigate to Microsoft Entra admin center \-\> Mobility \-\> MDM user scope

![](/assets/6/mdm_scope_1.png)

Select “Some” and add the **Corporate Users** group to the MDM scope.

![](/assets/6/.png)

### Allow users to join devices to Microsoft Entra ID

We need to specify users that are allowed to join devices to Entra ID. In most organizations, this is limited to corporate users with company-owned devices. This helps prevent personal devices from being unintentionally enrolled in Intune in BYOD scenarios.

Navigate to Entra ID \-\> Devices \-\> Device Settings \-\> Microsoft Entra join and registration settings

![](/assets/6/device_join1.png)

Select **Corporate Users** group as a selected member that may join devices to Entra ID

![](/assets/6/device_join2.png)

### Create an assigned device group

For Windows Autopilot device preparation to work properly, it is important to create an assigned device group. Unlike Windows Autopilot, where a dynamic group is generally preferred to automate device assignment, Windows Autopilot device preparation automatically adds devices to this assigned group during deployment provided that the **Intune Provisioning Client** is set as the group owner.

Navigate to Entra ID \-\> Groups \-\> New Group

**Configuration Values:**  
Group Type: Security  
Group Name: Windows Autopilot Device Preparation Devices  
Microsoft Entra roles can be assigned to the group: No  
Membership Type: Assigned

![](/assets/6/assigned_group1.png)

Add **Intune Provisioning Client** as the owner of the group

![](/assets/6/assigned_group2.png)

### Create a user group

Windows Autopilot device preparation uses a user group to determine the users that should be targeted by the device preparation policy. The group must be a security group, though the membership type can be either assigned or dynamic. For this setup, I am using the **Corporate Users** group.

### Assign applications to device group

Windows device preparation policy allows you to deploy up to 10 apps and scripts during the deployment process. 

Navigate to Microsoft Intune admin center → Apps → Windows → Create

![](/assets/6/app_0.png)

Select app type: Line-of-business. I am adding Cloudflare Warp using its msi file so this option makes the most sense.

![](/assets/6/app_0_1.png)

Specify Name and Publisher etc.

![](/assets/6/app_1.png)

Assign it to the **Windows Autopilot Device Preparation Devices** group under “Required” which will ensure that it’s automatically installed on enrolled devices.

Other options are:

* **Available for enrolled devices:** apps are displayed in the Company Portal app and website for users to optionally install.  
* **Uninstall:** Apps with this assignment are uninstalled from managed devices in the selected groups.

![](/assets/6/app_2.png)

Verify app creation and add additional apps using the same method. I added FileZilla as available for enrolled devices. It is a Win32 app so I had to convert the installation file into .intunewin format using Microsoft Win32 Content Prep Tool before uploading it to intune.

![](/assets/6/app_3.png)

It’s important to note that Windows Autopilot device preparation deployment runs before any user logs in so apps and scripts must run as the system, not as a user. Apps and scripts that need to be installed in the user context should be configured after the deployment is complete.

### Create an intune policy for remote desktop users (optional)

Since I am using a VM for my lab, I need to ensure that my account is added to the device’s local **Remote Desktop Users** group; otherwise, I would not be able to log in via RDP. This is not required in an enterprise environment with physical devices. However, since I had to do this for the lab, I decided to document the process.

To manage this properly, I will create a security group for users who need to be added to the local **Remote Desktop Users** group. I will then create an Intune policy to manage local group membership by adding this security group to the **Remote Desktop Users** local group on the intune devices.

Navigate to Entra ID \-\> Groups \-\> New Group

**Configuration Values:**  
Group Type: Security  
Group Name: Windows Remote Desktop Users  
Microsoft Entra roles can be assigned to the group: No  
Membership type: Assigned

![](/assets/6/rdp_1.png)

Navigate to Microsoft Intune admin center \-\> Endpoint Security \-\> Account Protection \-\> Create Policy

![](/assets/6/rdp_2.png)

**Configuration Values:**

![](/assets/6/rdp_3.png)

I named the policy **Remote Desktop Users Policy**

![](/assets/6/rdp_4.png)

**Configuration Values:**

Local group: Remote Desktop Users

Group and user action: Add (Update)

User selection type: Users/Groups

Selected user(s): **Windows Remote Desktop Users** group

![](/assets/6/rdp_5.png)

Added **Windows Autopilot Device Preparation Devices** group under Assignments tab

![](/assets/6/rdp_6.png)

Review and create the policy

![](/assets/6/rdp_7.png)

## RBAC

In an enterprise environment, it is best practice to create a custom role specifically for Windows Autopilot device preparation administration, following the principle of least privilege.

Navigate to Microsoft intune admin center \-\> Tenant administration \-\> Create \-\> Intune role

![](/assets/6/custom_role_a.png)

I named the role **“Windows Autopilot Administrator”**

![](/assets/6/custom_role_ab.png)

Only allow permissions required for device preparation for autopilot.

![](/assets/6/custom_role_c.png)  
![](/assets/6/custom_role_b.png)  

Define scope and add members through role assignment.

Navigate to **Windows Autopilot Administrator \-\> Assignments**

![](/assets/6/custome_role_0.png)

I named the assignment **Windows Autopilot Device Preparation Admins** 

![](/assets/6/custome_role.png) 

Add the **Autopilot Administrators** group - containing users responsible for administering Windows Autopilot device preparation - under Admin groups

![](/assets/6/custome_role1.png) 

Add the **Windows Autopilot Device Preparation Devices** group under group scopes

![](/assets/6/custome_role2.png) 

Confirm the role assignment

# Deployment and Testing

With all requirements fulfilled, it’s time to deploy a Windows Autopilot device preparation policy and see the results of my work.

Navigate to Microsoft Intune admin center \-\> Windows Autopilot device preparation \-\> Device preparation policies

![](/assets/6/windows_prep1.png) 

Create a new user driven policy

![](/assets/6/windows_prep2.png) 

I named it **Windows11\_Corporate**

![](/assets/6/windows_prep3.png) 

Add the **Windows Autopilot Device Preparation Devices** group in Device group tab

![](/assets/6/windows_prep4.png) 

Configure configuration settings:

![](/assets/6/windows_prep5.png) 

![](/assets/6/windows_prep6.png) 

![](/assets/6/windows_prep7.png)

![](/assets/6/windows_prep8.png) 

Add the **Corporate Users** group in Assignments tab 
 
![](/assets/6/windows_prep9.png) 

![](/assets/6/windows_prep10.png) 

It’s time for testing so I will switch to the Windows 11 VM.

Ensure that the device used for Windows Autopilot device preparation is not registered as a Windows Autopilot device. If it is already registered, the Windows Autopilot profile will take precedence over the device preparation policy.

Let’s observe the OOBE flow and the remaining steps shown visually in the screenshots below.

![](/assets/6/deployment_1.png) 

![](/assets/6/deployment_2.png)

![](/assets/6/deployment_3.png)

![](/assets/6/deployment_4.png)

![](/assets/6/deployment_5.png)

![](/assets/6/deployment_6.png)  
![](/assets/6/deployment_7.png)

![](/assets/6/deployment_8.png)

![](/assets/6/deployment_9.png)

![](/assets/6/deployment_10.png)

![](/assets/6/deployment_11.png)

Navigate to Settings \-\> Accounts \-\> Access work or school to confirm that the device is enrolled in Intune, the required Intune policies are applied, and the specified apps are installed.

![](/assets/6/deployment_12.png)

Ensured that all required apps are installed from the control panel  
![](/assets/6/deployment_13.png)

Ensure that the device shows up in the Intune portal and is compliant

![](/assets/6/deployment_14.png)

<br>
<br>

# **Official References**

| Topic | Official Link |
| :---- | :---- |
| Windows Autopilot device preparation requirements | [https://learn.microsoft.com/en-us/autopilot/device-preparation/requirements?tabs=software](https://learn.microsoft.com/en-us/autopilot/device-preparation/requirements?tabs=software) |
| Windows Autopilot device preparation scenarios  | [https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/scenarios](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/scenarios) |
| Windows Autopilot device preparation user-driven Microsoft Entra join in Intune | [https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-workflow](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-workflow) |
| Managing Win32 Apps | [https://learn.microsoft.com/en-us/intune/intune-service/apps/apps-win32-app-management](https://learn.microsoft.com/en-us/intune/intune-service/apps/apps-win32-app-management) |

<br>