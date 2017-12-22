# [DRAFT] Azure HPC Cluster with Lustre attached
This repository was created for a simple configuration of an HPC cluster inside of Azure with a Lustre File System configured and mounted.

Table of Contents
=================
* [Quickstart](#Lustre)
* [Lustre](#Lustre)
* [Deployment steps](#deployment-steps)
  * [Deploy Lustre MDS/MGS](#Deploy-the-Lustre-MDS/MGS)
  * [Deploy Lustre OSS](#Deploy-Lustre-OSS)
  * [Deploy Lustre Client](#Deploy-Lustre-Client)

# Quickstart
To deploy an Infiniband enabled compute cluster with a Lustre File Server attached and mounted:
1. Make sure you have quota for H-series (compute cluster) and F-series (jumpbox and storage cluster)

2. Open the [cloud shell](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/cloud-shell/quickstart.md) from the Azure portal

3. Clone the repository, `git clone https://github.com/tanewill/azhpc_lustre`

4. Change directory to azhpc_lustre `cd azhpc_lustre`

5. Deploy the cluster `./deploy.sh [RESOURCE_GROUP_NAME] [NUM_OSS_SERVERS] [NUM_SERVER_DISKS] [NUM_COMP_NODES]`
   - For example: `./deploy.sh LUSTRETESET-RG100 4 10 5`
   - This example would be a file server with 4 OSS servers and 40 total disks, for 160TB and 5 compute nodes
   - The total disk size is the number of OSS Servers multipled by the number of disks per server multipled by 4TB

6. Complete deployment will take around 40 minutes

7. The ssh key will be displayed upon completion, login to the jumpbox with that command

8. Compute node hostips are listed in the file 

# Purpose
The purpose of this article is to provide an introduction to IaaS HPC and HPC storage in the cloud and to provide some useful tools and information to quickly setup an HPC cluster with four different types of storage. Lustre is currently the most widely used parallel file system in HPC solutions. Lustre file systems can scale to tens of thousands of client nodes, tens of petabytes of storage. Lustre file system performed well for large file system, you can refer the testing results for the same.

# Introduction
High Performance Computing and storage in the cloud can be very confusing and it can be difficult to determine where to start. This repository is designed to be a first step in expoloring a cloud based HPC storage and compute architecture. There are many different configuration that could be used, but this repository focuses on an RDMA connected compute cluster and a Lustre file system that is attached. Three different deployment strategies are used, a Bash script for orchastration, an Azure Resource Manager (ARM) template for the compute cluster, and Azure Batch Shipyard for the file server deployment. After deployment fully independant and functioning IaaS HPC compute and storage cluster has been deployed based on the architecture below.

# HPC in the Cloud
- HPC in the cloud continues to gain momentum. 
[Inside HPC Article](https://insidehpc.com/2017/03/long-rise-hpc-cloud/)
[The Cloud is Great for HPC](https://www.theregister.co.uk/2017/06/16/the_cloud_is_great_for_hpc_discuss/)
		
- Azure's play in the HPC space has been significant
  * Infiniband enabled hardware
  * [H-Series](https://azure.microsoft.com/en-us/blog/availability-of-h-series-vms-in-microsoft-azure/)
  * [Massive HPC deals at financial services institutions, Oil and Gas companies, etc](https://www.forbes.com/sites/alexkonrad/2017/10/30/chevron-partners-with-microsoft-in-cloud/)
  * [Cray offering](https://www.cray.com/solutions/supercomputing-as-a-service/cray-in-azure)
		
- Unlike traditional HPC environments, cloud HPC environments can be created and destroyed quickly, completely, and easily in an Ad-Hoc and On Demand fashon. With large physical disks, many storage requirments can be satisified using the attached physical disks.
	
- Now with Azure enabling over 4,000 cores for a single Infiniband enabled MPI job the dataset size can potential exceed the 2TB attached Solid State Disks. With these large datasets a simple and flexible storage solution is needed.

# Architecture
## Example HPC Data Architecture
![alt text](https://github.com/tanewill/azhpc_lustre/blob/master/images/HPC_DataArch.png)

## Estimated Monthly Cost for North Central US
Estimates calculated from [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
 - Compute, 80 H16r cores
   - 5 H16 compute nodes @ 75% utilization, $5,459.81/month 
 - Storage, 256 TB
   - 1 DS3_v2 MGSMDT Server, $214.48/month
   - 8 F8s OSS Servers, $2,330.69/month
   - 64 (8/OSS Server) Premium, P50 Managed Disks. 256 TB, $31,716.20/month
   - 15 TB Azure Files, $912.63/month

Total Cost about $40,633.81/month (~$36,764.88/month with 3 year commit)

## Storage Deployment
There are four different types of storage that will be used for this HPC cluster. Using the default configuration there is over 29TB available for this compute cluster.

 * Physically Attached Storage as a burst buffer, located at /mnt/resource on each node
 * NFS shared from the jumpbox and located at /mnt/scratch, created in the hn-setup script here: hn-setup_gfs.sh
 * GFS shared from the storage cluster mounted at /mnt/gfs, created using Batch Shipyard, link, here in create_cluster.sh
 * Three 5TB Azure Files shares mounted to the jumpbox at /mnt/lts1,/mnt/lts2,/mnt/lts3. This is a CIFS share and can be mounted to both the Windows and Linux operating systems. These Azure File shares are subject to performance limits specified here. The size can be altered by increasing the quota here: create_cluster.sh

Below is an image that attempts to visualize the needed storage structure for an example workload. The Physically attached storage is the temporary storage, the Lustre is for the 'campaign' data that supports multiple workloads, finally the Azure Files share is for long term data retention.


![alt text](https://github.com/tanewill/azhpc_lustre/blob/master/images/WorkloadData.png)

## File System Architecture
Lustre clusters contain four kinds of systems:
 * File system clients, which can be used to access the file system.
 * Object storage servers (OSSs), which provide file I/O services and manage the object storage targets (OSTs).
 * Metadata servers (MDSs), which manage the names and directories in the file system and store them on a metadata target (MDT).
 * Management servers (MGSs), which work as master nodes for the entire cluster setup and contain information about all the nodes attached within the cluster. 
   - A single node can be used to serve as both an MDS and MGS.

![Lustre Architecture](/images/lustre_arch.png)

Note- Before setup Lustre FS make sure you have service principal (id, secrete and tenant id) to get artifacts from Azure.

# Rest API Deployment

* Deploy the Lustre MDS/MGS

  [![Click to deploy template on Azure](/images/deploybutton.png "Click to deploy template on Azure")](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftanewill%2Fazhpc_lustre%2Fmaster%2Ftemplates%2Flustre-master.json) 

* Deploy the Lustre OSS

  [![Click to deploy template on Azure](/images/deploybutton.png "Click to deploy template on Azure")](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftanewill%2Fazhpc_lustre%2Fmaster%2Ftemplates%2Flustre-server.json)

* Deploy the Lustre Clients

  [![Click to deploy template on Azure](/images/deploybutton.png "Click to deploy template on Azure")](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Ftanewill%2Fazhpc_lustre%2Fmaster%2Ftemplates%2Flustre-client.json)

