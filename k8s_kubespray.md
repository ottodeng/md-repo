---


---

<h1 id="deploy-kubernetes-with-kubespray（集成vmware-cloud-provider动态创建pv）">Deploy Kubernetes with Kubespray（集成VMware Cloud Provider动态创建PV）</h1>
<p>接上篇介绍，Terraform部署完k8s的所有虚拟机之后，接下来就是在deploy node上部署kubespray来自动化安装kubernetes.</p>
<p>在本篇教程中，我还将StorageClass和vSAN做联动，这样当应用申请PVC的时候，就会自动创建PV，并且将PV关联到vSAN上的vmfs.</p>
<h2 id="初始化ubuntu">2.1 初始化Ubuntu</h2>
<h3 id="配置host表">2.1.1 配置host表</h3>
<pre class=" language-bash"><code class="prism  language-bash">root@k8s-deploy:~<span class="token comment"># cat /etc/hosts</span>
127.0.0.1       localhost
<span class="token comment"># The following lines are desirable for IPv6 capable hosts</span>
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.0.0.21 k8s-master01
10.0.0.22 k8s-master02 
10.0.0.23 k8s-master03
10.0.0.31 k8s-node01 
10.0.0.32 k8s-node02 
10.0.0.33 k8s-node03 
10.0.0.34 k8s-node04 
10.0.0.35 k8s-node05
</code></pre>
<h3 id="配置deploy节点免密登陆到k8s的所有节点">2.1.2 配置deploy节点免密登陆到k8s的所有节点</h3>
<pre class=" language-bash"><code class="prism  language-bash">ssh-keygen
 ssh-copy-id -i k8s-master01
 ssh-copy-id -i k8s-master02
 ssh-copy-id -i k8s-master03
 ssh-copy-id -i k8s-node01
 ssh-copy-id -i k8s-node02
 ssh-copy-id -i k8s-node03
 ssh-copy-id -i k8s-node04
 ssh-copy-id -i k8s-node05

然后将deploy node SSH到每一台节点上，目的是提前将每台主机的指纹加入到deploy node的known_hosts里面
</code></pre>
<h3 id="所有节点配置ntp">2.1.3 所有节点配置NTP</h3>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">apt-get</span> <span class="token function">install</span> ntp -y

ntp servers:
s1a.time.edu.cn
s1b.time.edu.cn
s1c.time.edu.cn
  
systemctl restart ntp
</code></pre>
<h3 id="所有节点安装python">2.1.4 所有节点安装python</h3>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">apt-get</span> <span class="token function">install</span> python -y
</code></pre>
<h3 id="在deploy-node上安装ansible">2.2 在deploy node上安装ansible</h3>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">apt-get</span> <span class="token function">install</span> software-properties-common
apt-add-repository ppa:ansible/ansible
<span class="token function">apt-get</span> update
<span class="token function">apt-get</span> <span class="token function">install</span> ansible -y
</code></pre>
<p>安装完毕之后，请确保ansible是从PPA安装的，并且版本大于2.5.2。</p>
<pre class=" language-bash"><code class="prism  language-bash">root@k8s-deploy:~<span class="token comment"># ansible --version</span>
ansible 2.5.2
  config <span class="token function">file</span> <span class="token operator">=</span> /etc/ansible/ansible.cfg
  configured module search path <span class="token operator">=</span> <span class="token punctuation">[</span>u<span class="token string">'/root/.ansible/plugins/modules'</span>, u<span class="token string">'/usr/share/ansible/plugins/modules'</span><span class="token punctuation">]</span>
  ansible python module location <span class="token operator">=</span> /usr/lib/python2.7/dist-packages/ansible
  executable location <span class="token operator">=</span> /usr/bin/ansible
  python version <span class="token operator">=</span> 2.7.12 <span class="token punctuation">(</span>default, Dec  4 2017, 14:50:18<span class="token punctuation">)</span> <span class="token punctuation">[</span>GCC 5.4.0 20160609<span class="token punctuation">]</span>
 
root@k8s-deploy:~<span class="token comment"># ansible-playbook --version</span>
ansible-playbook 2.5.2
  config <span class="token function">file</span> <span class="token operator">=</span> /etc/ansible/ansible.cfg
  configured module search path <span class="token operator">=</span> <span class="token punctuation">[</span>u<span class="token string">'/root/.ansible/plugins/modules'</span>, u<span class="token string">'/usr/share/ansible/plugins/modules'</span><span class="token punctuation">]</span>
  ansible python module location <span class="token operator">=</span> /usr/lib/python2.7/dist-packages/ansible
  executable location <span class="token operator">=</span> /usr/bin/ansible-playbook
  python version <span class="token operator">=</span> 2.7.12 <span class="token punctuation">(</span>default, Dec  4 2017, 14:50:18<span class="token punctuation">)</span> <span class="token punctuation">[</span>GCC 5.4.0 20160609<span class="token punctuation">]</span>
root@k8s-deploy:~<span class="token comment">#</span>
</code></pre>
<p>修改ansible默认的inventory</p>
<pre class=" language-bash"><code class="prism  language-bash">root@k8s-deploy:/opt<span class="token comment"># cat /etc/ansible/hosts </span>
<span class="token punctuation">[</span>k8s-all<span class="token punctuation">]</span>
k8s-master01
k8s-master02
k8s-master03
k8s-node01
k8s-node02
k8s-node03
k8s-node04
k8s-node05

<span class="token punctuation">[</span>k8s-master<span class="token punctuation">]</span>
k8s-master01
k8s-master02
k8s-master03


<span class="token punctuation">[</span>k8s-node<span class="token punctuation">]</span>
k8s-node01
k8s-node02
k8s-node03
k8s-node04
k8s-node05
</code></pre>
<p>最后测试一下</p>
<pre class=" language-bash"><code class="prism  language-bash">ansible all -m <span class="token function">ping</span>
ansible k8s-all -m copy -a <span class="token string">'src=/etc/hosts dest=/etc/hosts'</span>
ansible k8s-all -m shell -a <span class="token string">'date'</span>
</code></pre>
<h3 id="在deploy-node上安装kubespray并运行">2.3 在deploy node上安装kubespray并运行</h3>
<p>kubespray的环境要求</p>
<ul>
<li>Ansible v2.4</li>
<li>Jinja 2.9</li>
<li>Netaddr</li>
<li>Allow IPv4 forwarding</li>
<li>Ssh key must copied to inverotry</li>
<li>Access to internet</li>
<li>Memory SWAP is off</li>
</ul>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">apt-get</span> <span class="token function">install</span> python-pip -y
pip <span class="token function">install</span> --upgrade pip
<span class="token function">hash</span> -r
pip <span class="token function">install</span> jinja2
pip <span class="token function">install</span> netaddr
<span class="token function">apt-get</span> <span class="token function">install</span> python-netaddr -y
  
ansible all -m shell -a <span class="token string">'cat /proc/sys/net/ipv4/ip_forward'</span>
ansible all -m shell -a <span class="token string">'echo 1 &gt; /proc/sys/net/ipv4/ip_forward'</span>
ansible all -m shell -a <span class="token string">'sysctl -p'</span>
ansible all -m shell -a <span class="token string">'cat /proc/sys/net/ipv4/ip_forward'</span>
ansible all -m shell -a <span class="token string">'swapoff -a'</span>  <span class="token comment">#建议手动到每个节点上将/etc/fstab/的swap部分注释掉</span>
ansible all -m shell -a <span class="token string">'free -h'</span>
</code></pre>
<p>安装kubespray</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">git</span> clone https://github.com/kubernetes-incubator/kubespray.git
</code></pre>
<p>修改默认的inventory</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token comment"># Copy ``inventory/sample`` as ``inventory/mycluster``</span>
<span class="token function">cp</span> -rfp inventory/sample inventory/mycluster

<span class="token comment"># Review and change parameters under ``inventory/mycluster/group_vars``</span>
<span class="token function">cat</span> inventory/mycluster/group_vars/all.yml
<span class="token function">cat</span> inventory/mycluster/group_vars/k8s-cluster.yml
</code></pre>
<p>修改后的/inventory/mycluster/hosts.ini如下</p>
<pre class=" language-ini"><code class="prism  language-ini">k8s-master01 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.21</span>
k8s-master02 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.22</span>
k8s-master03 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.23</span>
k8s-node01 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.31</span>
k8s-node02 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.32</span>
k8s-node03 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.33</span>
k8s-node04 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.34</span>
k8s-node05 ansible_ssh_host<span class="token attr-value"><span class="token punctuation">=</span>10.0.0.35</span>


<span class="token selector">[kube-master]</span>
k8s-master01
k8s-master02
k8s-master03

<span class="token selector">[etcd]</span>
k8s-master01
k8s-master02
k8s-master03

<span class="token selector">[kube-node]</span>
k8s-node01
k8s-node02
k8s-node03
k8s-node04
k8s-node05

<span class="token selector">[kube-ingress]</span>
k8s-node01
k8s-node02
k8s-node03
k8s-node04
k8s-node05

<span class="token selector">[k8s-cluster:children]</span>
kube-master
kube-node
kube-ingress
</code></pre>
<p>修改后的/inventory/mycluster/group_vars/all.yml如下</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token comment"># Valid bootstrap options (required): ubuntu, coreos, centos, none</span>
<span class="token key atrule">bootstrap_os</span><span class="token punctuation">:</span> none

<span class="token comment">#Directory where etcd data stored</span>
<span class="token key atrule">etcd_data_dir</span><span class="token punctuation">:</span> /var/lib/etcd

<span class="token comment"># Directory where the binaries will be installed</span>
<span class="token key atrule">bin_dir</span><span class="token punctuation">:</span> /usr/local/bin

<span class="token comment">## The access_ip variable is used to define how other nodes should access</span>
<span class="token comment">## the node.  This is used in flannel to allow other flannel nodes to see</span>
<span class="token comment">## this node for example.  The access_ip is really useful AWS and Google</span>
<span class="token comment">## environments where the nodes are accessed remotely by the "public" ip,</span>
<span class="token comment">## but don't know about that address themselves.</span>
<span class="token comment">#access_ip: 1.1.1.1</span>

<span class="token comment">### LOADBALANCING AND ACCESS MODES</span>
<span class="token comment">## Enable multiaccess to configure etcd clients to access all of the etcd members directly</span>
<span class="token comment">## as the "http://hostX:port, http://hostY:port, ..." and ignore the proxy loadbalancers.</span>
<span class="token comment">## This may be the case if clients support and loadbalance multiple etcd servers natively.</span>
<span class="token comment">#etcd_multiaccess: true</span>

<span class="token comment">### ETCD: disable peer client cert authentication.</span>
<span class="token comment"># This affects ETCD_PEER_CLIENT_CERT_AUTH variable</span>
<span class="token comment">#etcd_peer_client_auth: true</span>

<span class="token comment">## External LB example config</span>
<span class="token comment">## apiserver_loadbalancer_domain_name: "elb.some.domain"</span>
<span class="token comment">#loadbalancer_apiserver:</span>
<span class="token comment">#  address: 1.2.3.4</span>
<span class="token comment">#  port: 1234</span>

<span class="token comment">## Internal loadbalancers for apiservers</span>
<span class="token comment">#loadbalancer_apiserver_localhost: true</span>

<span class="token comment">## Local loadbalancer should use this port instead, if defined.</span>
<span class="token comment">## Defaults to kube_apiserver_port (6443)</span>
<span class="token comment">#nginx_kube_apiserver_port: 8443</span>

<span class="token comment">### OTHER OPTIONAL VARIABLES</span>
<span class="token comment">## For some things, kubelet needs to load kernel modules.  For example, dynamic kernel services are needed</span>
<span class="token comment">## for mounting persistent volumes into containers.  These may not be loaded by preinstall kubernetes</span>
<span class="token comment">## processes.  For example, ceph and rbd backed volumes.  Set to true to allow kubelet to load kernel</span>
<span class="token comment">## modules.</span>
<span class="token comment">#kubelet_load_modules: false</span>

<span class="token comment">## Internal network total size. This is the prefix of the</span>
<span class="token comment">## entire network. Must be unused in your environment.</span>
<span class="token comment">#kube_network_prefix: 18</span>

<span class="token comment">## With calico it is possible to distributed routes with border routers of the datacenter.</span>
<span class="token comment">## Warning : enabling router peering will disable calico's default behavior ('node mesh').</span>
<span class="token comment">## The subnets of each nodes will be distributed by the datacenter router</span>
<span class="token comment">#peer_with_router: false</span>

<span class="token comment">## Upstream dns servers used by dnsmasq</span>
<span class="token comment">#upstream_dns_servers:</span>
<span class="token comment">#  - 8.8.8.8</span>
<span class="token comment">#  - 8.8.4.4</span>

<span class="token comment">## There are some changes specific to the cloud providers</span>
<span class="token comment">## for instance we need to encapsulate packets with some network plugins</span>
<span class="token comment">## If set the possible values are either 'gce', 'aws', 'azure', 'openstack', 'vsphere', or 'external'</span>
<span class="token comment">## When openstack is used make sure to source in the openstack credentials</span>
<span class="token comment">## like you would do when using nova-client before starting the playbook.</span>
<span class="token key atrule">cloud_provider</span><span class="token punctuation">:</span> vsphere

<span class="token key atrule">vsphere_vcenter_ip</span><span class="token punctuation">:</span> <span class="token string">"10.0.0.10"</span>
<span class="token key atrule">vsphere_vcenter_port</span><span class="token punctuation">:</span> <span class="token number">443</span>
<span class="token key atrule">vsphere_insecure</span><span class="token punctuation">:</span> <span class="token number">1</span>
<span class="token key atrule">vsphere_user</span><span class="token punctuation">:</span> <span class="token string">"administrator@vsphere.local"</span>
<span class="token key atrule">vsphere_password</span><span class="token punctuation">:</span> <span class="token string">"CHANGEME"</span>
<span class="token key atrule">vsphere_datacenter</span><span class="token punctuation">:</span> <span class="token string">"DataCenter"</span>
<span class="token key atrule">vsphere_datastore</span><span class="token punctuation">:</span> <span class="token string">"vsanDatastore"</span>
<span class="token key atrule">vsphere_working_dir</span><span class="token punctuation">:</span> <span class="token string">"k8s-nodes"</span>
<span class="token key atrule">vsphere_scsi_controller_type</span><span class="token punctuation">:</span> <span class="token string">"pvscsi"</span>
<span class="token key atrule">vsphere_resource_pool</span><span class="token punctuation">:</span> ""k8s<span class="token punctuation">-</span>cluster<span class="token punctuation">-</span>RP"
<span class="token comment">#这些配置是为了StorageClass能够动态创建PV，k8s本身需要获取vCenter的权限</span>

<span class="token comment">## When azure is used, you need to also set the following variables.</span>
<span class="token comment">## see docs/azure.md for details on how to get these values</span>
<span class="token comment">#azure_tenant_id:</span>
<span class="token comment">#azure_subscription_id:</span>
<span class="token comment">#azure_aad_client_id:</span>
<span class="token comment">#azure_aad_client_secret:</span>
<span class="token comment">#azure_resource_group:</span>
<span class="token comment">#azure_location:</span>
<span class="token comment">#azure_subnet_name:</span>
<span class="token comment">#azure_security_group_name:</span>
<span class="token comment">#azure_vnet_name:</span>
<span class="token comment">#azure_vnet_resource_group:</span>
<span class="token comment">#azure_route_table_name:</span>

<span class="token comment">## When OpenStack is used, Cinder version can be explicitly specified if autodetection fails (Fixed in 1.9: https://github.com/kubernetes/kubernetes/issues/50461)</span>
<span class="token comment">#openstack_blockstorage_version: "v1/v2/auto (default)"</span>
<span class="token comment">## When OpenStack is used, if LBaaSv2 is available you can enable it with the following 2 variables.</span>
<span class="token comment">#openstack_lbaas_enabled: True</span>
<span class="token comment">#openstack_lbaas_subnet_id: "Neutron subnet ID (not network ID) to create LBaaS VIP"</span>
<span class="token comment">## To enable automatic floating ip provisioning, specify a subnet.</span>
<span class="token comment">#openstack_lbaas_floating_network_id: "Neutron network ID (not subnet ID) to get floating IP from, disabled by default"</span>
<span class="token comment">## Override default LBaaS behavior</span>
<span class="token comment">#openstack_lbaas_use_octavia: False</span>
<span class="token comment">#openstack_lbaas_method: "ROUND_ROBIN"</span>
<span class="token comment">#openstack_lbaas_provider: "haproxy"</span>
<span class="token comment">#openstack_lbaas_create_monitor: "yes"</span>
<span class="token comment">#openstack_lbaas_monitor_delay: "1m"</span>
<span class="token comment">#openstack_lbaas_monitor_timeout: "30s"</span>
<span class="token comment">#openstack_lbaas_monitor_max_retries: "3"</span>

<span class="token comment">## Uncomment to enable experimental kubeadm deployment mode</span>
<span class="token comment">#kubeadm_enabled: false</span>
<span class="token comment">## Set these proxy values in order to update package manager and docker daemon to use proxies</span>
<span class="token comment">#http_proxy: ""</span>
<span class="token comment">#https_proxy: ""</span>
<span class="token comment">## Refer to roles/kubespray-defaults/defaults/main.yml before modifying no_proxy</span>
<span class="token comment">#no_proxy: ""</span>

<span class="token comment">## Uncomment this if you want to force overlay/overlay2 as docker storage driver</span>
<span class="token comment">## Please note that overlay2 is only supported on newer kernels</span>
<span class="token comment">#docker_storage_options: -s overlay2</span>

<span class="token comment"># Uncomment this if you have more than 3 nameservers, then we'll only use the first 3.</span>
<span class="token comment">#docker_dns_servers_strict: false</span>

<span class="token comment">## Default packages to install within the cluster, f.e:</span>
<span class="token comment">#kpm_packages:</span>
<span class="token comment"># - name: kube-system/grafana</span>

<span class="token comment">## Certificate Management</span>
<span class="token comment">## This setting determines whether certs are generated via scripts or whether a</span>
<span class="token comment">## cluster of Hashicorp's Vault is started to issue certificates (using etcd</span>
<span class="token comment">## as a backend). Options are "script" or "vault"</span>
<span class="token comment">#cert_management: script</span>

<span class="token comment"># Set to true to allow pre-checks to fail and continue deployment</span>
<span class="token comment">#ignore_assert_errors: false</span>

<span class="token comment">## Etcd auto compaction retention for mvcc key value store in hour</span>
<span class="token comment">#etcd_compaction_retention: 0</span>

<span class="token comment">## Set level of detail for etcd exported metrics, specify 'extensive' to include histogram metrics.</span>
<span class="token comment">#etcd_metrics: basic</span>

<span class="token comment">## Etcd is restricted by default to 512M on systems under 4GB RAM, 512MB is not enough for much more than testing.</span>
<span class="token comment">## Set this if your etcd nodes have less than 4GB but you want more RAM for etcd. Set to 0 for unrestricted RAM.</span>
<span class="token comment">#etcd_memory_limit: "512M"</span>

<span class="token comment"># The read-only port for the Kubelet to serve on with no authentication/authorization. Uncomment to enable.</span>
<span class="token key atrule">kube_read_only_port</span><span class="token punctuation">:</span> <span class="token number">10255</span>
<span class="token comment"># 开启这项的目的是可以让heapster可以读取kubelet的10255来取metric</span>
</code></pre>
<p>修改后的/inventory/mycluster/group_vars/k8s-cluster.yml如下</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token comment"># Kubernetes configuration dirs and system namespace.</span>
<span class="token comment"># Those are where all the additional config stuff goes</span>
<span class="token comment"># the kubernetes normally puts in /srv/kubernetes.</span>
<span class="token comment"># This puts them in a sane location and namespace.</span>
<span class="token comment"># Editing those values will almost surely break something.</span>
<span class="token key atrule">kube_config_dir</span><span class="token punctuation">:</span> /etc/kubernetes
<span class="token key atrule">kube_script_dir</span><span class="token punctuation">:</span> <span class="token string">"{{ bin_dir }}/kubernetes-scripts"</span>
<span class="token key atrule">kube_manifest_dir</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_config_dir }}/manifests"</span>

<span class="token comment"># This is where all the cert scripts and certs will be located</span>
<span class="token key atrule">kube_cert_dir</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_config_dir }}/ssl"</span>

<span class="token comment"># This is where all of the bearer tokens will be stored</span>
<span class="token key atrule">kube_token_dir</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_config_dir }}/tokens"</span>

<span class="token comment"># This is where to save basic auth file</span>
<span class="token key atrule">kube_users_dir</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_config_dir }}/users"</span>

<span class="token key atrule">kube_api_anonymous_auth</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>

<span class="token comment">## Change this to use another Kubernetes version, e.g. a current beta release</span>
<span class="token key atrule">kube_version</span><span class="token punctuation">:</span> v1.9.5

<span class="token comment"># Where the binaries will be downloaded.</span>
<span class="token comment"># Note: ensure that you've enough disk space (about 1G)</span>
<span class="token key atrule">local_release_dir</span><span class="token punctuation">:</span> <span class="token string">"/tmp/releases"</span>
<span class="token comment"># Random shifts for retrying failed ops like pushing/downloading</span>
<span class="token key atrule">retry_stagger</span><span class="token punctuation">:</span> <span class="token number">5</span>

<span class="token comment"># This is the group that the cert creation scripts chgrp the</span>
<span class="token comment"># cert files to. Not really changeable...</span>
<span class="token key atrule">kube_cert_group</span><span class="token punctuation">:</span> kube<span class="token punctuation">-</span>cert

<span class="token comment"># Cluster Loglevel configuration</span>
<span class="token key atrule">kube_log_level</span><span class="token punctuation">:</span> <span class="token number">2</span>

<span class="token comment"># Users to create for basic auth in Kubernetes API via HTTP</span>
<span class="token comment"># Optionally add groups for user</span>
<span class="token key atrule">kube_api_pwd</span><span class="token punctuation">:</span> <span class="token string">"{{ lookup('password', inventory_dir + '/credentials/kube_user.creds length=15 chars=ascii_letters,digits') }}"</span>
<span class="token key atrule">kube_users</span><span class="token punctuation">:</span>
  <span class="token key atrule">kube</span><span class="token punctuation">:</span>
    <span class="token key atrule">pass</span><span class="token punctuation">:</span> <span class="token string">"{{kube_api_pwd}}"</span>
    <span class="token key atrule">role</span><span class="token punctuation">:</span> admin
    <span class="token key atrule">groups</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> system<span class="token punctuation">:</span>masters

<span class="token comment">## It is possible to activate / deactivate selected authentication methods (basic auth, static token auth)</span>
<span class="token comment">#kube_oidc_auth: false</span>
<span class="token comment">#kube_basic_auth: false</span>
<span class="token comment">#kube_token_auth: false</span>


<span class="token comment">## Variables for OpenID Connect Configuration https://kubernetes.io/docs/admin/authentication/</span>
<span class="token comment">## To use OpenID you have to deploy additional an OpenID Provider (e.g Dex, Keycloak, ...)</span>

<span class="token comment"># kube_oidc_url: https:// ...</span>
<span class="token comment"># kube_oidc_client_id: kubernetes</span>
<span class="token comment">## Optional settings for OIDC</span>
<span class="token comment"># kube_oidc_ca_file: {{ kube_cert_dir }}/ca.pem</span>
<span class="token comment"># kube_oidc_username_claim: sub</span>
<span class="token comment"># kube_oidc_groups_claim: groups</span>


<span class="token comment"># Choose network plugin (cilium, calico, contiv, weave or flannel)</span>
<span class="token comment"># Can also be set to 'cloud', which lets the cloud provider setup appropriate routing</span>
<span class="token key atrule">kube_network_plugin</span><span class="token punctuation">:</span> calico

<span class="token comment"># weave's network password for encryption</span>
<span class="token comment"># if null then no network encryption</span>
<span class="token comment"># you can use --extra-vars to pass the password in command line</span>
<span class="token key atrule">weave_password</span><span class="token punctuation">:</span> EnterPasswordHere

<span class="token comment"># Weave uses consensus mode by default</span>
<span class="token comment"># Enabling seed mode allow to dynamically add or remove hosts</span>
<span class="token comment"># https://www.weave.works/docs/net/latest/ipam/</span>
<span class="token key atrule">weave_mode_seed</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># This two variable are automatically changed by the weave's role, do not manually change these values</span>
<span class="token comment"># To reset values :</span>
<span class="token comment"># weave_seed: uninitialized</span>
<span class="token comment"># weave_peers: uninitialized</span>
<span class="token key atrule">weave_seed</span><span class="token punctuation">:</span> uninitialized
<span class="token key atrule">weave_peers</span><span class="token punctuation">:</span> uninitialized

<span class="token comment"># Set the MTU of Weave (default 1376, Jumbo Frames: 8916)</span>
<span class="token key atrule">weave_mtu</span><span class="token punctuation">:</span> <span class="token number">1376</span>

<span class="token comment"># Enable kubernetes network policies</span>
<span class="token key atrule">enable_network_policy</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># Kubernetes internal network for services, unused block of space.</span>
<span class="token key atrule">kube_service_addresses</span><span class="token punctuation">:</span> 10.233.0.0/18

<span class="token comment"># internal network. When used, it will assign IP</span>
<span class="token comment"># addresses from this range to individual pods.</span>
<span class="token comment"># This network must be unused in your network infrastructure!</span>
<span class="token key atrule">kube_pods_subnet</span><span class="token punctuation">:</span> 10.233.64.0/18

<span class="token comment"># internal network node size allocation (optional). This is the size allocated</span>
<span class="token comment"># to each node on your network.  With these defaults you should have</span>
<span class="token comment"># room for 4096 nodes with 254 pods per node.</span>
<span class="token key atrule">kube_network_node_prefix</span><span class="token punctuation">:</span> <span class="token number">24</span>

<span class="token comment"># The port the API Server will be listening on.</span>
<span class="token key atrule">kube_apiserver_ip</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_service_addresses|ipaddr('net')|ipaddr(1)|ipaddr('address') }}"</span>
<span class="token key atrule">kube_apiserver_port</span><span class="token punctuation">:</span> <span class="token number">6443 </span><span class="token comment"># (https)</span>
<span class="token key atrule">kube_apiserver_insecure_port</span><span class="token punctuation">:</span> <span class="token number">8080 </span><span class="token comment"># (http)</span>
<span class="token comment"># Set to 0 to disable insecure port - Requires RBAC in authorization_modes and kube_api_anonymous_auth: true</span>
<span class="token comment">#kube_apiserver_insecure_port: 0 # (disabled)</span>

<span class="token comment"># Kube-proxy proxyMode configuration.</span>
<span class="token comment"># Can be ipvs, iptables</span>
<span class="token key atrule">kube_proxy_mode</span><span class="token punctuation">:</span> ipvs
<span class="token comment"># 这里我修改了默认的iptables为ipvs</span>

<span class="token comment">## Encrypting Secret Data at Rest (experimental)</span>
<span class="token key atrule">kube_encrypt_secret_data</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># DNS configuration.</span>
<span class="token comment"># Kubernetes cluster name, also will be used as DNS domain</span>
<span class="token key atrule">cluster_name</span><span class="token punctuation">:</span> cluster.local
<span class="token comment"># Subdomains of DNS domain to be resolved via /etc/resolv.conf for hostnet pods</span>
<span class="token key atrule">ndots</span><span class="token punctuation">:</span> <span class="token number">2</span>
<span class="token comment"># Can be dnsmasq_kubedns, kubedns, coredns, coredns_dual, manual or none</span>
<span class="token key atrule">dns_mode</span><span class="token punctuation">:</span> coredns
<span class="token comment"># 这里我修改了默认的kubedns为coredns</span>
<span class="token comment"># Set manual server if using a custom cluster DNS server</span>
<span class="token comment">#manual_dns_server: 10.x.x.x</span>

<span class="token comment"># Can be docker_dns, host_resolvconf or none</span>
<span class="token key atrule">resolvconf_mode</span><span class="token punctuation">:</span> docker_dns
<span class="token comment"># Deploy netchecker app to verify DNS resolve as an HTTP service</span>
<span class="token key atrule">deploy_netchecker</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
<span class="token comment"># Ip address of the kubernetes skydns service</span>
<span class="token key atrule">skydns_server</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_service_addresses|ipaddr('net')|ipaddr(3)|ipaddr('address') }}"</span>
<span class="token key atrule">skydns_server_secondary</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_service_addresses|ipaddr('net')|ipaddr(4)|ipaddr('address') }}"</span>
<span class="token key atrule">dnsmasq_dns_server</span><span class="token punctuation">:</span> <span class="token string">"{{ kube_service_addresses|ipaddr('net')|ipaddr(2)|ipaddr('address') }}"</span>
<span class="token key atrule">dns_domain</span><span class="token punctuation">:</span> <span class="token string">"{{ cluster_name }}"</span>

<span class="token comment"># Path used to store Docker data</span>
<span class="token key atrule">docker_daemon_graph</span><span class="token punctuation">:</span> <span class="token string">"/var/lib/docker"</span>

<span class="token comment">## A string of extra options to pass to the docker daemon.</span>
<span class="token comment">## This string should be exactly as you wish it to appear.</span>
<span class="token comment">## An obvious use case is allowing insecure-registry access</span>
<span class="token comment">## to self hosted registries like so:</span>

<span class="token key atrule">docker_options</span><span class="token punctuation">:</span> <span class="token string">"--insecure-registry={{ kube_service_addresses }} --graph={{ docker_daemon_graph }}  {{ docker_log_opts }}"</span>
<span class="token key atrule">docker_bin_dir</span><span class="token punctuation">:</span> <span class="token string">"/usr/bin"</span>

<span class="token comment"># Settings for containerized control plane (etcd/kubelet/secrets)</span>
<span class="token key atrule">etcd_deployment_type</span><span class="token punctuation">:</span> docker
<span class="token key atrule">kubelet_deployment_type</span><span class="token punctuation">:</span> host
<span class="token key atrule">vault_deployment_type</span><span class="token punctuation">:</span> docker
<span class="token key atrule">helm_deployment_type</span><span class="token punctuation">:</span> host

<span class="token comment"># K8s image pull policy (imagePullPolicy)</span>
<span class="token key atrule">k8s_image_pull_policy</span><span class="token punctuation">:</span> IfNotPresent

<span class="token comment"># Kubernetes dashboard</span>
<span class="token comment"># RBAC required. see docs/getting-started.md for access details.</span>
<span class="token key atrule">dashboard_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>

<span class="token comment"># Monitoring apps for k8s</span>
<span class="token key atrule">efk_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># Helm deployment</span>
<span class="token key atrule">helm_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># Istio deployment</span>
<span class="token key atrule">istio_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># Registry deployment</span>
<span class="token key atrule">registry_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
<span class="token comment"># registry_namespace: "{{ system_namespace }}"</span>
<span class="token comment"># registry_storage_class: ""</span>
<span class="token comment"># registry_disk_size: "10Gi"</span>

<span class="token comment"># Local volume provisioner deployment</span>
<span class="token key atrule">local_volume_provisioner_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
<span class="token comment"># local_volume_provisioner_namespace: "{{ system_namespace }}"</span>
<span class="token comment"># local_volume_provisioner_base_dir: /mnt/disks</span>
<span class="token comment"># local_volume_provisioner_mount_dir: /mnt/disks</span>
<span class="token comment"># local_volume_provisioner_storage_class: local-storage</span>

<span class="token comment"># CephFS provisioner deployment</span>
<span class="token key atrule">cephfs_provisioner_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
<span class="token comment"># cephfs_provisioner_namespace: "{{ system_namespace }}"</span>
<span class="token comment"># cephfs_provisioner_cluster: ceph</span>
<span class="token comment"># cephfs_provisioner_monitors:</span>
<span class="token comment">#   - 172.24.0.1:6789</span>
<span class="token comment">#   - 172.24.0.2:6789</span>
<span class="token comment">#   - 172.24.0.3:6789</span>
<span class="token comment"># cephfs_provisioner_admin_id: admin</span>
<span class="token comment"># cephfs_provisioner_secret: secret</span>
<span class="token comment"># cephfs_provisioner_storage_class: cephfs</span>

<span class="token comment"># Nginx ingress controller deployment</span>
<span class="token key atrule">ingress_nginx_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>
<span class="token comment"># 将ingress打开</span>
<span class="token comment"># ingress_nginx_host_network: false</span>
<span class="token comment"># ingress_nginx_namespace: "ingress-nginx"</span>
<span class="token comment"># ingress_nginx_insecure_port: 80</span>
<span class="token comment"># ingress_nginx_secure_port: 443</span>
<span class="token comment"># ingress_nginx_configmap:</span>
<span class="token comment">#   map-hash-bucket-size: "128"</span>
<span class="token comment">#   ssl-protocols: "SSLv2"</span>
<span class="token comment"># ingress_nginx_configmap_tcp_services:</span>
<span class="token comment">#   9000: "default/example-go:8080"</span>
<span class="token comment"># ingress_nginx_configmap_udp_services:</span>
<span class="token comment">#   53: "kube-system/kube-dns:53"</span>

<span class="token comment"># Cert manager deployment</span>
<span class="token key atrule">cert_manager_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
<span class="token comment"># cert_manager_namespace: "cert-manager"</span>

<span class="token comment"># Add Persistent Volumes Storage Class for corresponding cloud provider ( OpenStack is only supported now )</span>
<span class="token key atrule">persistent_volumes_enabled</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>

<span class="token comment"># Make a copy of kubeconfig on the host that runs Ansible in {{ inventory_dir }}/artifacts</span>
<span class="token comment"># kubeconfig_localhost: false</span>
<span class="token comment"># Download kubectl onto the host that runs Ansible in {{ bin_dir }}</span>
<span class="token comment"># kubectl_localhost: false</span>

<span class="token comment"># dnsmasq</span>
<span class="token comment"># dnsmasq_upstream_dns_servers:</span>
<span class="token comment">#  - /resolvethiszone.with/10.0.4.250</span>
<span class="token comment">#  - 8.8.8.8</span>

<span class="token comment">#  Enable creation of QoS cgroup hierarchy, if true top level QoS and pod cgroups are created. (default true)</span>
<span class="token comment"># kubelet_cgroups_per_qos: true</span>

<span class="token comment"># A comma separated list of levels of node allocatable enforcement to be enforced by kubelet.</span>
<span class="token comment"># Acceptable options are 'pods', 'system-reserved', 'kube-reserved' and ''. Default is "".</span>
<span class="token comment"># kubelet_enforce_node_allocatable: pods</span>

<span class="token comment">## Supplementary addresses that can be added in kubernetes ssl keys.</span>
<span class="token comment">## That can be useful for example to setup a keepalived virtual IP</span>
<span class="token comment"># supplementary_addresses_in_ssl_keys: [10.0.0.1, 10.0.0.2, 10.0.0.3]</span>

<span class="token comment">## Running on top of openstack vms with cinder enabled may lead to unschedulable pods due to NoVolumeZoneConflict restriction in kube-scheduler.</span>
<span class="token comment">## See https://github.com/kubernetes-incubator/kubespray/issues/2141</span>
<span class="token comment">## Set this variable to true to get rid of this issue</span>
<span class="token key atrule">volume_cross_zone_attachment</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
</code></pre>
<p>修改所有的grc.io和quay.io的image，我已经将这些image镜像到我的docker hub的仓库上。<br>
在/kubespray目录下面创建change_registry.sh</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token shebang important">#!/bin/bash</span>
all_image_files<span class="token operator">=</span><span class="token punctuation">(</span>
roles/download/defaults/main.yml
roles/kubernetes-apps/ansible/defaults/main.yml
<span class="token punctuation">)</span>

<span class="token keyword">for</span> <span class="token function">file</span> <span class="token keyword">in</span> <span class="token variable">${all_image_files[@]}</span> <span class="token punctuation">;</span> <span class="token keyword">do</span>
    <span class="token function">sed</span> -i <span class="token string">'s/gcr.io\/google-containers\//ottodeng\/gcr.io_google-containers_/g'</span> <span class="token variable">$file</span>
    <span class="token function">sed</span> -i <span class="token string">'s/gcr.io\/google_containers\//ottodeng\/gcr.io_google_containers_/g'</span> <span class="token variable">$file</span>
    <span class="token function">sed</span> -i <span class="token string">'s/quay.io\/coreos\//ottodeng\/quay.io_coreos_/g'</span> <span class="token variable">$file</span>
    <span class="token function">sed</span> -i <span class="token string">'s/quay.io\/calico\//ottodeng\/quay.io_calico_/g'</span> <span class="token variable">$file</span>
    <span class="token function">sed</span> -i <span class="token string">'s/quay.io\/external_storage\//ottodeng\/quay.io_external_storage_/g'</span> <span class="token variable">$file</span>
    <span class="token function">sed</span> -i <span class="token string">'s/quay.io\/kubespray\//ottodeng\/quay.io_kubespray_/g'</span> <span class="token variable">$file</span>
    <span class="token function">sed</span> -i <span class="token string">'s/quay.io\/kubernetes-ingress-controller\//ottodeng\/quay.io_kubernetes-ingress-controller_/g'</span> <span class="token variable">$file</span>
<span class="token keyword">done</span>
</code></pre>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">chmod</span> +x change_registry.sh
./change_registry.sh
</code></pre>
<p>执行完毕之后，可以查看一下修改后的内容，确保已经修改成功</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">grep</span> -irn <span class="token string">"grc.io"</span>
<span class="token function">grep</span> -irn <span class="token string">"quay.io"</span>
</code></pre>
<p>给kubespray的执行结果输出日志</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">cd</span> ~/kubespray
<span class="token keyword">echo</span> <span class="token string">'log_path = /var/log/ansible.log'</span> <span class="token operator">&gt;</span> ansible.cfg
</code></pre>
<p>开始执行kubespray</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">cd</span> ~/kubespray
ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml
</code></pre>
<p>当执行完毕之后，大概的样子如下图所示：</p>
<p><img src="https://res.cloudinary.com/ottodeng/image/upload/v1526028915/image2018-5-4_16_57_29.png" alt="FINISH"></p>
<p>如果没有failed，则证明已经部署成功，如果有failed，则可以查看/var/log/ansible.log的失败日志。</p>
<h3 id="确认k8s集群运行正常，并且能够通过storageclass动态创建pv">2.4 确认k8s集群运行正常，并且能够通过StorageClass动态创建PV</h3>
<pre class=" language-bash"><code class="prism  language-bash">root@k8s-master01:~<span class="token comment"># kubectl get cs</span>
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   <span class="token punctuation">{</span><span class="token string">"health"</span><span class="token keyword">:</span> <span class="token string">"true"</span><span class="token punctuation">}</span>  
etcd-2               Healthy   <span class="token punctuation">{</span><span class="token string">"health"</span><span class="token keyword">:</span> <span class="token string">"true"</span><span class="token punctuation">}</span>  
etcd-0               Healthy   <span class="token punctuation">{</span><span class="token string">"health"</span><span class="token keyword">:</span> <span class="token string">"true"</span><span class="token punctuation">}</span>  
root@k8s-master01:~<span class="token comment"># kubectl get node</span>
NAME             STATUS    ROLES          AGE       VERSION
k8s-master01   Ready     master         7m        v1.9.5
k8s-master02   Ready     master         7m        v1.9.5
k8s-master03   Ready     master         7m        v1.9.5
k8s-node01     Ready     ingress,node   7m        v1.9.5
k8s-node02     Ready     ingress,node   7m        v1.9.5
k8s-node03     Ready     ingress,node   7m        v1.9.5
k8s-node04     Ready     ingress,node   7m        v1.9.5
k8s-node05     Ready     ingress,node   7m        v1.9.5
root@k8s-master01:~<span class="token comment"># kubectl cluster-info</span>
Kubernetes master is running at https://10.0.0.21:6443
CoreDNS is running at https://10.0.0.21:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
 
To further debug and diagnose cluster problems, use <span class="token string">'kubectl cluster-info dump'</span><span class="token keyword">.</span>
  
root@k8s-master01:~<span class="token comment"># kubectl get pod --all-namespaces -o wide</span>
NAMESPACE       NAME                                       READY     STATUS    RESTARTS   AGE       IP                NODE
ingress-nginx   ingress-nginx-controller-5pssh             1/1       Running   0          5m        10.233.90.129    k8s-node04
ingress-nginx   ingress-nginx-controller-76g64             1/1       Running   0          5m        10.233.64.65     k8s-node03
ingress-nginx   ingress-nginx-controller-9cxg8             1/1       Running   0          5m        10.233.97.66     k8s-node05
ingress-nginx   ingress-nginx-controller-z6qv6             1/1       Running   0          5m        10.233.121.129   k8s-node02
ingress-nginx   ingress-nginx-controller-z9bxw             1/1       Running   0          5m        10.233.65.65     k8s-node01
ingress-nginx   ingress-nginx-default-backend-v1.4-k2jr4   1/1       Running   0          5m        10.233.97.65     k8s-node05
kube-system     calico-node-7tcgx                          1/1       Running   0          5m        10.0.0.21      k8s-master01
kube-system     calico-node-cspmf                          1/1       Running   0          5m        10.0.0.22      k8s-master02
kube-system     calico-node-dqtxd                          1/1       Running   0          5m        10.0.0.32      k8s-node02
kube-system     calico-node-j46p2                          1/1       Running   0          5m        10.0.0.35      k8s-node05
kube-system     calico-node-mqxd9                          1/1       Running   0          5m        10.0.0.34      k8s-node04
kube-system     calico-node-nrng6                          1/1       Running   0          5m        10.0.0.23      k8s-master03
kube-system     calico-node-psk2q                          1/1       Running   0          5m        10.0.0.31      k8s-node01
kube-system     calico-node-s6thl                          1/1       Running   0          5m        10.0.0.33      k8s-node03
kube-system     coredns-7dfc584b6c-m4bs5                   1/1       Running   0          5m        10.233.121.130   k8s-node02
kube-system     coredns-7dfc584b6c-trwxp                   1/1       Running   0          5m        10.233.90.130    k8s-node04
kube-system     kube-apiserver-k8s-master01              1/1       Running   0          5m        10.0.0.21      k8s-master01
kube-system     kube-apiserver-k8s-master02              1/1       Running   0          5m        10.0.0.22      k8s-master02
kube-system     kube-apiserver-k8s-master03              1/1       Running   0          5m        10.0.0.23      k8s-master03
kube-system     kube-controller-manager-k8s-master01     1/1       Running   0          6m        10.0.0.21      k8s-master01
kube-system     kube-controller-manager-k8s-master02     1/1       Running   0          6m        10.0.0.22      k8s-master02
kube-system     kube-controller-manager-k8s-master03     1/1       Running   0          6m        10.0.0.23      k8s-master03
kube-system     kube-proxy-k8s-master01                  1/1       Running   0          5m        10.0.0.21      k8s-master01
kube-system     kube-proxy-k8s-master02                  1/1       Running   0          5m        10.0.0.22      k8s-master02
kube-system     kube-proxy-k8s-master03                  1/1       Running   0          5m        10.0.0.23      k8s-master03
kube-system     kube-proxy-k8s-node01                    1/1       Running   0          5m        10.0.0.31      k8s-node01
kube-system     kube-proxy-k8s-node02                    1/1       Running   0          6m        10.0.0.32      k8s-node02
kube-system     kube-proxy-k8s-node03                    1/1       Running   0          5m        10.0.0.33      k8s-node03
kube-system     kube-proxy-k8s-node04                    1/1       Running   0          6m        10.0.0.34      k8s-node04
kube-system     kube-proxy-k8s-node05                    1/1       Running   0          5m        10.0.0.35      k8s-node05
kube-system     kube-scheduler-k8s-master01              1/1       Running   0          6m        10.0.0.21      k8s-master01
kube-system     kube-scheduler-k8s-master02              1/1       Running   0          6m        10.0.0.22      k8s-master02
kube-system     kube-scheduler-k8s-master03              1/1       Running   0          6m        10.0.0.23      k8s-master03
kube-system     kubernetes-dashboard-66fc89bd85-k977x      1/1       Running   0          5m        10.233.121.131   k8s-node02
kube-system     nginx-proxy-k8s-node01                   1/1       Running   0          6m        10.0.0.31      k8s-node01
kube-system     nginx-proxy-k8s-node02                   1/1       Running   0          5m        10.0.0.32      k8s-node02
kube-system     nginx-proxy-k8s-node03                   1/1       Running   0          5m        10.0.0.33      k8s-node03
kube-system     nginx-proxy-k8s-node04                   1/1       Running   0          5m        10.0.0.34      k8s-node04
kube-system     nginx-proxy-k8s-node05                   1/1       Running   0          5m        10.0.0.35      k8s-node05
root@k8s-master01:~<span class="token comment">#</span>

root@k8s-master01:~<span class="token comment"># calicoctl node status</span>
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+------------+-------------+
<span class="token operator">|</span> PEER ADDRESS <span class="token operator">|</span>     PEER TYPE     <span class="token operator">|</span> STATE <span class="token operator">|</span>   SINCE    <span class="token operator">|</span>    INFO     <span class="token operator">|</span>
+--------------+-------------------+-------+------------+-------------+
<span class="token operator">|</span> 10.0.0.22    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
<span class="token operator">|</span> 10.0.0.23    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
<span class="token operator">|</span> 10.0.0.31    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
<span class="token operator">|</span> 10.0.0.32    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
<span class="token operator">|</span> 10.0.0.33    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
<span class="token operator">|</span> 10.0.0.34    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
<span class="token operator">|</span> 10.0.0.35    <span class="token operator">|</span> node-to-node mesh <span class="token operator">|</span> up    <span class="token operator">|</span> 2018-05-07 <span class="token operator">|</span> Established <span class="token operator">|</span>
+--------------+-------------------+-------+------------+-------------+

IPv6 BGP status
No IPv6 peers found.

</code></pre>
<p>创建StorageClass和PV</p>
<pre class=" language-yaml"><code class="prism  language-yaml">cat vsphere<span class="token punctuation">-</span>volume<span class="token punctuation">-</span>sc<span class="token punctuation">-</span>fast.yaml

<span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> storage.k8s.io/v1
<span class="token key atrule">kind</span><span class="token punctuation">:</span> StorageClass
<span class="token key atrule">metadata</span><span class="token punctuation">:</span> 
  <span class="token key atrule">name</span><span class="token punctuation">:</span> fast
<span class="token key atrule">provisioner</span><span class="token punctuation">:</span> kubernetes.io/vsphere<span class="token punctuation">-</span>volume
<span class="token key atrule">parameters</span><span class="token punctuation">:</span> 
  <span class="token key atrule">datastore</span><span class="token punctuation">:</span> vsanDatastore
  <span class="token key atrule">diskformat</span><span class="token punctuation">:</span> thin
  <span class="token key atrule">fstype</span><span class="token punctuation">:</span> ext3
</code></pre>
<pre class=" language-yaml"><code class="prism  language-yaml">cat vsphere<span class="token punctuation">-</span>volume<span class="token punctuation">-</span>pvcsc.yaml

<span class="token key atrule">kind</span><span class="token punctuation">:</span> PersistentVolumeClaim
<span class="token key atrule">apiVersion</span><span class="token punctuation">:</span> v1
<span class="token key atrule">metadata</span><span class="token punctuation">:</span>
  <span class="token key atrule">name</span><span class="token punctuation">:</span> pvcsc001
  <span class="token key atrule">annotations</span><span class="token punctuation">:</span>
    <span class="token key atrule">volume.beta.kubernetes.io/storage-class</span><span class="token punctuation">:</span> fast
<span class="token key atrule">spec</span><span class="token punctuation">:</span>
  <span class="token key atrule">accessModes</span><span class="token punctuation">:</span>
    <span class="token punctuation">-</span> ReadWriteOnce
  <span class="token key atrule">resources</span><span class="token punctuation">:</span>
    <span class="token key atrule">requests</span><span class="token punctuation">:</span>
      <span class="token key atrule">storage</span><span class="token punctuation">:</span> 2Gi
</code></pre>
<pre class=" language-bash"><code class="prism  language-bash">root@k8sz1-master01:~<span class="token comment"># kubectl get pvc</span>
NAME       STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvcsc001   Bound     pvc-5adb155b-4f7c-11e8-aaf4-00505698233c   2Gi        RWO            fast           4s
root@k8sz1-master01:~<span class="token comment"># kubectl get pv</span>
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM              STORAGECLASS   REASON    AGE
pvc-5adb155b-4f7c-11e8-aaf4-00505698233c   2Gi        RWO            Delete           Bound     default/pvcsc001   fast                     7s
root@k8sz1-master01:~<span class="token comment">#</span>
</code></pre>
<p>可以查看到pvc已经正常的Bound到PV</p>

