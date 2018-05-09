---


---

<h1 id="deploy-kubernetes-with-terraform--kubesprayansible">Deploy Kubernetes with Terraform &amp; Kubespray(ansible)</h1>
<p>具体环境如下：</p>

<table>
<thead>
<tr>
<th>ESXi HOST</th>
<th>IP</th>
<th>Remark</th>
</tr>
</thead>
<tbody>
<tr>
<td>esxi-host01</td>
<td>10.0.0.11</td>
<td>#ESXi host01</td>
</tr>
<tr>
<td>esxi-host02</td>
<td>10.0.0.12</td>
<td>#ESXi host02</td>
</tr>
<tr>
<td>esxi-host03</td>
<td>10.0.0.13</td>
<td>#ESXi host03</td>
</tr>
<tr>
<td>esxi-host04</td>
<td>10.0.0.14</td>
<td>#ESXi host04</td>
</tr>
<tr>
<td>esxi-host05</td>
<td>10.0.0.15</td>
<td>#ESXi host05</td>
</tr>
<tr>
<td>vCenter</td>
<td>10.0.0.10</td>
<td>#vCenter</td>
</tr>
<tr>
<td>k8s-deploy</td>
<td>10.0.0.9</td>
<td>#Deploy Node</td>
</tr>
<tr>
<td>k8s-master01</td>
<td>10.0.0.21</td>
<td>#Master Node</td>
</tr>
<tr>
<td>k8s-master02</td>
<td>10.0.0.22</td>
<td>#Master Node</td>
</tr>
<tr>
<td>k8s-master03</td>
<td>10.0.0.23</td>
<td>#Master Node</td>
</tr>
<tr>
<td>k8s-node01</td>
<td>10.0.0.31</td>
<td>#Workder Node</td>
</tr>
<tr>
<td>k8s-node02</td>
<td>10.0.0.32</td>
<td>#Workder Node</td>
</tr>
<tr>
<td>k8s-node03</td>
<td>10.0.0.33</td>
<td>#Workder Node</td>
</tr>
<tr>
<td>k8s-node04</td>
<td>10.0.0.34</td>
<td>#Workder Node</td>
</tr>
<tr>
<td>k8s-node05</td>
<td>10.0.0.35</td>
<td>#Workder Node</td>
</tr>
</tbody>
</table><h2 id="terraform">1. Terraform</h2>
<h3 id="what-is-terraform">What is Terraform?</h3>
<p>Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.</p>
<p>Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.</p>
<p>The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.</p>
<p>说白了，就是“<strong>Write, Plan, and create Infrastructure as Code</strong>”</p>
<hr>
<p>首先我们来安装Terraform</p>
<h3 id="install-go-1.10">1.1 Install GO 1.10</h3>
<pre><code>wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz tar -C
/usr/local -xzf go1.10.1.linux-amd64.tar.gz 

export PATH=/usr/local/go/bin:$PATH export GOPATH=/root/go export
GOROOT=/usr/local/go
</code></pre>
<h3 id="install-terraform-0.11.7">1.2 Install Terraform 0.11.7</h3>
<pre><code>wget https://releases.hashicorp.com/terraform/0.11.7/terraform_0.11.7_linux_amd64.zip
unzip terraform_0.11.7_linux_amd64.zip 
cp terraform /usr/local/bin/
</code></pre>

