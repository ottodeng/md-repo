# Deploy Kubernetes with Terraform & Kubespray(ansible)


## 1. Terraform

### What is Terraform?

Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.

The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.

说白了，就是“**Write, Plan, and create Infrastructure as Code**”

---

首先我们来安装Terraform

1.1 Install GO 1.10

wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.10.1.linux-amd64.tar.gz 

export PATH=/usr/local/go/bin:$PATH
export GOPATH=/root/go
export GOROOT=/usr/local/go
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg2MTEzNzI2MCwtOTc0MTYzODU4XX0=
-->