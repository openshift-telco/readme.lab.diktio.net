```bash
#############################################################################
DISCLAIMER: THESE ARE UNSUPPORTED COMMUNITY TOOLS.

THE REFERENCES ARE PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
#############################################################################
```
# **DRAFT** work in progress

# Description
This illustrates how to set-up scalable GitOps structures utilising Helm chart templating with OpenShift-GitOps (ArgoCD) to deploy OpenShift bare-metal clusters through OpenSHift ACM.

# Lab1
This lab configures a pre-deployed (day-0) cluster as an ACM management hub cluster that deploys a 3-node compact cluster using GitOps and ACM. The environment is connected to the Internet and therefore mirroring is not required. Note that the GitOps scripts are contain unused structures intended for a disconnected environment that will be utilised in Lab2.

The topology of  the lab is as illustrated below:

---
<p align="center">
  <img src="slides/topology.png" alt="drawing" width="666"/>
</p>

---


- SNO Management cluster:
    - VM 
        - 40 x vCPU
        - 64GB RAM
        - 128GB OS disk
        - 768GB ODF-LVM disk
    - Deployed by Assisted-Installer SaaS (https://console.redhat.com/openshift/)
    - OCP 4.10.11
- Compact cluster:
    - 3 x Supermicro X12 Workstation Motherboard
        - Intel i9 - 10 core
        - 64GB RAM
        - 1TB NVMe for OS
        - 500GB SSD for ODF
        - 2 x 10GE Intel-x710
- HTTP server (optional) to hold some iso images to save downloading down 80Mbps Internet link ;-)
- DNS server for resolution of lab domain and nodes and VIPs:
    - Management cluster entries:
        - master1.manage.lab.diktio.net
        - api.manage.lab.diktio.net
        - *.apps.manage.lab.diktio.net
    - Compact cluster entries:
        - master1.compact.lab.diktio.net
        - master2.compact.lab.diktio.net
        - master3.compact.lab.diktio.net
        - api.compact.lab.diktio.net
        - *.apps.compact.lab.diktio.net

# Lab2
This lab configures a pre-deployed (day-0) compact 3-node cluster as an ACM management hub cluster that deploys clusters using ACM ZTP-Workflow and/or ACm Assisted-Installer GitOps. The environment is connected to the Internet for the Management cluster but the deployed target clusters are disconnected and therefore mirroring is required.

The topology of  the lab is as illustrated below:

---
<p align="center">
  <img src="slides/topology-lab2.png" alt="drawing" width="666"/>
</p>

---


- Compact 3-node Management cluster:
    - 3 x VMs each with:
        - 14 x vCPU
        - 64GB RAM
        - 128GB OS disk
    - Deployed by Assisted-Installer SaaS (https://console.redhat.com/openshift/)
    - OCP 4.10.13
- SNO Clusters (SNO1, SNO2 and SNO3):
    - 3 x Supermicro X12 Workstation Motherboard
        - Intel i9 - 10 core
        - 64GB RAM
        - 1TB NVMe for OS
        - 500GB SSD for ODF-LVM
        - 2 x 10GE Intel-x710
- SNO Cluster (SNO11):
    - 1 x Supermicro X11 Workstation Motherboard
        - Intel i9 - 8 core
        - 64GB RAM
        - 1TB NVMe for OS
        - 500GB SSD for ODF-LVM
        - 2 x 10GE Intel-x710
- HTTP server (optional) to hold some iso images to save downloading down 80Mbps Internet link ;-)
- DNS server for resolution of lab domain and nodes and VIPs:
    - Management cluster entries:
        - master1.ocpgmt.lab.diktio.net
        - api.ocpmgmt.lab.diktio.net
        - *.apps.ocpmgmt.lab.diktio.net
    - Compact cluster entries:
        - master1.smcdu1.lab.diktio.net
            - api.smcdu1.lab.diktio.net
            - *.apps.smcdu1.lab.diktio.net
        - master1.smcdu2.lab.diktio.net
            - api.smcdu2.lab.diktio.net
            - *.apps.smcdu2.lab.diktio.net
        - master1.smcdu3.lab.diktio.net
            - api.smcdu3.lab.diktio.net
            - *.apps.smcdu3.lab.diktio.net
        - master1.smcdu11.lab.diktio.net
            - api.smcdu11.lab.diktio.net
            - *.apps.smcdu11.lab.diktio.net

## Github account requirements
The GitOps scripts for this lab expect HTTP token authentication to a personal account on github.com as this is a simple lab set-up.

For production deployments it is recommended to consider using a git org where you can set up read-only bot accounts for accessing the repositories.

The provided GitOps scripts may require some modifications if using another git online service or private git deployment.

# Customising the Management cluster configuration
## Step 0: Deploy the Management base cluster
This lab uses VM primarily for the reason that the physical hardware at hand did not have enough CPU resources to allow all operators including ACM observability (not deployed in current release od the lab). The VM is on a physical machine that has a 10 core i9 (20 vCPU) so this hypervisor is effectively 200% oversubscribed as the VM is given 40 CPU. A physical machine with a 16+ core (32+ vCPU) CPU will likely also do.

Whether a VM or BM is used for the Management cluster, it **must** be installed by IPI BM or Assisted-Installer as the bare-metal host operator is required.

The configuration deployed by this lab release assumes SNO and therefore deploys ODF-LVM (tech-preview) for central storage. For a multi-node management cluster ODF will be required and will be part of future releases. The GitOps code for full ODF is there but the management cluster configuration blueprint is tuned for ODF-LVM SNO.

The installation of the day-0 management  base cluster is beyond the scope of this set of instructions as it is believed to be well known to users and documented in OCP documentation. 

## Step 1: Edit Global values (/global/values.yaml)
Either create your own new repository or fork the supplied repository so you can customise the global/values.yaml file for your lab environment.

Parameters you must customise:
- global.domain - Your domain.
- global.git_user - Your Git service account name.
- global.git_token - Your HTTP access token to your Git account.
- global.dns - A local DNS server where you created the entires listed above. This will make life easier than trying to juggle things with /etc/hosts files. The actual build for this lab does require DNS entries (a DNS server address is though) as we'll be statically naming nodes.
- global.git_cluster_prefix_url - The URL prefix to your Git service account.
- global.mirror.list [1st entry] - Change to the desired OCP v4.10 release you'd like. Note that anything less than v4.10.9 has not been tested.
- global.mirror.list.install_mirror_name - Match the global.mirror.list.name you desire to install.
- global.ai.ssh_key - SSH public key to allow access to nodes.
- global.ai.pull_secret_name - Please do NOT change for ACM 2.4.X

## Step 2: Edit Management CLuster values (/<MGMT_FQDN>/values.yaml)

Parameters you must customise:
- mgmt.name - Name of the cluster.
- mgmt.git_url - The URL to your management cluster repository.
- mgmt.values_location - Replace the FQDN to  your management clusters FQDN (/values/<MGMT_FQDN>/values.yaml).
- mgmt.nodes.masters[name] - The FQDN of your SNO node.
- mgmt.os_images.root_fs_img_url - IP address of local HTTP server else uncomment Internet URL line.
- mgmt.os_images.root_fs_iso_url - IP address of local HTTP server else uncomment Internet URL line.
- mgmt.bmh.root_fs_img_url - IP address of local HTTP server else uncomment Internet URL line.

## Step 3: Create Management Cluster Repository (manage.lab.diktio.net)

Start with a blank repository:
```bash
cd <repository_path>
mkdir -p clusters

cp -r ../mgmt-manage.lab.diktio.net/deploy.sh \
  ../mgmt-manage.lab.diktio.net/gitops-push-operator-mirror .

```
Now link the blueprints:

- **Blueprint for SNO Management Hub cluster**
```bash
git submodule add https://github.com/openshift-telco/bp-mgmt-sno-acm.lab.diktio.net.git bp-mgmt
```

- **Mapping to environment wide common configurations and worker node blueprints**
```bash
git submodule add https://github.com/openshift-telco/bp-common.lab.diktio.net.git bp-common
```
- **Mapping environment and region wide Helm values.yaml files**
```bash
git submodule add <URL_to_your_values_repo> values
```
## Step 4: Configure the Management cluster via GitOps
At this point the values repository should be updated and committed **and** all submodules updated for the management cluster.

Update the repositories:
```bash
cd <repository_path>/bp-mgmt
git pull

cd ../bp-common
git pull

cd ../values
git pull

cd ../
git commit -a -m "blueprint sync"
git push
```

## Step 5: Kick off the Management cluster configuration
To kick off the deployment, you will need to have the Helm 3 CLI tool installed on your machine. This can be downloaded form https://console.redhat.com/openshift/downloads under the "Developer tools" section. 

Now kick off deployment with the deploy.sh script that runs 2 helm install commands to install the Openshift-gitops operator and configure it with a minimal config to start the pull from the repository and start configuring itself.

```bash
cd <repository_path>
bash deploy.sh <FQDN_management_cluster>
```
# Customising the target cluster deployment
## Step 1: Create Target Cluster Repository (cluster-compact.lab.diktio.net)

Start with a blank repository:
```bash
cd <repository_path>
mkdir -p workers

cp -r ../cluster-compact.lab.diktio.net/values.yaml .
```
### Now link the blueprints:

- **Blueprint for SNO Management Hub cluster**
```bash
git submodule add https://github.com/openshift-telco/bp-cluster-compact-acm-core.lab.diktio.net.git bp-cluster
```

- **Mapping to environment wide common configurations and worker node blueprints**
```bash
git submodule add https://github.com/openshift-telco/bp-common.lab.diktio.net.git bp-common
```
- **Mapping environment and region wide Helm values.yaml files**
```bash
git submodule add <URL_to_your_values_repo> values
```
## Step 2: Customise the values.yaml for the target cluster

### Parameters you must customise:
- cluster.name - Cluster name.
- cluster.api_vip - API VIP.
- cluster.ingress_vip - Ingres VIP.
- cluster.default_router - Local default-router. 
- cluster.install_mirror_name - OCP version to install. Must match the global.mirror.list.name you want.
- cluster.nodes.masters[boot_dev_hint] (each node) - The desired disk to install RHCOS
- cluster.nodes.masters[IP] (each node) - The static IP address of the given node.
- cluster.nodes.masters.interfaces[LIST] (each node) - List all interfaces on each node making sure the primary cluster-network interface is 1st in the list.
- cluster.nodes.masters[bmc_username] (each node) - Username for BMC access.
- cluster.nodes.masters[bmc_password] (each node) - Passowrd for BMC access.
- cluster.nodes.masters[bmc_address] (each node) - URL for nodes Redfish access.

## Step 3: Deploy the target cluster via GitOps
At this point the values repository should be updated and committed **and** all submodules updated for the target cluster.

Update the repositories:
```bash
cd <target_repository_path>/bp-cluster
git pull

cd ../bp-common
git pull

cd ../values
git pull

cd ../
git commit -a -m "blueprint sync"
git push
```
Kick of the deployment:
```bash
cd <management_repository_path>
mkdir -p clusters/<FQDN_Target_CLuster>
touch clusters/<FQDN_Target_CLuster>/deploy

git commit -a -m "Deploy Target CLuster"
git push
```

# Git Submodule Concept

---
<p align="center">
  <img src="slides/git_submodules.png" alt="drawing" width="666"/>
</p>

---
# Helm Values Build

---
<p align="center">
  <img src="slides/values.png" alt="drawing" width="666"/>
</p>

---
# Repository Links
## Global and Regional values.yaml files:
https://github.com/openshift-telco/values-global-lab.diktio.net

## Environment wide configurations and Node profiles:
https://github.com/openshift-telco/bp-common.lab.diktio.net

## Blueprint for Management (Hub) cluster configuration profile type SNO:
https://github.com/openshift-telco/bp-mgmt-sno-acm.lab.diktio.net

## Blueprint for target cluster profile type compact (3-node):
https://github.com/openshift-telco/bp-cluster-compact-acm-core.lab.diktio.net

## Management cluster repository:
https://github.com/openshift-telco/mgmt-manage.lab.diktio.net

## Target compact cluster repository:
https://github.com/openshift-telco/cluster-compact.lab.diktio.net
