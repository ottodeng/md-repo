---


---

<h1 id="deploy-kubernetes-with-terraform--kubesprayansible">Deploy Kubernetes with Terraform &amp; Kubespray(ansible)</h1>
<h2 id="terraform">1. Terraform</h2>
<h3 id="what-is-terraform">What is Terraform?</h3>
<p>Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.</p>
<p>Configuration files describe to Terraform the components needed to run a single application or your entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. As the configuration changes, Terraform is able to determine what changed and create incremental execution plans which can be applied.</p>
<p>The infrastructure Terraform can manage includes low-level components such as compute instances, storage, and networking, as well as high-level components such as DNS entries, SaaS features, etc.</p>
<p>说白了，就是“<strong>Write, Plan, and create Infrastructure as Code</strong>”</p>
<hr>
<p>首先我们来安装Terraform</p>
<h3 id="install-go-1.10">1.1 Install GO 1.10</h3>
<p>wget <a href="https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz">https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz</a><br>
tar -C /usr/local -xzf go1.10.1.linux-amd64.tar.gz</p>
<p>export PATH=/usr/local/go/bin:$PATH<br>
export GOPATH=/root/go<br>
export GOROOT=/usr/local/go</p>
<h3 id="section">1.2</h3>

