
# Deploy Kubernetes VMs with Terraform
 

具体环境如下：

| ESXi HOST | IP | Remark |
| -- | -- |--|
| esxi-host01 | 10.0.0.11 | #ESXi host01 |
| esxi-host02 | 10.0.0.12 | #ESXi host02 |
| esxi-host03 | 10.0.0.13 | #ESXi host03 |
| esxi-host04 | 10.0.0.14 | #ESXi host04 |
| esxi-host05 | 10.0.0.15 | #ESXi host05 |
| vCenter | 10.0.0.10 | #vCenter |
| Terraform | 10.0.0.5 | #Terraform Node |
| k8s-deploy | 10.0.0.9 | #Deploy Node |
| k8s-master01 | 10.0.0.21 | #Master Node |
| k8s-master02 | 10.0.0.22 | #Master Node |
| k8s-master03 | 10.0.0.23 | #Master Node |
| k8s-node01 | 10.0.0.31 | #Workder Node |
| k8s-node02 | 10.0.0.32 | #Workder Node |
| k8s-node03 | 10.0.0.33 | #Workder Node |
| k8s-node04 | 10.0.0.34 | #Workder Node |
| k8s-node05 | 10.0.0.35 | #Workder Node |

---

| OS | VERSION | Kernel |
|--|--|--|
| Ubuntu | 16.04 | 4.4.0-116 |

---
| Components | VERSION |
|--|--|
| kubernetes | 1.9.5 |
| etcd | 3.2.4 |
| calico | 2.6.8 |
| docker | 17.03-ce |

 

## 1. Terraform
  

### What is Terraform?

Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.

The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.

说白了，就是==Write, Plan, and create Infrastructure as Code==

---

  

首先我们来安装Terraform

  

### 1.1 Install GO 1.10

```bash
wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.10.1.linux-amd64.tar.gz
export PATH=/usr/local/go/bin:$PATH
export GOPATH=/root/go
export GOROOT=/usr/local/go
```

### 1.2 Install Terraform 0.11.7
```bash
wget https://releases.hashicorp.com/terraform/0.11.7/terraform_0.11.7_linux_amd64.zip
unzip terraform_0.11.7_linux_amd64.zip
cp terraform /usr/local/bin/
```


### 1.3 Terraform 文件

为了逻辑清晰，我将所有配置分成4个文件

```bash
root@terraform:~/terraform/lab-k8sz1-tf# ls
data_sources.tf k8s.tf terraform.tfvars variables.tf
```

`data_sources.tf`
存放vsphere已有的资源变量
```tcl
data "vsphere_datacenter" "datacenter" {
  name = "${var.vsphere_datacenter}"
}

data "vsphere_host" "hosts" {
  count         = "${length(var.esxi_hosts)}"
  name          = "${var.esxi_hosts[count.index]}"
  datacenter_id = "${data.vsphere_datacenter.datacenter.id}"
}

data "vsphere_resource_pool" "resource_pool" {
  name          = "${var.vsphere_resource_pool}"
  datacenter_id = "${data.vsphere_datacenter.datacenter.id}"
}

data "vsphere_virtual_machine" "template" {
  name          = "${var.vsphere_vm_template}"
  datacenter_id = "${data.vsphere_datacenter.datacenter.id}"
}

data "vsphere_datastore" "datastore" {
  name          = "${var.vsphere_datastore}"
  datacenter_id = "${data.vsphere_datacenter.datacenter.id}"
}

data "vsphere_network" "network" {
  name          = "${var.vsphere_port_group}"
  datacenter_id = "${data.vsphere_datacenter.datacenter.id}"
}

```

  

`k8s.tf`

存放具体需要创建的k8s的VM资源

```tcl

# Configure vSphere provider
provider "vsphere" {
        vsphere_server = "${var.vsphere_vcenter}"
        user = "${var.vsphere_user}"
        password = "${var.vsphere_password}"
        allow_unverified_ssl = "${var.vsphere_unverified_ssl}"
}

# Create a vSphere VM folder
resource "vsphere_folder" "vm_folder" {
        datacenter_id = "${data.vsphere_datacenter.datacenter.id}"
        type = "vm"
        path = "${var.vsphere_vm_folder}"
}

# Create a vSphere VM in the k8s-vms folder
resource "vsphere_virtual_machine" "k8s-masters" {
  count            = "${var.k8s_master_count}"
  name             = "${var.k8s_master_name}${count.index + 1}"
  resource_pool_id = "${data.vsphere_resource_pool.resource_pool.id}"
  datastore_id     = "${data.vsphere_datastore.datastore.id}"
  host_system_id = "${data.vsphere_host.hosts.*.id[count.index]}"
  folder = "${vsphere_folder.vm_folder.path}"

  num_cpus = "${var.k8s_master_cpu}"
  memory   = "${var.k8s_master_memory}"
  guest_id = "${data.vsphere_virtual_machine.template.guest_id}"
  enable_disk_uuid = "true"

  network_interface {
    network_id   = "${data.vsphere_network.network.id}"
  }

  disk {
    label = "disk0" 
    size = "${data.vsphere_virtual_machine.template.disks.0.size}"
  }

  clone {
    template_uuid = "${data.vsphere_virtual_machine.template.id}"

    customize {
      linux_options {
        host_name = "${var.k8s_master_name}${count.index + 1}"
        domain    = "${var.virtual_machine_domain}"
        time_zone = "${var.vsphere_time_zone}"
      }

      network_interface {
        ipv4_address = "${var.vsphere_ipv4_address}${21 + count.index}"
        ipv4_netmask = "${var.vsphere_ipv4_netmask}"
      }

      ipv4_gateway    = "${var.vsphere_ipv4_gateway}"
      dns_suffix_list = "${var.virtual_machine_search_domain}"
      dns_server_list = "${var.vsphere_dns_servers}"
    }
  }
}

resource "vsphere_virtual_machine" "k8s-nodes" {
  count            = "${var.k8s_node_count}"
  name             = "${var.k8s_node_name}${count.index + 1}"
  resource_pool_id = "${data.vsphere_resource_pool.resource_pool.id}"
  datastore_id     = "${data.vsphere_datastore.datastore.id}"
  host_system_id = "${data.vsphere_host.hosts.*.id[count.index]}"
  folder = "${vsphere_folder.vm_folder.path}"

  num_cpus = "${var.k8s_node_cpu}"
  memory   = "${var.k8s_node_memory}"
  guest_id = "${data.vsphere_virtual_machine.template.guest_id}"
  enable_disk_uuid = "true"

  network_interface {
    network_id   = "${data.vsphere_network.network.id}"
  }

  disk {
    label = "disk0" 
    size = "${data.vsphere_virtual_machine.template.disks.0.size}"
  }

  clone {
    template_uuid = "${data.vsphere_virtual_machine.template.id}"

    customize {
      linux_options {
        host_name = "${var.k8s_node_name}${count.index + 1}"
        domain    = "${var.virtual_machine_domain}"
        time_zone = "${var.vsphere_time_zone}"
      }

      network_interface {
        ipv4_address = "${var.vsphere_ipv4_address}${31 + count.index}"
        ipv4_netmask = "${var.vsphere_ipv4_netmask}"
      }

      ipv4_gateway    = "${var.vsphere_ipv4_gateway}"
      dns_suffix_list = "${var.virtual_machine_search_domain}"
      dns_server_list = "${var.vsphere_dns_servers}"
    }
  }
}

resource "vsphere_virtual_machine" "k8s-deploy" {
  #count            = "1"
  name             = "${var.k8s_deploy_name}"
  resource_pool_id = "${data.vsphere_resource_pool.resource_pool.id}"
  datastore_id     = "${data.vsphere_datastore.datastore.id}"
  #host_system_id = "${data.vsphere_host.hosts.*.id[count.index]}"
  folder = "${vsphere_folder.vm_folder.path}"

  num_cpus = "${var.k8s_deploy_cpu}"
  memory   = "${var.k8s_deploy_memory}"
  guest_id = "${data.vsphere_virtual_machine.template.guest_id}"
  #enable_disk_uuid = "true"

  network_interface {
    network_id   = "${data.vsphere_network.network.id}"
  }

  disk {
    label = "disk0" 
    size = "${data.vsphere_virtual_machine.template.disks.0.size}"
  }

  clone {
    template_uuid = "${data.vsphere_virtual_machine.template.id}"

    customize {
      linux_options {
        host_name = "${var.k8s_deploy_name}"
        domain    = "${var.virtual_machine_domain}"
        time_zone = "${var.vsphere_time_zone}"
      }

      network_interface {
        ipv4_address = "${var.vsphere_ipv4_address}9"
        ipv4_netmask = "${var.vsphere_ipv4_netmask}"
      }

      ipv4_gateway    = "${var.vsphere_ipv4_gateway}"
      dns_suffix_list = "${var.virtual_machine_search_domain}"
      dns_server_list = "${var.vsphere_dns_servers}"
    }
  }
}

```
  

`variables.tf`

用来存放变量

```tcl

# vCenter connection

variable "vsphere_user" {
        description = "vSphere user name"
}

variable "vsphere_password" {
        description = "vSphere password"
}

variable "vsphere_vcenter" {
        description = "vCenter server FQDN or IP"
}

variable "vsphere_unverified_ssl" {
        description = "Is the vCenter using a self signed certificate (true/false)"
}

# VM specifications

variable "vsphere_datacenter" {
        description = "In which datacenter the VM will be deployed"
}

#variable "vsphere_vm_name" {
#        description = "What is the name of the VM"
#}

variable "vsphere_vm_template" {
        description = "Where is the VM template located"
}

variable "vsphere_vm_folder" {
        description = "In which folder the VM will be store"
}

variable "vsphere_cluster" {
        description = "In which cluster the VM will be deployed"
}

variable "vsphere_resource_pool" {
        description = "Resource Pool"
}

variable "vsphere_vcpu_number" {
        description = "How many vCPU will be assigned to the VM (default: 1)"
        default = "1"
}

variable "vsphere_memory_size" {
        description = "How much RAM will be assigned to the VM (default: 1024)"
        default = "1024"
}

variable "vsphere_datastore" {
        description = "What is the name of the VM datastore"
}

variable "vsphere_port_group" {
        description = "In which port group the VM NIC will be configured (default: VM Network)"
        default = "VM Network"
}

variable "vsphere_ipv4_address" {
        description = "What is the IPv4 address of the VM"
}

variable "vsphere_ipv4_netmask" {
        description = "What is the IPv4 netmask of the VM (default: 24)"
        default = "24"
}

variable "vsphere_ipv4_gateway" {
        description = "What is the IPv4 gateway of the VM"
}

variable "vsphere_dns_servers" {
        description = "DNS server list"
        type = "list"
}

variable "virtual_machine_search_domain" {
        description = "DNS search domain list"
        type = "list"
}

variable "vsphere_time_zone" {
        description = "What is the timezone of the VM (default: UTC)"
}

variable "virtual_machine_domain" {
        description = "virtual machine domain"
        default = "homeoffice.cn.wal-mart.com"
}

variable "k8s_master_name" {
        description = "k8s master name"
}

variable "k8s_master_cpu" {
        description = "k8s master cpu"
}

variable "k8s_master_memory" {
        description = "k8s master memory"
}

variable "k8s_master_count" {
        description = "k8s master count"
}

variable "k8s_node_name" {
        description = "k8s node name"
}

variable "k8s_node_cpu" {
        description = "k8s node cpu"
}

variable "k8s_node_memory" {
        description = "k8s node memory"
}

variable "k8s_node_count" {
        description = "k8s node count"
}

variable "k8s_deploy_name" {
        description = "k8s deployment node name"
}

variable "k8s_deploy_cpu" {
        description = "k8s deploy cpu"
}

variable "k8s_deploy_memory" {
        description = "k8s deploy memory"
}

variable "esxi_hosts" {
        description = "ESXi HOSTs"
        type = "map"
}
```

  

`terraform.tfvars`

用来存放所有变量的实例，基本上我们只需要修改这个文件中的内容

```tcl

# vCenter connection

vsphere_vcenter = "10.0.0.10"

vsphere_user = "administrator@vsphere.local"

vsphere_password = "CHANGEME"

vsphere_unverified_ssl = "true"

  

# VM specifications

vsphere_datacenter = "DataCenter"

vsphere_resource_pool = "k8s-cluster-RP"

vsphere_vm_folder = "k8s-vms"

vsphere_vm_template = "ubuntu16.04-template"

vsphere_cluster = "k8s-Cluster"

vsphere_datastore = "vsanDatastore"

vsphere_port_group = "VM Network"

vsphere_ipv4_address = "10.0.0."

vsphere_ipv4_netmask = "24"

vsphere_ipv4_gateway = "10.0.0.1"

vsphere_dns_servers = ["114.114.114.114","223.5.5.5"]

  

vsphere_time_zone = "Asia/Shanghai"

  

k8s_deploy_name = "k8s-deploy"

k8s_deploy_cpu = "2"

k8s_deploy_memory = "4096"

#k8s deploy node IP is $vsphere_ipv4_address.9

  

k8s_master_name = "k8s-master0"

k8s_master_cpu = "2"

k8s_master_memory = "8196"

k8s_master_count = "3"

#k8s master node IP is starts with $vsphere_ipv4_address.21

#Don't deploy master node count more than 10

  

k8s_node_name = "k8s-node0"

k8s_node_cpu = "8"

k8s_node_memory = "16384"

k8s_node_count = "5"

#k8s master node IP is starts with $vsphere_ipv4_address.31

#Don't deploy worker node count more than 10

  
  

esxi_hosts = {

"0" = "10.0.0.11"

"1" = "10.0.0.12"

"2" = "10.0.0.13"

"3" = "10.0.0.14"

"4" = "10.0.0.15"

}

```

  

### 1.4 在vCenter里面创建预置的资源

  

1.4.1 创建Resource Pool，并且名字设置为`k8s-cluster-RP`

  

1.4.2 创建VM Template，我们可以使用ubuntu16.04的ISO安装完毕之后，就可以直接关机并且转换为tempalte，并且名字设置为`ubuntu16.04-template`

这步具体就不演示了，比较简单。

  
  

### 1.5 Terraform GOGOGO

  

```shell

root@terraform:~/terraform/# terraform init

  

Initializing provider plugins...

- Checking for available provider plugins on https://releases.hashicorp.com...

- Downloading plugin for provider "vsphere" (1.4.1)...

  

The following providers do not have any version constraints in configuration,

so the latest version was installed.

  

To prevent automatic upgrades to new major versions that may contain breaking

changes, it is recommended to add version = "..." constraints to the

corresponding provider blocks in configuration, with the constraint strings

suggested below.

  

* provider.vsphere: version = "~> 1.4"

  

Terraform has been successfully initialized!

  

You may now begin working with Terraform. Try running "terraform plan" to see

any changes that are required for your infrastructure. All Terraform commands

should now work.

  

If you ever set or change modules or backend configuration for Terraform,

rerun this command to reinitialize your working directory. If you forget, other

commands will detect it and remind you to do so if necessary.

root@terraform:~/terraform/#

```

  

```shell

root@terraform:~/terraform/# terraform plan

  

#如果没有配置错误，这里会出现每个创建的资源的详细配置项

  

```

  
  

```shell

  

root@terraform:~/terraform/prod-k8sz1-tf# terraform apply

vsphere_virtual_machine.k8s-masters.1: Still creating... (1m20s elapsed)

vsphere_virtual_machine.k8s-nodes.4: Still creating... (1m20s elapsed)

vsphere_virtual_machine.k8s-deploy: Creation complete after 1m29s (ID: 42187d77-86a2-ef21-f7aa-b443576029ae)

vsphere_virtual_machine.k8s-masters[2]: Creation complete after 1m29s (ID: 4218bd31-cbb1-b181-2699-6af3f00563b5)

vsphere_virtual_machine.k8s-masters.0: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-nodes.2: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-nodes.1: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-nodes.0: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-nodes.3: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-masters.1: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-nodes.4: Still creating... (1m30s elapsed)

vsphere_virtual_machine.k8s-nodes[4]: Creation complete after 1m34s (ID: 4218e1b4-ea21-ca96-1e78-6d6c30b655e9)

vsphere_virtual_machine.k8s-nodes[0]: Creation complete after 1m35s (ID: 42182c7f-cffe-4109-33e9-0df8cc113261)

vsphere_virtual_machine.k8s-masters[0]: Creation complete after 1m36s (ID: 4218dbe8-a641-7db6-982a-5b215cd0571f)

vsphere_virtual_machine.k8s-masters[1]: Creation complete after 1m37s (ID: 42185559-a55f-7aab-37c3-6706494c061d)

vsphere_virtual_machine.k8s-nodes[1]: Creation complete after 1m37s (ID: 42184531-0cbc-6ed5-e025-98b9ca6dc312)

vsphere_virtual_machine.k8s-nodes[3]: Creation complete after 1m39s (ID: 4218ed5b-f131-d96e-24a7-4d9b0e5b9c89)

vsphere_virtual_machine.k8s-nodes[2]: Creation complete after 1m39s (ID: 421855f2-6d31-1b7f-18d1-5870a0548bac)

  

Apply complete! Resources: 10 added, 0 changed, 0 destroyed.

root@terraform:~/terraform/prod-k8sz1-tf#

```

  

执行完毕之后，就能看到vCenter里面已经创建好所有的k8s集群的主机

  

`terraform show`能够看到创建的所有资源的详细信息
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMwMzkxMDM5MSwxMDcxNzUxNzE1LDIwMD
E4Njk3MDAsODU4MDQ3NzIsLTE2NTgxMzI4NTYsMjA1MTI2NDY5
NCwtMTg2ODc1MDY5NywtOTc0MTYzODU4XX0=
-->