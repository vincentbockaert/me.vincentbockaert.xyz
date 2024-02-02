---
title: 'K3s on Hetzner'
date: "2024-02-02T18:34:46+01:00"
draft: false
tags:
  - DevOps
  - kubernetes
  - k3s
  - Hetzner cloud
---

## Intro - k3s on hcloud

![](/img/projects/k3s-on-hetzner/german-cloud.jpg)

Recently, I found myself wanting to learn more about kubernetes, as well as spending some time on both running and managing a cluster of my own.

Renting a cluster from cloud providers like Azure, GCP and AWS, is straightforward but then I'd have cough up a lot of money for little compute.
Additionally, I wouldn't really be managing the servers or nodes if I went with a managed solution.

Let's be honest... setting it yourself is part of the fun :D

 _(Note: Google does have a very limited free-tier)_

Therefore I got searching for a cheap cloud provider, offering mostly raw VM's.
Additionally the provider should have a terraform provider, as I like to automate infrastructure provisioning, if the provider also has datacenters relatively close to me, it would be even better.

With the above in mind, I landed on [Hetzner Cloud (referral link for some free bucks)](https://hetzner.cloud/?ref=QpeW4Od6re9X).
- They offer VM's with 2 CPU's, 4 GB RAM at less than 5 euros a month.
- Have a well-documentend terraform provider
- Have datacenters in Germany, Helsinki and USA
    - As I'm living in Belgium, the datacenter in Germany will do fine

For the kubernetes ,_distribution or flavor_, I went with [k3s](https://k3s.io/), the reason being that k3s is minimalistic, it is easy to set up while still being scalable/configurable + is known to work well with ARM servers (which I'm using).

![](/img/projects/k3s-on-hetzner/10xEngineer.png)

## Terraform resource provisioning 

With the 2 choices out of the way, it's time set up the infrastructure resources.
For this we can leverage Terraform to do the heavy lifting / provisioning for us.

If you just want the terraform code, you can check it out here on my [GitHub](https://github.com/vincentbockaert/tf-hcloud-infra).

_Note: this repository could contain more than is required for a simple k3s setup, as it manages all of my running Hetzner resources. (With some reasonable exceptions)_

### Setting up the network

By default when you create Virtual Machines in Hetzner, you can assign public IP addresses, which we will still use.
Nevertheless, for inter-node or inter-app communication we want to remain within an internal network.

For this reason, we create a main network with 1 subnet dedicated to k3s nodes.

```terraform
# vpc.tf
resource "hcloud_network" "main" {
  ip_range = "10.0.0.0/16"
  name     = "main-${terraform.workspace}"
  labels = {
    "Environment" = "${terraform.workspace}"
  }
}

resource "hcloud_network_subnet" "k3s" {
  ip_range     = "10.0.1.0/24"
  network_id   = hcloud_network.main.id
  network_zone = "eu-central"
  type         = "cloud"
}
```

### Provisioning servers

Now that we have a network, we can start by adding some k3s server nodes to it:

```terraform
resource "hcloud_server" "k3s_server" {
  count       = local.k3s_server_count
  name        = "server${count.index}.k3s.${terraform.workspace}.hc.vincentbockaert.xyz"
  server_type = "cax11"
  image       = "debian-12"
  datacenter  = "nbg1-dc3"
  user_data   = data.template_cloudinit_config.k3s_server[count.index].rendered
  labels = {
    "Environment" = terraform.workspace
    "Arch"        = "ARM64"
    "NodeNumber"  = count.index
    "k3s"         = "server" # used for applying firewall rules
  }
  ssh_keys = [
    hcloud_ssh_key.this.id
  ]
  public_net {
    ipv4_enabled = true
    ipv6_enabled = true
  }
}

resource "hcloud_server" "k3s_agent" {
  count       = local.k3s_agent_count
  name        = "agent${count.index}.k3s.${terraform.workspace}.hc.vincentbockaert.xyz"
  server_type = "cax11"
  image       = "debian-12"
  datacenter  = "nbg1-dc3"
  user_data   = data.template_cloudinit_config.k3s_agent[count.index].rendered
  labels = {
    "Environment" = terraform.workspace
    "Arch"        = "ARM64"
    "NodeNumber"  = count.index
    "k3s"         = "agent" # used for applying firewall rules
  }
  ssh_keys = [
    hcloud_ssh_key.this.id
  ]
  public_net {
    ipv4_enabled = true
    ipv6_enabled = true
  }
}

resource "hcloud_ssh_key" "this" {
  name       = "default"
  public_key = var.defaultSSHPublicKey
}
```

Some explanation for the above code, it essentially defines 3 resources
- 1 [hcloud_server](https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs/resources/server) for k3s server nodes (think of them as management nodes for kubernetes)
    - multiple instances are made using the `count` operator.
- 1 [hcloud_server](https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs/resources/server) for k3s agent nodes (dedicated k3s nodes, only able run workloads, does not have a management api)
    - multiple instances can be made using the `count` operator, or none as server nodes can also run workloads, thus one might not need agent nodes when at small scale
- 1 [ssh key](https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs/resources/ssh_key) which is added to the servers on boot

For the server configuration:
- enabling ipv4/ipv6 as needed since NAT gateways, _like in aws_, don't exist in Hetzner
    - I still recommend enabling IPv4, _despite the extra cost_, as otherwise you can't pull images from DockerHub, can't clone from GitHub, etc.
- a cloud-init file is passed to `user_data` to set up
    - a dedicated sudo user
    - block root over ssh 
    - and configure CA Signed Trusted Host SSH keys, see more info [here]({{< ref "/projects/openssh-ca-signed-host-keys.md" >}})
- some labels are added identifying their use for "k3s", which is used in the firewall resource to know on which servers to apply the rules to

As mentioned, the `count` operator is used to specify how many servers we want, which is passed along using a _locals block_

```terraform
locals {
  k3s_server_count = 3
  k3s_agent_count  = 0
}
```

The servers are almost ready to be created, but we can't forget to attach them to the internal network:

```terraform
resource "hcloud_server_network" "k3s_agent_subnet_attachment" {
  count     = local.k3s_agent_count
  server_id = hcloud_server.k3s_agent[count.index].id
  subnet_id = hcloud_network_subnet.k3s.id
}

resource "hcloud_server_network" "k3s_server_subnet_attachment" {
  count     = local.k3s_server_count
  server_id = hcloud_server.k3s_server[count.index].id
  subnet_id = hcloud_network_subnet.k3s.id
}
```

And of course a basic firewall should be applied, in this case simply allowing ssh connections from anywhere _(though I highly recommend limiting this)_.

```terraform
resource "hcloud_firewall" "k3s" {
  name = "k3s nodes inbound"
  // ssh from anywhere
  rule {
    direction = "in"
    protocol  = "tcp"
    port      = "22"
    source_ips = [
      "0.0.0.0/0",
      "::/0"
    ]
  }
  
  apply_to {
    # https://docs.hetzner.cloud/#label-selector
    label_selector = "k3s"
  }
}
```

For internal networking, there is no possibility to put a hcloud_firewall to use on internal networking, they only serve a purpose for public networking.
Thus, all _(internal)_ traffic between servers on the same network is allowed.

Additionally, you can use some niceties when applying this terraform code, to output the IP address information:

```terraform
output "server_names" {
  value = concat(
    [for node in hcloud_server.k3s_server : "${node.name} ==> AAA ==> ${node.ipv6_address}"],
    [for node in hcloud_server.k3s_server : "${node.name} ==> A ==> ${node.ipv4_address}"],
    [for node in hcloud_server.k3s_agent : "${node.name} ==> AAA ==> ${node.ipv6_address}"],
    [for node in hcloud_server.k3s_agent : "${node.name} ==> A ==> ${node.ipv4_address}"]
  )
}
```

### Provisioning k3s onto the servers

With the servers created, we can begin deploying k3s onto them.
Luckily for us, the k3s team has an excellent script to install k3s which after installation can still be modified, started/stopped using the installed systemd configuration.

For now, connecting to the first node, we run the initialization with the parameters:

- `--cluster-init`
    - _only needed once to of course initialize a cluster_
- `--disable-cloud-controller`
    - _as we will deploy a custom cloud controller in order to use Hetzner Cloud Load Balancers from within the cluster_
- `--flannel-iface`
    - _rkube networking will happen using Flannel on the existing hcloud network, thus we requires the interface name attached to this internal network_
- `--kubelet-arg="cloud-provider=external"`
    - _needed in order to use the Hetzner Cloud Load Balancer later on_
- `--secrets-encryption`
    - to ensure encryption at rest of kubernetes secrets using AES-CBC
- `--disable=traefik`
    - by default k3s deploys with traefik as Ingress Controller, I much prefer rolling our own version of Traefik/Nginx Ingress/etc
- `--tls-san='server0.k3s.prod.hc.vincentbockaert.xyz'`
    - adds SubjectAlternativeNames to the certificate of the k3s master api endpoint, needed if you want to call the api over these DNS names, i.e. with `kubectl`
- `--token='SECRET_HERE'`
    - _a secure token which nodes can use to join your cluster, so be sure to never leak it_

```bash
curl -sfL https://get.k3s.io | sh -s - server \
    --cluster-init \
    --disable-cloud-controller \
    --node-name="$(hostname -f)" \
    --flannel-iface=enp7s0 \
    --kubelet-arg="cloud-provider=external" \
    --secrets-encryption \
    --disable=traefik \
    --tls-san='server0.k3s.prod.hc.vincentbockaert.xyz' \
    --tls-san='server1.k3s.prod.hc.vincentbockaert.xyz' \
    --tls-san='server2.k3s.prod.hc.vincentbockaert.xyz' \
    --tls-san='master.k3s.prod.hc.vincentbockaert.xyz' \
    --token='SECRET_HERE'
```

Next up, you'll want to have the other 2 servers join the cluster as well using this slightly different snippet on them:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
	--server https://PRIVATE_IP_OF_FIRST_SERVER_NODE:6443 \
    --disable-cloud-controller \
    --node-name="$(hostname -f)" \
    --flannel-iface=enp7s0 \
    --kubelet-arg="cloud-provider=external" \
    --secrets-encryption \
    --disable=traefik \
    --tls-san='server0.k3s.prod.hc.vincentbockaert.xyz' \
    --tls-san='server1.k3s.prod.hc.vincentbockaert.xyz' \
    --tls-san='server2.k3s.prod.hc.vincentbockaert.xyz' \
    --tls-san='master.k3s.prod.hc.vincentbockaert.xyz' \
    --token='SERVER_HERE'
```

Lastly, once done, we can verify the nodes are communicating properly using the following:

```bash
sudo kubectl get nodes
# to fetch a starting kubeconfig:
sudo cat /etc/rancher/k3s/k3s.yaml
```

If you want to, you can copy the k3s.yaml file to your local machine and continue using it from there _(slightly modifying it first to point to your DNS name)_.
Your `~/.kube/config` would then look similar to the following:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: blahblah
    server: https://server0.k3s.prod.hc.vincentbockaert.xyz:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: blahblah
    client-key-data: blahblah
```

### Integrating cluster with Hetzner Cloud

With the working cluster, we can get started on enabling integration with the _native_ Hetzner Cloud resources, to get close to a _managed_ solution.
This is enabled by using 2 amazing helm charts provided by Hetzner, though oddly, not really advertised:
- [`hcloud-cloud-controller-manager`](https://github.com/hetznercloud/hcloud-cloud-controller-manager?tab=readme-ov-file#features)
- [`hcloud-csi`](https://github.com/hetznercloud/csi-driver)

The former comes with the following features _(description taken straight from the their GitHub repo)_:
- Node
    - Updates your `Node` objects with information about the server from the Cloud & Robot API.
    - Instance Type, Location, Datacenter, Server ID, IPs.
- Node Lifecycle:
    - Cleans up stale `Node` objects when the server is deleted in the API.
- Routes (if enabled):
    - Routes traffic to the pods through Hetzner Cloud Networks. Removes one layer of indirection in CNIs that support this.
    - _In other words, IP assignments are coming from the Hetzner network :)_
- Load Balancer:
    - Watches Services with `type: LoadBalancer` and creates Hetzner Cloud Load Balancers for them, adds Kubernetes Nodes as targets for the Load Balancer.

While the latter, `hcloud-csi`, is a Container Storage Interface driver enabling the use of ReadWriteOnce volumes, i.e. we can use Kubernetes PVC with Hetzner Cloud Volumes as the backing storage.

Luckily for us, Hetzner Cloud has provided 2 excellent helm charts to implement this kind of integration for us.

However, to establish communication with the API's from Hetzner, some extra configuration is needed.
For starters a kubernetes secret is needed, [containing a api token for hcloud](https://docs.hetzner.com/cloud/api/getting-started/generating-api-token/), which we'll put in a namespace dedicated to this kind of "_operations_" work.

```bash
kubectl create ns ops-tools
kubectl -n ops-tools create secret generic hcloud --from-literal="token=$HCLOUD_TOKEN" --from-literal=network=$HCLOUD_NETWORK_NAME_ID
```

With this secret in place, all that's still needed is to apply the helm charts to our cluster.
One could do this 1-by-1, managing them separately, but I chose to make my own Chart `ops-tools` with the hcloud charts as dependencies + an nginx-ingress.

```yaml
# Chart.yaml
apiVersion: v2
name: ops-tools
version: 1.2.1
description: Helm chart for operations tools and infrastructure items
dependencies:
  # prerequisite: kubectl -n kube-system create secret generic hcloud --from-literal=token=<hcloud API token>
  - name: hcloud-cloud-controller-manager
    repository: https://charts.hetzner.cloud
    version: 1.19.0

  # allows using Hetzner Cloud Volumes as persistent volumes
  - name: hcloud-csi
    repository: https://charts.hetzner.cloud
    version: 2.6.0

  - name: ingress-nginx
    repository: https://kubernetes.github.io/ingress-nginx
    version: 4.9.0
    condition: ingress-nginx.enabled
```

With as values the following:

```yaml
# values.yaml
hcloud-cloud-controller-manager:
  networking:
    enabled: true
    clusterCIDR: 10.42.0.0/16 # This is the default cluster CIDR for k3s

ingress-nginx:
  enabled: false # only enable after hcloud-cloud-controller-manager is installed
  controller:
    name: kubelb
    service:
      enabled: true
      single: true # otherwise one is created for TCP and one for UDP
      type: LoadBalancer
      annotations: 
        load-balancer.hetzner.cloud/location: nbg1 
        load-balancer.hetzner.cloud/name: kubelb
        load-balancer.hetzner.cloud/use-private-ip: "true"
        load-balancer.hetzner.cloud/type: lb11 # smallest type allowed in hcloud
```

This is applied using:

```bash
helm dep update ./ops-tools # assuming Chart.yaml and values.yaml are under this folder
helm upgrade -n ops-tools --install -f ./ops-tools/values.yaml ops-tools ./ops-tools/ 
```

It's important to make sure that the `hcloud-cloud-controller-manager` is installed before installing `ingress-nginx`, therefore I highly recommend splitting it up in 2 update steps, by using the `enabled: false/true`.

If you don't do this, you can end up in a state where, the nginx-ingress doesn't use your LoadBalancer IP but instead tries to use the public IP's of the k3s nodes, which won't work regardless.

Furthermore, when creating the ingress-nginx, it's important to add the annotations specific to hcloud, especially making sure that `location` is the same as the one from your nodes. 
More elaboration on each of the options can be found at the [golang package](https://pkg.go.dev/github.com/hetznercloud/hcloud-cloud-controller-manager/internal/annotation#Name).

If you want the complete helm code, you can find it here on my [GitHub](https://github.com/vincentbockaert/helm-hcloud).

That being said, with all of this, a real cluster is created with most of the bells and whistles one might expect from a real managed solutions.

The only nuisance being, that when you add a new node you will need to run the k3s script after provisioning the resource with terraform, but who knows maybe I can automate that too, perhaps a future post? 😎

