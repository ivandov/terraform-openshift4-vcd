
# OpenShift UPI Deployment with Static IPs on VMWare Cloud Director
## Overview
Deploy OpenShift 4.6 and later on VMWare Cloud Director using static IP addresses for CoreOS nodes.  The `ignition` module will inject code into the cluster that will automatically approve all node CSRs.  This runs only once at cluster creation.  You can delete the `ibm-post-deployment` namespace once your cluster is up and running.

**NOTE**: This requires OpenShift 4.6 or later, if you're looking for 4.5 or earlier, see the [VCD Toolkit](https://github.com/vmware-ibm-jil/vcd_toolkit_for_openshift) or the `terraform-openshift4-vmware pre-4.6` [branch](https://github.com/ibm-cloud-architecture/terraform-openshift4-vmware/tree/pre-4.6)

**NOTE**: Requires terraform 0.13 or later.  

**Change History:**
  - 3/5/2021:
    - **Due to networking changes and updated configurations and software on the Bastion, it is recommended that you not reuse an existing VDC and Bastion. If you really need or want to reuse your existing VDC, you should delete the existing Bastion as well as all the existing DNAT, SNAT and any FW rules that you created. There should be a Default rule to Deny all public and private access**
    - Added full creation of Bastion and all networking including creation of vdc network, fw rules, dnat and snat rules.
    - Full install and configure all software on Bastion including dnsmasq, ansible, terraform, oc client, nginx web server for ignition
    - Load terraform repo and terraform.tfvars onto Bastion
    - Load pull-secret and additionalTrustBundle cert if present on host machine.
    - When creating an OCP Cluster, all fw, dnat rules, Bastion /etc/hosts and dsnmasq.conf updates are performed when you create the cluster.


  - 2/18/2021:

   - Airgapped install is now supported. You need to build your own mirror.
   - When you create a cluster, the firewall rules and DNAT rule for that cluster will be automatically created. You should probably delete any DNAT or FW rules that relate to any clusters you have previously built. If you have previously created a Edge Firewall Rule for ALLOW Internet access for all resources on your vcd network, you probably should delete that rule. It will interfere with the new automated Firewall. The automation will add Firewall rules for all the VM's that require it when you create a cluster. You will need to add a new set of variables in your `terraform.tfvars` file. The instructions have been updated to relect this and only create rules for the Bastion server. If you create additional servers in your vcd, outside of the automation, you should add internet access to these servers by following the setup instructions for the Bastion.
   - At the end of the Terraform apply, several (hopefully) useful variables are printed out to help you out. Open an issue if you would like more/less/different info listed.   


- 2/14/2021 - Move explanation for our choice to use DHCP for static IP provisioning.  See https://github.com/ibm-cloud-architecture/terraform-openshift4-vcd/issues/3

- 2/01/2021 - Added "Experimental Flag" "create_vms_only". If you set this flag to true, OCP won't be installed, the vm's will be created and the OCP installer will be loaded to the installer/cluster_id directory. There is currently a bug so when you run `terraform apply` the first time, it fails with some error messages after creation of a few VM's but just run `terraform apply` again and it should complete successfully

- 1/25/2021 - The Loadbalancer VM will be automatically started at install time to ensure that DHCP and DNS are ready when the other machines are started.  

- 1/25/2021 - Create project.  This project is a combination of the terraform scripting from [ibm-cloud-architecture/terraform-openshift4-vmware](https://github.com/ibm-cloud-architecture/terraform-openshift4-vmware) and the setup instructions from [VCD Toolkit](https://github.com/vmware-ibm-jil/vcd_toolkit_for_openshift).
The benefits of this code vs. the VCD Toolkit are:
  - Supports OCP 4.6
  - Supports variable number of workers.  VCD toolkit hardcoded the worker count and  related LoadBalancer configuration.
  - Automatic approval of outstanding CSR's.  No more manual step.
  - Less manual steps overall.  Does not require multiple steps of running scripts, then terraform then more scripts. Just set variables, run terraform, then start VM's.



## Architecture

OpenShift 4.6 User-Provided Infrastructure

![topology](./media/vcd_arch.png)

# Installation Process
## Order a VCD
You will order a **VMware Solutions Shared** instance in IBM Cloud(below).  When you order a new instance, a **DataCenter** is created in vCloud Director.  It takes about an hour.

#### Procedure:
* in IBM Cloud > VMWare > Overview,  select **VMWare Solutions Shared**
* name your virtual data center
* pick the resource group.  
* agree to the terms and click `Create`
* then in VMware Solutions > Resources you should see your VMWare Solutions Shared being created.  After an hour or less it will be **ready to use**You will need to edit terraform.tfvars as appropriate, setting up all the information necessary to create your cluster. You will need to set the vcd information as well as public ip's, etc. This file will eventually be copied to the newly created Bastion.


#### Initial VCD setup
* Click on the VMWare Shared Solution instance named from the Resources list
* Set your admin password, and save it
* Click the button to launch your  **vCloud Director console**
* We recommend that you create individual Users/passwords for each person accessing the environment
* Make note of the 5 public ip address on the screen. You will need to use them later to access the Bastion and your OCP clusters
* Note: You don't need any Private network Endpoints unless you want to access the VDC from other IBM Cloud accounts over Private network

# Installing the Bastion and initial network configuration
## Setup Host Machine
You will need a "Host" machine to perform the initial Bastion install and configuration. This process has only been tested on a RHEL8 Linux machine but may work on other linux based systems that support the required software. You should have the following installed on your Host:
 - ansible [instructions here](https://docs.ansible.com/ansible/latest/installation_guide/index.html)
 - git
 - terraform [instructons here](https://www.terraform.io/downloads.html)


On your Host, clone the git repository. After cloning the repo, You will need to edit `terraform.tfvars` as appropriate, setting up all the information necessary to create your cluster. You will need to set the vcd information as well as public ip's, etc. Instructions on gathering key pieces of informaton are below.

```
git clone https://github.com/ibm-cloud-architecture/terraform-openshift4-vcd
cd terraform_openshift4-vcd
cp terraform.tfvars.example terraform.tfvars
```
Edit terraform.tfvars per the terraform variables section
## Gather Information for terraform.tfvars
#### Find vApp Template from the Image Catalog
We need a catalog of VM images to use for our OpenShift VMs and the Bastion.
Fortunately IBM provides a set of images that are tailored to work for OpenShift deployments.
To browse the available images:
* From your vCloud Director console, click on **Libraries** in the header menu.
* select *vApp Templates*
* There may be several images in the list that we can use, pick the one that matches the version of OCP that you intend to install:
  * rhcos OpenShift 4.6.8 - OpenShift CoreOS template
  * rhcos OpenShift 4.7.0 - OpenShift CoreOS template
  * RedHat-8-Template-Official
* If you want to add your own Catalogs and more, see the [documentation about catalogs](#about-catalogs)

#### Networking Info
VCD Networking is covered in general in the [Operator Guide/Networking](https://cloud.ibm.com/docs/vmwaresolutions?topic=vmwaresolutions-shared_vcd-ops-guide#shared_vcd-ops-guide-networking). Below is the specific network configuration required.


The Bastion installation process will now create all the Networking entries necessary for the environment. You simply need to pick
 - a **Network Name** (ex. ocpnet)
 - a **Gateway/CIDR** (ex. 172.16.0.1/24)
 - an **external** ip for use by the Bastion
 - an **internal** ip for use by the bastion


The Default FW rules created will Deny all traffic except for the Bastion which will have access both to the Public Internet and the IBM Cloud Private Network. DNAT and SNAT rules will be set up for the Bastion to support the above.

When you create a cluster, the FW will be set up as follows.
- The loadbalancer will always have Internet access as it needs to pull images from docker.io and quay.io in order to operate properly.
- If you do not request an airgap install, all workers and masters will be allowed access via the FW
- A DNAT rule will be set up so that you can access you cluster from your workstation regardless of whether or not you requested airgap.

DHCP is not enabled on the Network as it will interfere with the DHCP server running in the cluster. If you have previously enabled it for use in the vcd toolkit, you should now disable it.


#### Choosing an External IP  for your cluster and Bastion and retrieving the Red Hat Activation key
Configure the Edge Service Gateway (ESG) to provide inbound and outbound connectivity.  For a network overview diagram, followed by general Edge setup instruction, see: https://cloud.ibm.com/docs/vmwaresolutions?topic=vmwaresolutions-shared_vcd-ops-guide#shared_vcd-ops-guide-create-network

Each vCloud Datacenter comes with 5 IBM Cloud public IP addresses which we can use for SNAT and DNAT translations in and out of the datacenter instance.  VMWare vCloud calls these `sub-allocated` addresses.
The sub-allocated address are available in IBM Cloud on the vCloud instance Resources page.
Gather the following information that you will need when configuring the ESG:
* Make a `list of the IPs and Sub-allocated IP Addresses` for the ESG.   
![Public IP](media/public_ip.jpg)


- Take an unused IP and set `cluster_public_ip` and for `public_bastion_ip`
- The Red Hat Activation key can be retrieved from this screen to populate `rhel_key`


- Your terraform.tfvars entries should look something like this:    
```
 cluster_public_ip  = "161.yyy.yy.yyy"

 initialization_info     = {
    public_bastion_ip = "161.xxx.xx.xxx"
    bastion_password = "OCP4All"
    internal_bastion_ip = "172.16.0.10"
    terraform_ocp_repo = "https://github.com/ibm-cloud-architecture/terraform-openshift4-vcd"
    rhel_key = "xxxxxxxxxxxxxxxxxxxxxx"
    machine_cidr = "172.16.0.1/24"
    network_name      = "ocpnet"
    static_start_address    = "172.16.0.150"
    static_end_address      = "172.16.0.220"
    }
```

#### Retrieve pull secret from Red Hat sites
Retrieve the [OpenShift Pull Secret](https://cloud.redhat.com/openshift/install/vsphere/user-provisioned) and place in a file on the Bastion Server. Default location is `~/.pull-secret`

## Perform Bastion install
Once you have finished editing your terraform.tfvars file you can execute the following commands. Terraform will now create the Bastion, install and configure all necessary software and perform all network customizations associated with the Bastion. The terraform.tfvars file will be copied to the Bastion server. The pull secret and additionalTrustBundle will be copied to the Bastion if they were specified in terraform.tfvars and are in the specified location on the Host machine. If you plan to create the pull secret and additionalTrustBundle on the Bastion directly and didn't put them on your Host, ignore the error messages about the copy failing.
```
terraform init
terraform -chdir=bastion-vm plan --var-file="../terraform.tfvars"
terraform -chdir=bastion-vm apply --var-file="../terraform.tfvars" --auto-approve
```

The result looks something like this:
```
null_resource.setup_bastion (local-exec): PLAY RECAP *********************************************************************
null_resource.setup_bastion (local-exec): 150.239.22.38              : ok=26   changed=26   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

null_resource.setup_bastion: Creation complete after 3m32s [id=1639642181061551613]

Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:

login_bastion = "Next Step login to Bastion via: ssh root@1xxx.xx.xx.38"
```
#### Login to Bastion
Use the generated command to login to the Bastion
`ssh root@1xxx.xx.xx.38`
The result should look somthing like this below. You can ignore the messages about registering the Red Hat VM with the activation key as this was done as part of the provisioning

```
To register the Red Hat VM with your RHEL activation key in IBM RHEL Capsule Server, you must enable VM access to connect to the IBM service network.  For more information, see Enabling VM access to IBM Cloud Services by using the private network (https://cloud.ibm.com/docs/services/vmwaresolutions?topic=vmware-solutions-shared_vcd-ops-guide#shared_vcd-ops-guide-enable-access).

Complete the following steps to register the Red Hat VM with your RHEL activation key. For more information about accessing instance details, see Viewing Virtual Data Center instances (https://cloud.ibm.com/docs/services/vmwaresolutions?topic=vmware-solutions-shared_managing#shared_managing-viewing).

1) From the IBM Cloud for VMware Solutions console, click the instance name in the VMware Solutions Shared instance table.

2) On the instance details page, locate and make note of the Red Hat activation key.

3) Run the following commands from the Red Hat VM:

rpm -ivh http://52.117.132.7/pub/katello-ca-consumer-latest.noarch.rpm

uuid=`uuidgen`

echo '{"dmi.system.uuid": "'$uuid'"}' > /etc/rhsm/facts/uuid_override.facts

subscription-manager register --org="customer" --activationkey="${activation_key}" --force
Where:
${activation_key} is the Red Hat activation key that is located on the instance details page.

Last login: Sat Mar  6 01:50:40 2021 from 24.34.132.100
[root@vm-rhel8 ~]#

```
You can look to make sure that your pull secret was copied:
```
[root@vm-rhel8 ~]# ls
airgap.crt  pull-secret
[root@vm-rhel8 ~]#

```
You can now go to the vcd directory. It is now placed in /opt/terraform. You will find your terraform.tfvars in the directory. You can inspect it to ensure that it is complete.
```
[root@vm-rhel8 ~]# cd /opt/terraform/
[root@vm-rhel8 terraform]# ls
bastion-vm      haproxy.conf  lb       media    output.tf  storage  terraform.tfvars          variables.tf  vm
csr-approve.sh  ignition      main.tf  network  README.md  temp     terraform.tfvars.example  versions.tf
[root@vm-rhel8 terraform]#

```
If your terraform.tfvars file is complete, you can run the commands to create your cluster. The FW, DNAT and /etc/hosts entries on the Bastion will now be created too.
```
terraform init
terraform apply --auto-approve
```

#### Client setup

On the **Client** that you will access the OCP Console, (your Mac, PC, etc.) add name resolution to direct console to the **Public IP** of the LoadBalancer in /etc/hosts on the client that will login to the Console UI.
  As an example:
```
  1.2.3.4 api.ocp44-myprefix.my.com
  1.2.3.4 api-int.ocp44-myprefix.my.com
  1.2.3.4 console-openshift-console.apps.ocp44-myprefix.my.com
  1.2.3.4 oauth-openshift.apps.ocp44-myprefix.my.com
```

**NOTE:** On a MAC, make sure that the permissions on your /etc/host file is correct.  
If it looks like this:   
`$ ls -l /etc/hosts
-rw-------  1 root  wheel  622  1 Feb 08:57 /etc/hosts`   

Change to this:  
`$ sudo chmod ugo+r /etc/hosts
$ ls -l /etc/hosts
-rw-r--r--  1 root  wheel  622  1 Feb 08:57 /etc/hosts`




#### terraform variables

| Variable                     | Description                                                  | Type | Default |
| ---------------------------- | ------------------------------------------------------------ | ---- | ------- |
| vcd_url               | url for the VDC api                    | string | https://daldir01.vmware-solutions.cloud.ibm.com/api |
| vcd_user                     | VCD username                                             | string | - |
| vcd_password             | VCD password                                             | string | - |
| vcd_vdc                      | VCD VDC name          | string | - |
| cluster_id                   | VCD Cluster where OpenShift will be deployed             | string | - |
| vcd_org   |  VCD Org from VCD Console screen |  string |   |
| rhcos_template | Name of CoreOS OVA template from prereq #2 | string | - |
| lb_template | VCD Network for OpenShift nodes                   | string | - |
| vcd_catalog   | Name of VCD Catalog containing your templates  | string  |  Public Catalog |
| vm_dns_addresses           | List of DNS servers to use for your OpenShift Nodes          | list   | 8.8.8.8, 8.8.4.4               |
|mac_address_prefix   |  The prefix used to create mac addresses for dhcp reservations. The last 2 digits are derived from the last 2 digits of the ip address of a given machine. The final octet of the ip address for the vm's should not be over 99. |  string |  00:50:56:01:30 |
| base_domain                | Base domain for your OpenShift Cluster                       | string | -                              |
| bootstrap_ip_address|IP Address for bootstrap node|string|-|
| control_plane_count          | Number of control plane VMs to create                        | string | 3                |
| control_plane_memory         | Memory, in MB, to allocate to control plane VMs              | string | 16384            |
| control_plane_num_cpus| Number of CPUs to allocate for control plane VMs             |string|4|
| control_disk  | size in MB   | string  |  - |
| compute_ip_addresses|List of IP addresses for your compute nodes|list|-|
| compute_count|Number of compute VMs to create|string|3|
| compute_memory|Memory, in MB, to allocate to compute VMs|string|8192|
| compute_num_cpus|Number of CPUs to allocate for compute VMs|string|3|
| compute_disk  | size in MB   | string  |  - |
| storage_ip_addresses|List of IP addresses for your storage nodes|list|`Empty`|
| storage_count|Number of storage VMs to create|string|0|
| storage_memory               | Memory, in MB to allocate to storage VMs                     | string | 65536            |
| storage_num_cpus             | Number of CPUs to allocate for storage VMs                   | string | 16               |
| storage_disk  | size in MB must be min of 2097152 to install OCS   | string  |  2097152 |
| lb_ip_address                | IP Address for LoadBalancer VM on same subnet as `machine_cidr` | string | -                |
| loadbalancer_lb_ip_address   | IP Address for LoadBalancer VM for secondary NIC on same subnet as `loadbalancer_lb_machine_cidr` | string | -                |
| loadbalancer_lb_machine_cidr | CIDR for your LoadBalancer CoreOS VMs in `subnet/mask` format | string | -                |
| openshift_pull_secret        | Path to your OpenShift [pull secret](https://cloud.redhat.com/openshift/install/vsphere/user-provisioned) | string |~/.pull-secret               |
| openshift_cluster_cidr       | CIDR for pods in the OpenShift SDN                           | string | 10.128.0.0/14    |
| openshift_service_cidr       | CIDR for services in the OpenShift SDN                       | string | 172.30.0.0/16    |
| openshift_host_prefix        | Controls the number of pods to allocate to each node from the `openshift_cluster_cidr` CIDR. For example, 23 would allocate 2^(32-23) 512 pods to each node. | string | 23               |
| create_loadbalancer_vm | Create the LoadBalancer VM and use it as a DNS server for your cluster.  If set to `false` you must provide a valid pre-configured LoadBalancer for your `api` and `*.apps` endpoints and DNS Zone for your `cluster_id`.`base_domain`. | bool | false |
| cluster_public_ip |Public IP address to be used for your OCP Cluster Console   |  string |   |
|create_vms_only   |  **Experimental** If you set this to true, running `terraform apply` will fail after bootstrap machine. Just run `terraform apply` again and it should complete sucessfully | bool  | false |
|**initialization_info object** |   |   |   |
|public_bastion_ip |  Choose 1 of the 5 Public ip's for ssh access to the Bastion.| String  |   |
| machine_cidr | CIDR for your CoreOS VMs in `subnet/mask` format.            | string | -                              |
|internal_bastion_ip   |  The internal ip for the Bastion. Must be assigned an ip address within your machine_addr range. (ex. 172.16.0.10) | string  |   |
|bastion_password   |  Initial Password for Bastion |  string |   |
|terraform_ocp_repo   |  The github repo to be deployed to the Bastion (usually https://github.com/ibm-cloud-architecture/terraform-openshift4-vcd) | string  |   |
| rhel_key  |  Red Hat Activation key used to register the Bastion.   |  string |   |
|network_name   |  The network name that will be used for your Bastion and OCP cluster (ex. ocpnet) | string  |   |
|static_start_address   |  The start of the reserved static ip range on your network. (ex. 172.16.0.150) | string  |   |
|static_end_address   |  The end of the reserved static ip range on your network (ex. 172.16.0.200) |  string |   |
|bastion_template   |  The vApp Template name to use for your Bastion (ex. RedHat-8-Template-Official ) |  string |   |
|**airgapped object** | (only necessary for airgapped install)  |   |   |
|  enabled | set to true for airgapped, false for regular install  |  bool |  false |
|ocp_ver_rel   | Full version and release loaded into your mirror (ex. 4.6.15)  | string  | -  |
|mirror_ip   |  ip address of the server hosting mirro | string  | -  |
| mirror_fqdn  | fqdn of the mirror host. Must match the name in the mirrors registry's cert  |  string | -  |
|  mirror_port | port of the mirror  |string   | -  |
|  mirror_repository |  name of repo in mirror containing OCP install images (currently should be set to `ocp4/openshift4` see RH Doc for details) |  string |  - |
|additional_trust_bundle   |  name of file containing cert for mirror | string  |  - |



#### Let OpenShift finish the installation:
Once terraform has completed sucessfully, you will see several pieces of information display. As sample is below:
```
Apply complete! Resources: 31 added, 0 changed, 0 destroyed.

Outputs:

export_kubeconfig = "export KUBECONFIG=/opt/terraform-openshift4-vcd/installer/testinf/auth/kubeconfig"
kubeadmin_user_info = [
  "kubeadmin",
  "6y9pL-RPoTq-SKADP-M6K5S",
]
openshift_console_url = "https://console-openshift-console.apps.testinf.cdastu.com"
public_ip = "xxx.yyy.zz.81"

```
Once you power on the machines it should take about 20 mins for your cluster to become active. To debug see **Debugging the OCP installation** below.

- power on all the VMs in the VAPP.

- The cluster userid and password are output from the `terraform apply` command.
- You can copy the export command generated to define KUBECONFIG. Alternately, you can get the info using the following methods:

  - You can also retrieve the password as follows:  
  cd to authentication directory:  
   `cd <clusternameDir>/auth`
    This directory contains both the cluster config and the kubeadmin password for UI login
 - export KUBECONFIG= clusternameDir/auth/kubeconfig   

    Example:   
   `export KUBECONFIG=/root/terraform-openshift-vmware/installer/stuocpvmshared1/auth/kubeconfig`
- If you want to watch the install, you can  
  `ssh -i installer/stuocpvmshared1/openshift_rsa core@<bootstrap ip>`  into the bootstrap console and watch the logs. Bootstrap will print the jounalctl command when you login: `journalctl -b -f -u release-image.service -u bootkube.service`. You will see lots of messages (including error messages) and in 15-20 minutes, you should see a message about the bootstrap service completing. Once this happens, exit the bootstrap node.

  You can now watch the OpenShift install progress.

`oc get nodes`
```
 NAME                                  STATUS   ROLES    AGE   VERSION
 master-00.ocp44-myprefix.my.com   Ready    master   16m   v1.17.1+6af3663
 master-01.ocp44-myprefix.my.com   Ready    master   16m   v1.17.1+6af3663
 master-02.ocp44-myprefix.my.com   Ready    master   16m   v1.17.1+6af36
```

Watch the cluster operators. Confirm the RH cluster operators are all 'Available'

`watch -n 5 oc get co`

```
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.22    True        False         False      79m
cloud-credential                           4.5.22    True        False         False      100m
cluster-autoscaler                         4.5.22    True        False         False      89m
config-operator                            4.5.22    True        False         False      90m
console                                    4.5.22    True        False         False      14m
csi-snapshot-controller                    4.5.22    True        False         False      18m
dns                                        4.5.22    True        False         False      96m
etcd                                       4.5.22    True        False         False      95m
image-registry                             4.5.22    True        False         False      91m
ingress                                    4.5.22    True        False         False      84m
insights                                   4.5.22    True        False         False      90m
kube-apiserver                             4.5.22    True        False         False      95m
kube-controller-manager                    4.5.22    True        False         False      95m
kube-scheduler                             4.5.22    True        False         False      92m
kube-storage-version-migrator              4.5.22    True        False         False      12m
machine-api                                4.5.22    True        False         False      90m
machine-approver                           4.5.22    True        False         False      94m
machine-config                             4.5.22    True        False         False      70m
marketplace                                4.5.22    True        False         False      13m
monitoring                                 4.5.22    True        False         False      16m
network                                    4.5.22    True        False         False      97m
node-tuning                                4.5.22    True        False         False      53m
openshift-apiserver                        4.5.22    True        False         False      12m
openshift-controller-manager               4.5.22    True        False         False      90m
openshift-samples                          4.5.22    True        False         False      53m
operator-lifecycle-manager                 4.5.22    True        False         False      96m
operator-lifecycle-manager-catalog         4.5.22    True        False         False      97m
operator-lifecycle-manager-packageserver   4.5.22    True        False         False      14m
service-ca                                 4.5.22    True        False         False      97m
storage                                    4.5.22    True        False         False      53m

```
#### Debugging the OCP installation

As noted above you power on all the VMs at once and magically OpenShift gets installed.  This section will explain enough of the magic so that you can figure out what happened when things go wrong. See Reference section below for deeper debug instructions

When the machines boot for the first time they each have special logic which runs scripts to further configure the VMs.  The first time they boot up using DHCP for network configuration.  When they boot, the "ignition" configuration is applied which switches the VMs to static IP, and then the machines reboot.  TODO how do you tell if this step failed?  Look at the VMs in VCD console to get clues about their network config?

Assuming Bootstrap VM boots correctly, the first thing it does is pull additional ignition data from the bastion HTTP server.  If you don't see a 200 get successful in the bastion HTTP server log within a few minutes of Bootstrap being powered on, that is a problem

Next Bootstrap installs an OCP control plane on itself, as well as an http server that it uses to help the master nodes get their own cluster setup.  You can ssh into boostrap (ssh core@172.16.0.20) and watch the logs.  Bootstrap will print the jounalctl command when you login: `journalctl -b -f -u release-image.service -u bootkube.service` . Look carefully at the logs. Typical problems at this stage are:
  * bad/missing pull secret
  * no internet access - double check your edge configuration, run typical ip network debug

The installer should have created Edge Firewall Rules for the OCP Cluster that look something like this:

![FW Example](./media/fwexample.png)

The DNAT rule for the OCP Console should look something like this:

![DNAT Example](./media/dnatexample.png)

## Configuration to enable OCP console login
- The console login information are now output as part of the terraform apply. Alternately, you can retrieve the information as follows:    


```
oc get routes console -n openshift-console

 NAME      HOST/PORT                                                  PATH   SERVICES   PORT    TERMINATION          WILDCARD
 console   console-openshift-console.apps.ocp44-myprefix.my.com          console    https   reencrypt/Redirect   None
```


- From a browser, connect to the "console host" from the `oc get routes` command with https. You will need to accept numerous security warnings as the deployment is using self-signed certificates.
- id is `kubeadmin` and password is in `<clusternameDir>/auth/kubeadmin-password`



## Optional Steps:
#### Use SSH Key rather than password authentication for Bastion login (optional)
The Bastion install process will create an ssh key and place it in  `~/.ssh/bastion_id`  

For login to Bastion, you can choose to use SSH Keys and disable password login:
  - edit /etc/ssh/sshd_config and set password authentication to no
  - add your bastion_id.pub ssh key to .ssh/authorised_keys on bastion

#### Move SSH to higher port (optional)
If you want to move your ssh port to a higher port to slow down hackers that are constantly looking to hack your server at port 22 then do the following:
1. Edit /etc/ssh/sshd_config and uncomment the line that currently reads #Port 22 and change to your desired port and save file. `systemctl stop sshd` `systemctl start sshd`
2. `semanage port -a -t ssh_port_t -p tcp <your new port number>`
3. Enter:   
`firewall-cmd --add-port=<your new port>/tcp --zone=public --permanent`  
`firewall-cmd --reload`
1. Update your edge FW rule for your Bastion with your new port. (replace port 22 with your new port)
### Add an NFS Server to provide Persistent storage.
-  [Here is an article on how to set this up](https://medium.com/faun/openshift-dynamic-nfs-persistent-volume-using-nfs-client-provisioner-fcbb8c9344e). Make the NFS Storage Class the default Storage Class   

  `oc patch storageclass managed-nfs-storage -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'`


### Create Openshift Container Storage Cluster for persistent storage.
 If you added storage nodes and want to add OCS instead of NFS for storage provisioner, Instructions here -> [Openshift Container Storage - Bare Metal Path](https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.6/html-single/deploying_openshift_container_storage_using_bare_metal_infrastructure/index#installing-openshift-container-storage-operator-using-the-operator-hub_rhocs)

 `oc patch storageclass ocs-storagecluster-cephfs -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'`

### Enable Registry
 - [Enable the OCP Image registry using your NFS Storage](https://docs.openshift.com/container-platform/4.5/registry/configuring_registry_storage/configuring-registry-storage-baremetal.html)
 - [Exposing the Registry](https://docs.openshift.com/container-platform/4.5/registry/securing-exposing-registry.html)

### Airgapped Installation
You will need a registry to store your images. A simple registry can be found [here](https://www.redhat.com/sysadmin/simple-container-registry).  
You will need to create your own mirror or use an existing mirror to do an airgapped install. Instructions to create a mirror for OpenShift 4.6 can be found [here](https://docs.openshift.com/container-platform/4.6/installing/install_config/installing-restricted-networks-preparations.html#installing-restricted-networks-preparations).

In order for the accept CSR code to work, you will have to  
`podman pull quay.io/openshift/origin-cli:latest` and push to your repository   
`podman push <mirror_fqdn>:<mirror_port>/openshift/origin-cli:latest`

You should edit your pull secret and remove the section that refers to `cloud.openshift.com`. This removes Telemetry and Health Reporting. If you don't do this before installation, you will get an error in the insights operator. After installation, go [here](https://docs.openshift.com/container-platform/4.6/support/remote_health_monitoring/opting-out-of-remote-health-reporting.html) for instructions to disable Telemetry Reporting.

You may receive an Alert stating `Cluster version operator has not retrieved updates in xh xxm 17s. Failure reason RemoteFailed . For more information refer to https://console-openshift-console.apps.<cluster_id>.<base_domain>.com/settings/cluster/` this is normal and can be ignored.

You will also need to mirror any operators that you will need and place them in the mirror. Instructions can be found [here](https://docs.openshift.com/container-platform/4.6/operators/admin/olm-restricted-networks.html)

You will need to follow the instructions carefully in order to setup imagesources for any operators that you want to install.

An example of the airgapped object:
```
airgapped = {
      enabled = true
      ocp_ver_rel = "4.6.15"
      mirror_ip = "172.16.0.10"
      mirror_fqdn = "bastion.airgapfull.cdastu.com"
      mirror_port = "5000"
      mirror_repository = "ocp4/openshift4"
      additionalTrustBundle = "~/airgap.crt"      
      }
```
### Deleting Cluster (and reinstalling)
If you want to delete your cluster, you should use `terraform destroy` as this will delete you cluster and remove all resources on VCD, including FW rules. If you manually delete your cluster via the VCD Console, remember to delete the FW rules and DNAT rules associated with your cluster or the reinstall may fail. The FW and DNAT rules should be tagged with your cluster name in the name and/or description fields.

You should keep your installer/cluster_id directory because the ssh keys that you need to access any of the vm's in your cluster are in this directory.

If you delete your cluster and want to reinstall, you will need to remove the installer/cluster_id directory or the install will likely fail when you start the VM's. You will see messages referencing x.509 certificate errors in the logs of the bootstrap and other servers if you forget to delete the directory.

`rm -rf installer/<your cluster id>`
