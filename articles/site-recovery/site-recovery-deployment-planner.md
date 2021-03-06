---
title: Azure Site Recovery deployment planner for VMware-to-Azure| Microsoft Docs
description: This is the Azure Site Recovery deployment planner user guide.
services: site-recovery
documentationcenter: ''
author: nsoneji
manager: garavd
editor:

ms.assetid:
ms.service: site-recovery
ms.workload: storage-backup-recovery
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: hero-article
ms.date: 12/04/2017
ms.author: nisoneji

---
# Azure Site Recovery Deployment Planner for VMware to Azure
This article is the Azure Site Recovery Deployment Planner user guide for VMware-to-Azure production deployments.

## Overview

Before you begin protecting any VMware virtual machines (VMs) by using Site Recovery, allocate sufficient bandwidth, based on your daily data-change rate, to meet your desired recovery point objective (RPO). Be sure to deploy the right number of configuration servers and process servers on-premises.

You also need to create the right type and number of target Azure storage accounts. You create either standard or premium storage accounts, factoring in growth on your source production servers because of increased usage over time. You choose the storage type per VM, based on workload characteristics (for example, read/write I/O operations per second [IOPS], or data churn) and Site Recovery limits.

The Azure Site Recovery deployment planner (version 2) is a command-line tool available for both Hyper-V to Azure and VMware to Azure disaster recovery scenarios. You can remotely profile your VMware VMs by using this tool (with no production impact whatsoever) to understand the bandwidth and Azure Storage requirements for successful replication and test failover. You can run the tool without installing any Site Recovery components on-premises. However, to get accurate achieved throughput results, we recommend that you run the planner on a Windows Server that meets the minimum requirements of the Site Recovery configuration server that you would eventually need to deploy as one of the first steps in production deployment.

The tool provides the following details:

**Compatibility assessment**

* A VM eligibility assessment, based on number of disks, disk size, IOPS, churn, boot type(EFI/BIOS), and OS version
 
**Network bandwidth need versus RPO assessment**

* The estimated network bandwidth that's required for delta replication
* The throughput that Site Recovery can get from on-premises to Azure
* The number of VMs to batch, based on the estimated bandwidth to complete initial replication in a given amount of time

**Azure infrastructure requirements**

* The storage type (standard or premium storage account) requirement for each VM
* The total number of standard and premium storage accounts to be set up for replication
* Storage-account naming suggestions, based on Azure Storage guidance
* The storage-account placement for all VMs
* The number of Azure cores to be set up before test failover or failover on the subscription
* The Azure VM-recommended size for each on-premises VM

**On-premises infrastructure requirements**
* The required number of configuration servers and process servers to be deployed on-premises

**Estimated DR cost to Azure** 
* Estimated total DR cost to Azure: compute, storage, network, and Azure Site Recovery license cost
* Detail cost analysis per VM


>[!IMPORTANT]
>
>Because usage is likely to increase over time, all the preceding tool calculations are performed assuming a 30 percent growth factor in  workload characteristics, and using a 95th percentile value of all the profiling metrics (read/write IOPS, churn, and so forth). Both of these elements (growth factor and percentile calculation) are configurable. To learn more about growth factor, see the "Growth-factor considerations" section. To learn more about  percentile value, see the "Percentile value used for the calculation" section.
>

## Support matrix

| | **VMware to Azure** |**Hyper-V to Azure**|**Azure to Azure**|**Hyper-V to secondary site**|**VMware to secondary site**
--|--|--|--|--|--
Supported scenarios |Yes|Yes|No|Yes*|No
Supported Version | vCenter 6.5, 6.0 or 5.5| Windows Server 2016, Windows Server 2012 R2 | NA |Windows Server 2016, Windows Server 2012 R2|NA
Supported configuration|vCenter, ESXi| Hyper-V cluster, Hyper-V host|NA|Hyper-V cluster, Hyper-V host|NA|
Number of servers that can be profiled per running instance of the Azure Site Recovery Deployment Planner |Single (VMs belonging to one vCenter Server or one ESXi server can be profiled at a time)|Multiple (VMs across multiple hosts or host clusters can be profile at a time)| NA |Multiple (VMs across multiple hosts or host clusters can be profile at a time)| NA

*The tool is primarily for the Hyper-V to Azure disaster recovery scenario. For Hyper-V to secondary site disaster recovery, it can be used only to understand source side recommendations like required network bandwidth, required free storage space on each of the source Hyper-V servers, and initial replication batching numbers and batch definitions.  Ignore the Azure recommendations and costs from the report. Also, the Get Throughput operation is not applicable for the Hyper-V to secondary site disaster recovery scenario.

## Prerequisites
The tool has two main phases: profiling and report generation. There is also a third option to calculate throughput only. The requirements for the server from which the profiling and throughput measurement is initiated are presented in the following table:

| Server requirement | Description|
|---|---|
|Profiling and throughput measurement| <ul><li>Operating system: Microsoft Windows Server 2016 or Microsoft Windows Server 2012 R2<br>(ideally matching at least the [size recommendations for the configuration server](https://aka.ms/asr-v2a-on-prem-components))</li><li>Machine configuration: 8 vCPUs, 16 GB RAM, 300 GB HDD</li><li>[Microsoft .NET Framework 4.5](https://aka.ms/dotnet-framework-45)</li><li>[VMware vSphere PowerCLI 6.0 R3](https://aka.ms/download_powercli)</li><li>[Microsoft Visual C++ Redistributable for Visual Studio 2012](https://aka.ms/vcplusplus-redistributable)</li><li>Internet access to Azure from this server</li><li>Azure storage account</li><li>Administrator access on the server</li><li>Minimum 100 GB of free disk space (assuming 1000 VMs with an average of three disks each, profiled for 30 days)</li><li>VMware vCenter statistics level settings should be set to 2 or high level</li><li>Allow 443 port: ASR Deployment Planner uses this port to connect to vCenter server/ESXi host</ul></ul>|
| Report generation | A Windows PC or Windows Server with Microsoft Excel 2013 or later |
| User permissions | Read-only permission for the user account that's used to access the VMware vCenter server/VMware vSphere ESXi host during profiling |

> [!NOTE]
>
>The tool can profile only VMs with VMDK and RDM disks. It cannot profile VMs with iSCSI or NFS disks. Site Recovery does support iSCSI and NFS disks for VMware servers but, because the deployment planner is not inside the guest and it profiles only by using vCenter performance counters, the tool does not have visibility into these disk types.
>

## Download and extract the deployment planner tool
1. Download the latest version of the [Azure Site Recovery deployment planner](https://aka.ms/asr-deployment-planner).  
The tool is packaged in a .zip folder. The current version of the tool supports only the VMware-to-Azure scenario.

2. Copy the .zip folder to the Windows server from which you want to run the tool.  
You can run the tool from Windows Server 2012 R2 if the server has network access to connect to the vCenter server/vSphere ESXi host that holds the VMs to be profiled. However, we recommend that you run the tool on a server whose hardware configuration meets the [configuration server sizing guideline](https://aka.ms/asr-v2a-on-prem-components). If you have already deployed Site Recovery components on-premises, run the tool from the configuration server.

 We recommend that you have the same hardware configuration as the configuration server (which has an in-built process server) on the server where you run the tool. Such a configuration ensures that the achieved throughput that the tool reports matches the actual throughput that Site Recovery can achieve during replication. The throughput calculation depends on available network bandwidth on the server and hardware configuration (CPU, storage, and so forth) of the server. If you run the tool from any other server, the throughput is calculated from that server to Microsoft Azure. Also, because the hardware configuration of the server might differ from that of the configuration server, the achieved throughput that the tool reports might be inaccurate.

3. Extract the .zip folder.  
The folder contains multiple files and subfolders. The executable file is ASRDeploymentPlanner.exe in the parent folder.

    Example:  
    Copy the .zip file to E:\ drive and extract it.
    E:\ASR Deployment Planner_v2.0zip

    E:\ASR Deployment Planner_v2.0\ASRDeploymentPlanner.exe

### Updating to the latest version of deployment planner
If you have previous version of the deployment planner, do either of the following:
 * If the latest version doesn't contain a profiling fix and profiling is already in progress on your current version of the planner, continue the profiling.
 * If the latest version does contain a profiling fix, we recommended that you stop profiling on your current version and restart the profiling with the new version.


 >[!NOTE]
 >
 >When you start profiling with the new version, pass the same output directory path so that the tool appends profile data on the existing files. A complete set of profiled data will be used to generate the report. If you pass a different output directory, new files are created, and old profiled data is not used to generate the report.
 >
 >Each new deployment planner is a cumulative update of the .zip file. You don't need to copy the newest files to the previous folder. You can create and use a new folder.

## Next steps
* [Run the deployment planner](site-recovery-vmware-deployment-planner-run.md).
