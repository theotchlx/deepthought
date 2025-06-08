# Deepthought installation and configuration

Deepthought is my new homelab cluster, running Kubernetes on Talos, with Rook for Ceph storage. It runs on old machines and laptops, bare metal, with either Kubevirt or Kata Containers for virtualization.

Kubernetes nodes don't live in VMs, because there is no need for virtual isolation (this is not a managed Kubernetes service, but a cluster over my own compute resources).

Currently, the cluster consists three nodes, one being the control plane. I plan to add more nodes later, and have a three-nodes control plane. Currently, the control plane also runs the Etcd cluster, and is schedulableable for workloads.

The machines will be referred by their node name and private IP addresses, as:

- Gateway router, IP: 192.168.1.1
- Node name: talos-cp-zaphod, IP: 192.168.1.32
- Node name: talos-work-trillian, IP: 192.168.1.24
- Node name: talos-work-marvin, IP: 192.168.1.48

Each machine has one SSD for the OS, and one or more HDDs for data storage. The SSDs are used for the OS and Kubernetes system static pods, while the HDDs are used for Rook Ceph OSDs.

## Talos

Talos is a Linux distribution designed for running Kubernetes clusters. It immutable, managed declaratively and remotely, and secure through its minimalism (~10 binaries, ~10 running processes).

### Talos installation

Retrieve the ISO image from [the v1.10.3 release](https://github.com/siderolabs/talos/releases/download/v1.10.3/metal-amd64.iso). Do NOT use ventoy to boot the ISO, as it is incompatible with Talos (this took ~2h debug).

Boot the ISO on the machine that will serve as the first control plane node. Retrieve its IP if DHCP or statis IP is configured - else set its IP.  
In my case, I configured statis IP allocation based on the MAC address of the machines. That means the network interface will always get the same IP address, even across reboots/outages.

Once the control plane is booted, generate and retrieve the Talos configuration file. I named it "deepthought" but this name is only local to your laptop.

```bash
talosctl gen config deepthought https://192.168.1.32:6443
```

This generates three files:
- `talosconfig`, for accessing the cluster.
- `controlplane.yaml`, the declarative file generated with happy defaults to install a control plane node.
- `worker.yaml`, the declarative file generated with happy defaults to install a worker node.

Since `controlplane.yaml` was generated from the (ISO-) booted controlplane, that means the configuration is going to work. You may review it, mainly on the disks and network side. It was OK for me, installing the OS on the SSD.

We may now apply the configuration, to install Talos on the first control plane node:

```bash
talosctl apply-config --insecure -n 192.168.1.32 --file controlplane.yaml
```

> Note that port 50000 must be available for TCP. This is usually the case.

The OS will reboot and you can remove the USB. The Talos (and future Kubernetes) control plane is now installed on the machine, the machine will now boot to Talos on reboot.

Boot the other machines (worker nodes) the same way. When the ISO is booted, deploy the worker node configuration (make sure it fits the disk configuration and network configuration):

```bash
talosctl apply-config --insecure -n 192.168.1.48 --file worker.yaml
```

```bash
talosctl apply-config --insecure -n 192.168.1.24 --file worker.yaml
```

Let's run a few tests to ensure the Talos nodes are up and running correctly (a Talos dashboard also shows on the machines screens).

Get talos version on the machines:

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.32 -e 192.168.1.32 version
```

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.48 -e 192.168.1.48 version
```

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.24 -e 192.168.1.24 version
```

A few explanations of the commands:

- We must specify the `talosconfig` file, as Talos machines are not reachable through SSH, but through the Talos API, that is configured with generated TLS certificates. Note that the root CA has an expiration date of 1 year.
- Nodes and endpoints: Talos uses machines as "endpoints" to access other machines, the "nodes", on which we wish to run the commands.

### Kubernetes installation

Now, we can proceed to bootstrap the Kubernetes cluster from the control plane node:

```bash
talosctl bootstrap --nodes 192.168.1.32 --endpoints 192.168.1.32 --talosconfig=./talosconfig
```

This will also deploy an etcd cluster on the control plane node, and configure the Kubernetes API server to use it.

We can now retrieve the Kubernetes client configuration file (`~/.kube/config`) to access the cluster from our laptop:

```bash
talosctl kubeconfig --nodes 192.168.1.32 --endpoints 192.168.1.32  --talosconfig=./talosconfig
```

This updates your ~/.kube/config file according to the control plane config, and sets the default context to the Talos cluster (earlier named "deepthought").

Let's run a few more tests to ensure the Kubernetes cluster is up and running correctly, and that everything is working as expected.

Get info on the nodes:

```bash
kubectl get nodes
```

Explore the Talos cluster:

Make sure the Talos cluster is healthy and running correctly.

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.32 -e 192.168.1.32 health
```

Get the Talos dashboard remotely.

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.32 -e 192.168.1.32 dashboard
```

Get information on the disks.

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.32 -e 192.168.1.32 get disks
```

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.48 -e 192.168.1.48 get disks
```

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.24 -e 192.168.1.24 get disks
```

For the Ceph OSDs to get dispatched later, make sure the appropriate disks (not the OS disks) are clean with:

```bash
talosctl --talosconfig=./talosconfig --nodes 192.168.1.<ip> -e 192.168.1.<ip> wipe disk <device name>
```

---

## Rook Ceph

Configuring a Ceph cluster is complex. You may use this section as a overview of how to install the Rook operator and Ceph cluster on Talos using Helm, but the Ceph cluster configuration will heavily depend on your hardware and requirements.

### Rook operator installation

let's start by configuration and deploying the Rook operator with Helm.

```bash
mkdir rook-ceph-operator ;cd rook-ceph-operator
curl https://raw.githubusercontent.com/rook/rook/refs/heads/master/deploy/charts/rook-ceph/values.yaml > ./values.yaml
```

There should not be much to configure in the `values.yaml` file of the operator. See [the documentation](https://rook.io/docs/rook/latest/Helm-Charts/operator-chart/#configuration) to configure the operator.

Deploy the operator:

```
helm repo add rook-release https://charts.rook.io/release
helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph -f values.yaml
```

Talos configured a production-grade Kubernetes cluster, which means the PodSecurity Policy is set to `restricted` by default. As such, we need to configure the rook-ceph namespace for the future Ceph CSI drivers to work: they need to be privileged, to access host devices. Label the namespace as such:

```bash
kubectl label namespace rook-ceph pod-security.kubernetes.io/enforce=privileged
```

### Rook Ceph cluster installation

Let's configure and deploy the Ceph cluster using Rook and Helm.

```bash
cd .. ;mkdir rook-ceph-cluster ;cd rook-ceph-cluster
curl https://raw.githubusercontent.com/rook/rook/refs/heads/master/deploy/charts/rook-ceph-cluster/values.yaml > ./values.yaml
```

> /!\ This is a complex step. The retrieved default configuration is not a happy default, and HAS to be configured THOROUGHLY, or it will not work.
> If you broke your Ceph cluster by botching the configuration, follow [this documentation](https://rook.io/docs/rook/latest/Getting-Started/ceph-teardown/) to clean up your cluster. It is not easy.

See [the documentation](https://rook.io/docs/rook/latest/Helm-Charts/ceph-cluster-chart/#installing) for more details.

Deploy the Ceph cluster:

```bash
helm repo add rook-release https://charts.rook.io/release
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster \
   --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster -f values.yaml
```

I plan to add more nodes in the future. To add worker nodes to the cluster:

- Follow the Talos setup steps with worker nodes commands only.
- Update rook-ceph-cluster values.yaml to add the disks.

