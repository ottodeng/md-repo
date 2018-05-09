# Deploy Kubernetes with Terraform & Kubespray(ansible)

具体环境如下：

| ESXi HOST | IP | Remark | 
|--|--|--|
| esxi-host01 | 10.0.0.11 | #ESXi host01 |
| esxi-host02 | 10.0.0.12 | #ESXi host02 |
| esxi-host03 | 10.0.0.13 | #ESXi host03 |
| esxi-host04 | 10.0.0.14 | #ESXi host04 |
| esxi-host05 | 10.0.0.15 | #ESXi host05 |
| vCenter | 10.0.0.10 | #vCenter |
| k8s-deploy | 10.0.0.9 | #Deploy Node |
| k8s-master01 | 10.0.0.21 | #Master Node |
| k8s-master02 | 10.0.0.22 | #Master Node |
| k8s-master03 | 10.0.0.23 | #Master Node |
| k8s-node01 | 10.0.0.31 | #Workder Node |
| k8s-node02 | 10.0.0.32 | #Workder Node |
| k8s-node03 | 10.0.0.33 | #Workder Node |
| k8s-node04 | 10.0.0.34 | #Workder Node |
| k8s-node05 | 10.0.0.35 | #Workder Node |


## 1. Terraform

### What is Terraform?

Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.

The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.

说白了，就是“**Write, Plan, and create Infrastructure as Code**”

---

首先我们来安装Terraform

### 1.1 Install GO 1.10

    wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz tar -C
    /usr/local -xzf go1.10.1.linux-amd64.tar.gz 
    
    export PATH=/usr/local/go/bin:$PATH export GOPATH=/root/go export
    GOROOT=/usr/local/go

### 1.2 Install Terraform 0.11.7

    wget https://releases.hashicorp.com/terraform/0.11.7/terraform_0.11.7_linux_amd64.zip
    unzip terraform_0.11.7_linux_amd64.zip 
    cp terraform /usr/local/bin/
   


<!--stackedit_data:
eyJoaXN0b3J5IjpbODU4MDQ3NzIsLTE2NTgxMzI4NTYsMjA1MT
I2NDY5NCwtMTg2ODc1MDY5NywtOTc0MTYzODU4XX0=
-->