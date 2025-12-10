SCCM (now rebranded as Microsoft Endpoint Configuration Manager - MECM)
will still be called SCCM throughout this documentation because... come
on Microsoft.

This project walks through the infrastructure setup and installation
phase, configured as close to an enterprise build as possible while
still scaled down for a lab environment.

I have divided the project into four major steps:

- Set Up Active Directory Infrastructure (FA-DC01)

- Build SQL + Site Server for SCCM (FA-SCCM)

- Prepare FA-SCCM for SCCM Installation

- Install System Center Configuration Manager

Post-configuration tasks (boundaries, discovery methods, clients, OSD,
etc.) will be documented separately.

**Infrastructure Components:**

  --------------------------------------------------------------
  **Virtual            **Description**        **IP Address**
  Component**                                 
  -------------------- ---------------------- ------------------
  Address Space        Address space used in  172.16.171.0
                       this lab               

  FA-DC01              Domain Controller      172.16.171.5

  FA-SCCM              SCCM Site Server + SQL 172.16.171.10
                       Server                 
  --------------------------------------------------------------

**Download Resources:**

  --------------------------------------------------------------------------------------------------------------------------------
  **Name**                        **Download Link**
  ------------------------------- ------------------------------------------------------------------------------------------------
  Windows Server 2022             <https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022>

  SQL Server 2022                 <https://www.microsoft.com/en-us/evalcenter/evaluate-sql-server-2022>

  SQL Reporting Services 2022     <https://www.microsoft.com/en-us/download/details.aspx?id=104502>

  SQL Server Management Studio    <https://learn.microsoft.com/en-us/ssms/install/install>

  Windows ADK and WinPE Add-on    <https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install>

  Configuration Manager 2403      <https://www.microsoft.com/en-us/evalcenter/evaluate-microsoft-endpoint-configuration-manager>
  (current branch)                
  --------------------------------------------------------------------------------------------------------------------------------

All software listed above is compatible with Configuration Manager
version 2403 (verified).

# 1. Set Up Active Directory Infrastructure (FA-DC01)

Created a virtual machine, installed Windows Server 2022, added the AD
DS role, and promoted the server to a domain controller.

**Configuration Values:**

- Server Name: **FA-DC01**

- Static IP: **172.16.171.5**

- Hostname: **FA-DC01**

- Domain Name: **farhan-lab.local**

- FQDN: **fa-dc01.farhan-lab.local**

![](media/image1.png){width="5.366407480314961in"
height="3.36833552055993in"}

**DNS Configuration**

Added Domain Name Services (DNS) role on the server and verified
functionality.

- nslookup farhan-lab.local

<!-- -->

- nslookup FA-DC01.farhan-lab.local

- nslookup 172.16.171.5

- Created reverse lookup zone (172.16.171.x)

![](media/image2.png){width="2.18169728783902in"
height="3.173377077865267in"}\
\
![](media/image3.png){width="6.5in" height="2.282638888888889in"}\
\
![A screenshot of a computer AI-generated content may be
incorrect.](media/image4.png){width="6.5in"
height="2.1194444444444445in"}

**OU Structure**\
Created an OU structure suited for my lab. This is optional, but I
prefer creating a dedicated root OU so GPO scope is clean.

![](media/image5.png){width="4.486679790026247in"
height="3.9389391951006125in"}

**Service Accounts**\
Created service accounts to be used by SCCM and permissions for service
accounts are tailored for their specific function following the least
access model.

  ----------------------------------------------------------------------------
  **Service Account**  **Purpose**
  -------------------- -------------------------------------------------------
  SCCM_Admin           Used to install SCCM and perform admin-level
                       configuration

  SCCM_SQLService      Runs SQL services

  SCCM_NetworkAccess   Used for client communication / network access (not
                       relevant for this project)

  SCCM_ClientPush      Used for pushing SCCM client agents (not relevant for
                       this project)
  ----------------------------------------------------------------------------

**System Management Container**

- Created **System Management** container in ADSIEdit

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image6.png){width="2.5078860454943133in"
> height="2.883665791776028in"}\
> \
> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image7.png){width="2.7886439195100614in"
> height="2.3476476377952755in"}\
> \
> ![](media/image8.png){width="5.3366535433070865in"
> height="2.568017279090114in"}

- Delegated **Full Control** to **FA-SCCM\$** (computer account)

> ![](media/image9.png){width="2.417487970253718in"
> height="2.7097790901137357in"}\
> \
> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image10.png){width="5.749211504811899in"
> height="3.860488845144357in"}\
> \
> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image11.png){width="5.772317366579178in"
> height="3.5024070428696414in"}

# 2. Build SQL + Site Server for SCCM (FA-SCCM)

I am using one server for both SQL and SCCM (collocated). This is fine
for labs and small organizations but not recommended for large
enterprises due to higher I/O requirements which may limit the high
availability options.

Installed Windows Server 2022 on a new virtual machine and joined it to
domain.

**Configuration Values:**

- Static IP: **172.16.171.10**

- Preferred DNS Server: **172.16.171.5** (FA-DC01)

- Hostname: **FA-SCCM**

- Join domain: **farhan-lab.local**

![A screenshot of a computer AI-generated content may be
incorrect.](media/image12.png){width="5.680555555555555in"
height="3.613442694663167in"}

![](media/image13.png){width="2.8516174540682413in"
height="3.340057961504812in"}

**Storage Preparation for SCCM**

Disk allocation is critical for SCCM performance. Microsoft recommends
dedicated RAID volumes for database files.

I created separate virtual disks for (scaled down the disk space for my
lab environment):

  ------------------------------------------------------------------------
  **Virtual Hard Drive**      **Disk    **Usage**
                              Space**   
  --------------------------- --------- ----------------------------------
  SCCM_INSTALL                50GB      Configuration Manager application
                                        and log files

  SCCM_SQL_MDF                20GB      Site database data file (.mdf)

  SCCM_SQL_LDF                15GB      Site database log file (.ldf)

  SQL_TempDB                  15GB      Temp database files (.mdf and
                                        .ldf)

  SQL_WSUS_Database           10GB      WSUS database files
  (optional)                            

  SCCM_Application_Sources    10GB      Application source files for
  (optional)                            software deployments

  SCCM_ContentLibrary         10GB      All content files for software
  (optional)                            deployments
  ------------------------------------------------------------------------

![A screenshot of a computer AI-generated content may be
incorrect.](media/image14.png){width="7.5in"
height="2.265277777777778in"}

Created a file named **no_sms_on_drive.sms** and placed it in every
drive except SCCM_ContentLibrary to prevent the content library from
being installed on any other drive.

**Install SQL Server on FA-SCCM**

- Install SQL Server 2022

![](media/image15.png){width="5.996884295713036in"
height="2.673611111111111in"}**\**

- Enable Database Engine Services

> ![](media/image16.png){width="6.051550743657043in"
> height="3.2118055555555554in"}

- Verify collation is set to: **SQL_Latin1_General_CP1_CI_AS**

- Configure SQL services to run as **SCCM_SQLService** (service account)

> ![](media/image17.png){width="6.006944444444445in"
> height="4.388406605424322in"}

- Add **SCCM_Admin** (service account) as SQL **sysadmin\**
  ![](media/image18.png){width="5.965277777777778in"
  height="4.468435039370079in"}**\**

- Change tempDB location → SQL_TempDB

  - Increase tempDB autogrowth to 256MB (optional: so the file growth is
    not as often)

**Configure Firewall Rules for SQL Server**

Allow inbound domain traffic for ports: **1433** and **4022**

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image19.png){width="4.648075240594926in"
> height="3.7223982939632547in"}
>
> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image20.png){width="4.629863298337708in"
> height="3.753943569553806in"}\
> \
> ![](media/image21.png){width="4.643849518810149in"
> height="3.7760258092738406in"}\
> \
> ![](media/image22.png){width="4.627760279965004in"
> height="3.6730358705161854in"}

**Additional Software and Updates**

- Install latest SQL Server 2022 Cumulative Update

- Install SQL Reporting Services 2022

- Install SQL Server Management Studio (SSMS)

**SQL Memory Configuration**

- Set SQL Min Memory: **80% of RAM**

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image23.png){width="4.683819991251093in"
> height="3.9053630796150482in"}

# 3. Prepare FA-SCCM for SCCM Installation

**Install Windows Roles and Features**

Install **BITS**, **RDC**, **IIS**, and other pre-requisites using
PowerShell command:

Install-WindowsFeature
Web-Static-Content,Web-Default-Doc,Web-Dir-Browsing,Web-Http-Errors,Web-Http-Redirect,Web-Net-Ext,Web-ISAPI-Ext,Web-Http-Logging,Web-Log-Libraries,Web-Request-Monitor,Web-Http-Tracing,Web-Windows-Auth,Web-Filtering,Web-Stat-Compression,Web-Mgmt-Tools,Web-Mgmt-Compat,Web-Metabase,Web-WMI,BITS,RDC

**WSUS Installation for Software Update Point (SUP) Configuration**

- Add WSUS role

- Select **SQL Connectivity** instead of WID

> ![](media/image24.png){width="3.6338134295713034in"
> height="3.3028390201224846in"}

- WSUS content path → SCCM_ContentLibrary drive

- Set DB server → FA-SCCM.farhan-lab.local

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image25.png){width="5.362775590551181in"
> height="2.386434820647419in"}

- Launch WSUS Post-Install Tasks

Withing SQL Server Management Studio:

- Increase WsusPool Queue Length to **2000**

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image26.png){width="4.735015310586177in"
> height="2.997088801399825in"}

- Increase Private Memory Limit ×4

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image27.png){width="2.359621609798775in"
> height="2.720120297462817in"}

**Move WSUS Database to Dedicated Disk**

- Stop WSUS service (IIS Manager + Services.msc)

> ![](media/image28.png){width="4.921135170603675in"
> height="3.1700306211723532in"}\
> \
> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image29.png){width="4.965299650043745in"
> height="2.9336646981627297in"}

Within SQL Server Management Studio:

- Copy existing SUSDB.mdf & SUSDB_log.ldf path

- Detach DB

> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image30.png){width="4.5173501749781275in"
> height="3.1412959317585303in"}\
> \
> ![A screenshot of a computer AI-generated content may be
> incorrect.](media/image31.png){width="4.498423009623797in"
> height="3.7099496937882765in"}

- Move SUSDB.mdf & SUSDB_log.ldf files from original location to
  **SQL_WSUS_Database** drive

- Re-attach DB

- Start WSUS services again (IIS Manager + Services.msc)

**ADK + WinPE**

- Install Windows ADK:

  - Deployment Tools

  - USMT

> ![](media/image32.png){width="4.1727252843394576in"
> height="3.0337259405074364in"}

- Install Windows PE add-on

- Install Windows ADK latest cumulative update

**Extend AD Schema for SCCM**

- Extract Configuration Manager setup files

- Run extadsch.exe from SMSSETUP\\BIN\\X64

- Verify success via:

- C:\\ExtADSch.log

> ![](media/image33.png){width="4.956397637795275in"
> height="2.856353893263342in"}

# 

# 4. Install System Center Configuration Manager 

**SCCM Installation**

Ran **Splash.hta**, installed a Primary Site.

**Configuration Values**

- **Site Code:** FAL

- **Site Name:** Farhan Ali Lab Primary Site

- **Install Path:** SCCM_INSTALL drive

- **SQL Server:** FA-SCCM.farhan-lab.local

- **Database:** SCCM_FAL

- Change SQL datafile path → SCCM_SQL_MDF drive

- Change SQL log path → SCCM_SQL_LDF drive

**Site System Roles**

- Install:

  - Management Point (MP)

  - Distribution Point (DP)

**Post-Install**

- Verify installation completed successfully

> ![](media/image34.png){width="5.046841644794401in"
> height="3.719242125984252in"}\
> \
> ![](media/image35.png){width="5.087671697287839in"
> height="2.649828302712161in"}

- Set **CMTrace.exe** default log viewer and copy it from source file to
  a different location so CM updates don't interfere with its
  functionality.

**Helpful logs (Microsoft Configuration Manager\\Logs):**

- sitecomp.log (site creation progress)

- CMUpdate.log

- hman.log

# Related Microsoft Documentation

1.  **Site and site system prerequisites for System Center Configuration
    Manager:**
    <https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/site-and-site-system-prerequisites>

2.  **Supported SQL Server versions for System Center Configuration
    Manager:**
    <https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/support-for-sql-server-versions>

3.  **Recommended hardware for System Center Configuration Manager:**
    [https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/recommended-hardware](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/recommended-hardware%20)

4.  **Supported Active Directory domains for System Center Configuration
    Manager:**
    <https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/support-for-active-directory-domains>

5.  **Prepare Active Directory for site publishing:**
    <https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/network/extend-the-active-directory-schema>

6.  **The content library in System Center Configuration Manager
    (no_sms_on_drive.sms file):**
    <https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/hierarchy/the-content-library>

Other References

Shout out to my guy **Justin Chalfant** at Patch My PC, couldn't have
done it without his detailed guide:
<https://youtu.be/amrg_mlFvuk?si=KSHFS0FwOEqUkIzP>
