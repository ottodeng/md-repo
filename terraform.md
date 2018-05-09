---


---

<h1 id="deploy-kubernetes-vms-with-terraform">Deploy Kubernetes VMs with Terraform</h1>
<p>具体环境如下：</p>
<p>| ESXi HOST | IP | Remark |</p>
<p>|–|--|–|</p>
<p>| esxi-host01 | 10.0.0.11 | #ESXi host01 |</p>
<p>| esxi-host02 | 10.0.0.12 | #ESXi host02 |</p>
<p>| esxi-host03 | 10.0.0.13 | #ESXi host03 |</p>
<p>| esxi-host04 | 10.0.0.14 | #ESXi host04 |</p>
<p>| esxi-host05 | 10.0.0.15 | #ESXi host05 |</p>
<p>| vCenter | 10.0.0.10 | #vCenter |</p>
<p>| Terraform | 10.0.0.5 | #Terraform Node |</p>
<p>| k8s-deploy | 10.0.0.9 | #Deploy Node |</p>
<p>| k8s-master01 | 10.0.0.21 | #Master Node |</p>
<p>| k8s-master02 | 10.0.0.22 | #Master Node |</p>
<p>| k8s-master03 | 10.0.0.23 | #Master Node |</p>
<p>| k8s-node01 | 10.0.0.31 | #Workder Node |</p>
<p>| k8s-node02 | 10.0.0.32 | #Workder Node |</p>
<p>| k8s-node03 | 10.0.0.33 | #Workder Node |</p>
<p>| k8s-node04 | 10.0.0.34 | #Workder Node |</p>
<p>| k8s-node05 | 10.0.0.35 | #Workder Node |</p>
<hr>
<p>| OS | VERSION | Kernel |</p>
<p>|–|--|–|</p>
<p>| Ubuntu | 16.04 | 4.4.0-116 |</p>
<hr>
<p>| Components | VERSION |</p>
<p>|–|--|</p>
<p>| kubernetes | 1.9.5 |</p>
<p>| etcd | 3.2.4 |</p>
<p>| calico | 2.6.8 |</p>
<p>| docker | 17.03-ce |</p>
<h2 id="terraform">1. Terraform</h2>
<h3 id="what-is-terraform">What is Terraform?</h3>
<p>Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.</p>
<p>Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.</p>
<p>The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.</p>
<p>说白了，就是<mark>Write, Plan, and create Infrastructure as Code</mark></p>
<hr>
<p>首先我们来安装Terraform</p>
<h3 id="install-go-1.10">1.1 Install GO 1.10</h3>
<pre class=" language-bash"><code class="prism  language-bash">
<span class="token function">wget</span> https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz

<span class="token function">tar</span> -C /usr/local -xzf go1.10.1.linux-amd64.tar.gz

<span class="token function">export</span> PATH<span class="token operator">=</span>/usr/local/go/bin:<span class="token variable">$PATH</span>

<span class="token function">export</span> GOPATH<span class="token operator">=</span>/root/go

<span class="token function">export</span> GOROOT<span class="token operator">=</span>/usr/local/go

</code></pre>
<h3 id="install-terraform-0.11.7">1.2 Install Terraform 0.11.7</h3>
<pre class=" language-bash"><code class="prism  language-bash">
<span class="token function">wget</span> https://releases.hashicorp.com/terraform/0.11.7/terraform_0.11.7_linux_amd64.zip

unzip terraform_0.11.7_linux_amd64.zip

<span class="token function">cp</span> terraform /usr/local/bin/

</code></pre>
<h3 id="terraform-文件">1.3 Terraform 文件</h3>
<p>为了逻辑清晰，我将所有配置分成4个文件</p>
<pre class=" language-bash"><code class="prism  language-bash">
root@terraform:~/terraform/lab-k8sz1-tf<span class="token comment"># ls</span>

data_sources.tf k8s.tf terraform.tfvars variables.tf

</code></pre>
<p><code>data_sources.tf</code></p>
<p>存放vsphere已有的资源变量</p>
<pre class=" language-tcl"><code class="prism  language-tcl">
data <span class="token string">"vsphere_datacenter"</span> <span class="token string">"datacenter"</span> <span class="token punctuation">{</span>

name = <span class="token string">"${var.vsphere_datacenter}"</span>

<span class="token punctuation">}</span>

  

data <span class="token string">"vsphere_host"</span> <span class="token string">"hosts"</span> <span class="token punctuation">{</span>

count = <span class="token string">"${length(var.esxi_hosts)}"</span>

name = <span class="token string">"${var.esxi_hosts[count.index]}"</span>

datacenter_id = <span class="token string">"${data.vsphere_datacenter.datacenter.id}"</span>

<span class="token punctuation">}</span>

  

data <span class="token string">"vsphere_resource_pool"</span> <span class="token string">"resource_pool"</span> <span class="token punctuation">{</span>

name = <span class="token string">"${var.vsphere_resource_pool}"</span>

datacenter_id = <span class="token string">"${data.vsphere_datacenter.datacenter.id}"</span>

<span class="token punctuation">}</span>

  

data <span class="token string">"vsphere_virtual_machine"</span> <span class="token string">"template"</span> <span class="token punctuation">{</span>

name = <span class="token string">"${var.vsphere_vm_template}"</span>

datacenter_id = <span class="token string">"${data.vsphere_datacenter.datacenter.id}"</span>

<span class="token punctuation">}</span>

  

data <span class="token string">"vsphere_datastore"</span> <span class="token string">"datastore"</span> <span class="token punctuation">{</span>

name = <span class="token string">"${var.vsphere_datastore}"</span>

datacenter_id = <span class="token string">"${data.vsphere_datacenter.datacenter.id}"</span>

<span class="token punctuation">}</span>

  
  

data <span class="token string">"vsphere_network"</span> <span class="token string">"network"</span> <span class="token punctuation">{</span>

name = <span class="token string">"${var.vsphere_port_group}"</span>

datacenter_id = <span class="token string">"${data.vsphere_datacenter.datacenter.id}"</span>

<span class="token punctuation">}</span>

</code></pre>
<p><code>k8s.tf</code></p>
<p>存放具体需要创建的k8s的VM资源</p>
<pre class=" language-tcl"><code class="prism  language-tcl">
<span class="token comment"># Configure vSphere provider</span>

provider <span class="token string">"vsphere"</span> <span class="token punctuation">{</span>

vsphere_server = <span class="token string">"${var.vsphere_vcenter}"</span>

user = <span class="token string">"${var.vsphere_user}"</span>

password = <span class="token string">"${var.vsphere_password}"</span>

allow_unverified_ssl = <span class="token string">"${var.vsphere_unverified_ssl}"</span>

<span class="token punctuation">}</span>

  

<span class="token comment"># Create a vSphere VM folder</span>

resource <span class="token string">"vsphere_folder"</span> <span class="token string">"vm_folder"</span> <span class="token punctuation">{</span>

datacenter_id = <span class="token string">"${data.vsphere_datacenter.datacenter.id}"</span>

type = <span class="token string">"vm"</span>

path = <span class="token string">"${var.vsphere_vm_folder}"</span>

<span class="token punctuation">}</span>

  
  

<span class="token comment"># Create a vSphere VM in the k8s-vms folder</span>

resource <span class="token string">"vsphere_virtual_machine"</span> <span class="token string">"k8s-masters"</span> <span class="token punctuation">{</span>

count = <span class="token string">"${var.k8s_master_count}"</span>

name = <span class="token string">"${var.k8s_master_name}${count.index + 1}"</span>

resource_pool_id = <span class="token string">"${data.vsphere_resource_pool.resource_pool.id}"</span>

datastore_id = <span class="token string">"${data.vsphere_datastore.datastore.id}"</span>

host_system_id = <span class="token string">"${data.vsphere_host.hosts.*.id[count.index]}"</span>

folder = <span class="token string">"${vsphere_folder.vm_folder.path}"</span>

  

num_cpus = <span class="token string">"${var.k8s_master_cpu}"</span>

<span class="token keyword">memory</span> = <span class="token string">"${var.k8s_master_memory}"</span>

guest_id = <span class="token string">"${data.vsphere_virtual_machine.template.guest_id}"</span>

enable_disk_uuid = <span class="token string">"true"</span>

  
  

network_interface <span class="token punctuation">{</span>

network_id = <span class="token string">"${data.vsphere_network.network.id}"</span>

<span class="token punctuation">}</span>

  

disk <span class="token punctuation">{</span>

label = <span class="token string">"disk0"</span>

size = <span class="token string">"${data.vsphere_virtual_machine.template.disks.0.size}"</span>

<span class="token punctuation">}</span>

  

clone <span class="token punctuation">{</span>

template_uuid = <span class="token string">"${data.vsphere_virtual_machine.template.id}"</span>

  

customize <span class="token punctuation">{</span>

linux_options <span class="token punctuation">{</span>

host_name = <span class="token string">"${var.k8s_master_name}${count.index + 1}"</span>

domain = <span class="token string">"${var.virtual_machine_domain}"</span>

time_zone = <span class="token string">"${var.vsphere_time_zone}"</span>

<span class="token punctuation">}</span>

  

network_interface <span class="token punctuation">{</span>

ipv4_address = <span class="token string">"${var.vsphere_ipv4_address}${21 + count.index}"</span>

ipv4_netmask = <span class="token string">"${var.vsphere_ipv4_netmask}"</span>

<span class="token punctuation">}</span>

  

ipv4_gateway = <span class="token string">"${var.vsphere_ipv4_gateway}"</span>

dns_suffix_list = <span class="token string">"${var.virtual_machine_search_domain}"</span>

dns_server_list = <span class="token string">"${var.vsphere_dns_servers}"</span>

<span class="token punctuation">}</span>

<span class="token punctuation">}</span>

<span class="token punctuation">}</span>

  

resource <span class="token string">"vsphere_virtual_machine"</span> <span class="token string">"k8s-nodes"</span> <span class="token punctuation">{</span>

count = <span class="token string">"${var.k8s_node_count}"</span>

name = <span class="token string">"${var.k8s_node_name}${count.index + 1}"</span>

resource_pool_id = <span class="token string">"${data.vsphere_resource_pool.resource_pool.id}"</span>

datastore_id = <span class="token string">"${data.vsphere_datastore.datastore.id}"</span>

host_system_id = <span class="token string">"${data.vsphere_host.hosts.*.id[count.index]}"</span>

folder = <span class="token string">"${vsphere_folder.vm_folder.path}"</span>

  

num_cpus = <span class="token string">"${var.k8s_node_cpu}"</span>

<span class="token keyword">memory</span> = <span class="token string">"${var.k8s_node_memory}"</span>

guest_id = <span class="token string">"${data.vsphere_virtual_machine.template.guest_id}"</span>

enable_disk_uuid = <span class="token string">"true"</span>

  
  

network_interface <span class="token punctuation">{</span>

network_id = <span class="token string">"${data.vsphere_network.network.id}"</span>

<span class="token punctuation">}</span>

  

disk <span class="token punctuation">{</span>

label = <span class="token string">"disk0"</span>

size = <span class="token string">"${data.vsphere_virtual_machine.template.disks.0.size}"</span>

<span class="token punctuation">}</span>

  

clone <span class="token punctuation">{</span>

template_uuid = <span class="token string">"${data.vsphere_virtual_machine.template.id}"</span>

  

customize <span class="token punctuation">{</span>

linux_options <span class="token punctuation">{</span>

host_name = <span class="token string">"${var.k8s_node_name}${count.index + 1}"</span>

domain = <span class="token string">"${var.virtual_machine_domain}"</span>

time_zone = <span class="token string">"${var.vsphere_time_zone}"</span>

<span class="token punctuation">}</span>

  

network_interface <span class="token punctuation">{</span>

ipv4_address = <span class="token string">"${var.vsphere_ipv4_address}${31 + count.index}"</span>

ipv4_netmask = <span class="token string">"${var.vsphere_ipv4_netmask}"</span>

<span class="token punctuation">}</span>

  

ipv4_gateway = <span class="token string">"${var.vsphere_ipv4_gateway}"</span>

dns_suffix_list = <span class="token string">"${var.virtual_machine_search_domain}"</span>

dns_server_list = <span class="token string">"${var.vsphere_dns_servers}"</span>

<span class="token punctuation">}</span>

<span class="token punctuation">}</span>

<span class="token punctuation">}</span>

  

resource <span class="token string">"vsphere_virtual_machine"</span> <span class="token string">"k8s-deploy"</span> <span class="token punctuation">{</span>

<span class="token comment">#count = "1"</span>

name = <span class="token string">"${var.k8s_deploy_name}"</span>

resource_pool_id = <span class="token string">"${data.vsphere_resource_pool.resource_pool.id}"</span>

datastore_id = <span class="token string">"${data.vsphere_datastore.datastore.id}"</span>

<span class="token comment">#host_system_id = "${data.vsphere_host.hosts.*.id[count.index]}"</span>

folder = <span class="token string">"${vsphere_folder.vm_folder.path}"</span>

  

num_cpus = <span class="token string">"${var.k8s_deploy_cpu}"</span>

<span class="token keyword">memory</span> = <span class="token string">"${var.k8s_deploy_memory}"</span>

guest_id = <span class="token string">"${data.vsphere_virtual_machine.template.guest_id}"</span>

<span class="token comment">#enable_disk_uuid = "true"</span>

  
  

network_interface <span class="token punctuation">{</span>

network_id = <span class="token string">"${data.vsphere_network.network.id}"</span>

<span class="token punctuation">}</span>

  

disk <span class="token punctuation">{</span>

label = <span class="token string">"disk0"</span>

size = <span class="token string">"${data.vsphere_virtual_machine.template.disks.0.size}"</span>

<span class="token punctuation">}</span>

  

clone <span class="token punctuation">{</span>

template_uuid = <span class="token string">"${data.vsphere_virtual_machine.template.id}"</span>

  

customize <span class="token punctuation">{</span>

linux_options <span class="token punctuation">{</span>

host_name = <span class="token string">"${var.k8s_deploy_name}"</span>

domain = <span class="token string">"${var.virtual_machine_domain}"</span>

time_zone = <span class="token string">"${var.vsphere_time_zone}"</span>

<span class="token punctuation">}</span>

  

network_interface <span class="token punctuation">{</span>

ipv4_address = <span class="token string">"${var.vsphere_ipv4_address}9"</span>

ipv4_netmask = <span class="token string">"${var.vsphere_ipv4_netmask}"</span>

<span class="token punctuation">}</span>

  

ipv4_gateway = <span class="token string">"${var.vsphere_ipv4_gateway}"</span>

dns_suffix_list = <span class="token string">"${var.virtual_machine_search_domain}"</span>

dns_server_list = <span class="token string">"${var.vsphere_dns_servers}"</span>

<span class="token punctuation">}</span>

<span class="token punctuation">}</span>

<span class="token punctuation">}</span>

</code></pre>
<p><code>variables.tf</code></p>
<p>用来存放变量</p>
<pre class=" language-tcl"><code class="prism  language-tcl">
<span class="token comment"># vCenter connection</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_user"</span> <span class="token punctuation">{</span>

description = <span class="token string">"vSphere user name"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_password"</span> <span class="token punctuation">{</span>

description = <span class="token string">"vSphere password"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_vcenter"</span> <span class="token punctuation">{</span>

description = <span class="token string">"vCenter server FQDN or IP"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_unverified_ssl"</span> <span class="token punctuation">{</span>

description = <span class="token string">"Is the vCenter using a self signed certificate (true/false)"</span>

<span class="token punctuation">}</span>

  

<span class="token comment"># VM specifications</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_datacenter"</span> <span class="token punctuation">{</span>

description = <span class="token string">"In which datacenter the VM will be deployed"</span>

<span class="token punctuation">}</span>

  

<span class="token comment">#variable "vsphere_vm_name" {</span>

<span class="token comment"># description = "What is the name of the VM"</span>

<span class="token comment">#}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_vm_template"</span> <span class="token punctuation">{</span>

description = <span class="token string">"Where is the VM template located"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_vm_folder"</span> <span class="token punctuation">{</span>

description = <span class="token string">"In which folder the VM will be store"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_cluster"</span> <span class="token punctuation">{</span>

description = <span class="token string">"In which cluster the VM will be deployed"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_resource_pool"</span> <span class="token punctuation">{</span>

description = <span class="token string">"Resource Pool"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_vcpu_number"</span> <span class="token punctuation">{</span>

description = <span class="token string">"How many vCPU will be assigned to the VM (default: 1)"</span>

default = <span class="token string">"1"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_memory_size"</span> <span class="token punctuation">{</span>

description = <span class="token string">"How much RAM will be assigned to the VM (default: 1024)"</span>

default = <span class="token string">"1024"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_datastore"</span> <span class="token punctuation">{</span>

description = <span class="token string">"What is the name of the VM datastore"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_port_group"</span> <span class="token punctuation">{</span>

description = <span class="token string">"In which port group the VM NIC will be configured (default: VM Network)"</span>

default = <span class="token string">"VM Network"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_ipv4_address"</span> <span class="token punctuation">{</span>

description = <span class="token string">"What is the IPv4 address of the VM"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_ipv4_netmask"</span> <span class="token punctuation">{</span>

description = <span class="token string">"What is the IPv4 netmask of the VM (default: 24)"</span>

default = <span class="token string">"24"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_ipv4_gateway"</span> <span class="token punctuation">{</span>

description = <span class="token string">"What is the IPv4 gateway of the VM"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_dns_servers"</span> <span class="token punctuation">{</span>

description = <span class="token string">"DNS server list"</span>

type = <span class="token string">"list"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"virtual_machine_search_domain"</span> <span class="token punctuation">{</span>

description = <span class="token string">"DNS search domain list"</span>

type = <span class="token string">"list"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"vsphere_time_zone"</span> <span class="token punctuation">{</span>

description = <span class="token string">"What is the timezone of the VM (default: UTC)"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"virtual_machine_domain"</span> <span class="token punctuation">{</span>

description = <span class="token string">"virtual machine domain"</span>

default = <span class="token string">"homeoffice.cn.wal-mart.com"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_master_name"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s master name"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_master_cpu"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s master cpu"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_master_memory"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s master memory"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_master_count"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s master count"</span>

<span class="token punctuation">}</span>

  
  

<span class="token scope constant">variable</span> <span class="token string">"k8s_node_name"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s node name"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_node_cpu"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s node cpu"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_node_memory"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s node memory"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_node_count"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s node count"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_deploy_name"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s deployment node name"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_deploy_cpu"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s deploy cpu"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"k8s_deploy_memory"</span> <span class="token punctuation">{</span>

description = <span class="token string">"k8s deploy memory"</span>

<span class="token punctuation">}</span>

  

<span class="token scope constant">variable</span> <span class="token string">"esxi_hosts"</span> <span class="token punctuation">{</span>

description = <span class="token string">"ESXi HOSTs"</span>

type = <span class="token string">"map"</span>

<span class="token punctuation">}</span>

</code></pre>
<p><code>terraform.tfvars</code></p>
<p>用来存放所有变量的实例，基本上我们只需要修改这个文件中的内容</p>
<pre class=" language-tcl"><code class="prism  language-tcl">
<span class="token comment"># vCenter connection</span>

vsphere_vcenter = <span class="token string">"10.0.0.10"</span>

vsphere_user = <span class="token string">"administrator@vsphere.local"</span>

vsphere_password = <span class="token string">"CHANGEME"</span>

vsphere_unverified_ssl = <span class="token string">"true"</span>

  

<span class="token comment"># VM specifications</span>

vsphere_datacenter = <span class="token string">"DataCenter"</span>

vsphere_resource_pool = <span class="token string">"k8s-cluster-RP"</span>

vsphere_vm_folder = <span class="token string">"k8s-vms"</span>

vsphere_vm_template = <span class="token string">"ubuntu16.04-template"</span>

vsphere_cluster = <span class="token string">"k8s-Cluster"</span>

vsphere_datastore = <span class="token string">"vsanDatastore"</span>

vsphere_port_group = <span class="token string">"VM Network"</span>

vsphere_ipv4_address = <span class="token string">"10.0.0."</span>

vsphere_ipv4_netmask = <span class="token string">"24"</span>

vsphere_ipv4_gateway = <span class="token string">"10.0.0.1"</span>

vsphere_dns_servers = <span class="token punctuation">[</span><span class="token string">"114.114.114.114"</span>,<span class="token string">"223.5.5.5"</span><span class="token punctuation">]</span>

  

vsphere_time_zone = <span class="token string">"Asia/Shanghai"</span>

  

k8s_deploy_name = <span class="token string">"k8s-deploy"</span>

k8s_deploy_cpu = <span class="token string">"2"</span>

k8s_deploy_memory = <span class="token string">"4096"</span>

<span class="token comment">#k8s deploy node IP is $vsphere_ipv4_address.9</span>

  

k8s_master_name = <span class="token string">"k8s-master0"</span>

k8s_master_cpu = <span class="token string">"2"</span>

k8s_master_memory = <span class="token string">"8196"</span>

k8s_master_count = <span class="token string">"3"</span>

<span class="token comment">#k8s master node IP is starts with $vsphere_ipv4_address.21</span>

<span class="token comment">#Don't deploy master node count more than 10</span>

  

k8s_node_name = <span class="token string">"k8s-node0"</span>

k8s_node_cpu = <span class="token string">"8"</span>

k8s_node_memory = <span class="token string">"16384"</span>

k8s_node_count = <span class="token string">"5"</span>

<span class="token comment">#k8s master node IP is starts with $vsphere_ipv4_address.31</span>

<span class="token comment">#Don't deploy worker node count more than 10</span>

  
  

esxi_hosts = <span class="token punctuation">{</span>

<span class="token string">"0"</span> = <span class="token string">"10.0.0.11"</span>

<span class="token string">"1"</span> = <span class="token string">"10.0.0.12"</span>

<span class="token string">"2"</span> = <span class="token string">"10.0.0.13"</span>

<span class="token string">"3"</span> = <span class="token string">"10.0.0.14"</span>

<span class="token string">"4"</span> = <span class="token string">"10.0.0.15"</span>

<span class="token punctuation">}</span>

</code></pre>
<h3 id="在vcenter里面创建预置的资源">1.4 在vCenter里面创建预置的资源</h3>
<p>1.4.1 创建Resource Pool，并且名字设置为<code>k8s-cluster-RP</code></p>
<p>1.4.2 创建VM Template，我们可以使用ubuntu16.04的ISO安装完毕之后，就可以直接关机并且转换为tempalte，并且名字设置为<code>ubuntu16.04-template</code></p>
<p>这步具体就不演示了，比较简单。</p>
<h3 id="terraform-gogogo">1.5 Terraform GOGOGO</h3>
<pre class=" language-shell"><code class="prism  language-shell">
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

  

* provider.vsphere: version = "~&gt; 1.4"

  

Terraform has been successfully initialized!

  

You may now begin working with Terraform. Try running "terraform plan" to see

any changes that are required for your infrastructure. All Terraform commands

should now work.

  

If you ever set or change modules or backend configuration for Terraform,

rerun this command to reinitialize your working directory. If you forget, other

commands will detect it and remind you to do so if necessary.

root@terraform:~/terraform/#

</code></pre>
<pre class=" language-shell"><code class="prism  language-shell">
root@terraform:~/terraform/# terraform plan

  

#如果没有配置错误，这里会出现每个创建的资源的详细配置项

  

</code></pre>
<pre class=" language-shell"><code class="prism  language-shell">
  

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

</code></pre>
<p>执行完毕之后，就能看到vCenter里面已经创建好所有的k8s集群的主机</p>
<p><code>terraform show</code>能够看到创建的所有资源的详细信息</p>

