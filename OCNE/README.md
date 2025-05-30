# Vagrant project to set up Oracle Cloud Native Environment on Oracle Linux 8

Due to major changes in OCNE 2.0 release this Vagrant project is not updated yet for OCNE 2 and deploys the latest OCNE 1.* release.

This Vagrant project will deploy and configure the following components:

- One or more control plane nodes (one by default, 3 in HA mode)
- One or more worker nodes (2 by default)
- An optional operator node for the Oracle Cloud Native Environment
Platform API Server and Platform CLI tool (default is to install these
components on the first control plane node)

If you enable multiple control plane nodes, an operator node is automatically deployed
to provide egress routing for the cluster.

All control plane and worker nodes will have the Oracle Cloud Native
Environment Platform Agent installed and configured to communicate with the
Platform API Server on the operator node.

The installation includes the Kubernetes module for Oracle Cloud
Native Environment which deploys Kubernetes [1.25.7](https://docs.oracle.com/en/operating-systems/olcne/1.6/relnotes/components.html#components)
configured to use the CRI-O runtime interface. Two runtime engines are installed,
[runC](https://docs.oracle.com/en/operating-systems/olcne/1.6/runtimes/runc.html#runc)
and (now deprecated) [Kata](https://docs.oracle.com/en/operating-systems/olcne/1.6/runtimes/kata.html#kata) Containers.

You may set your cluster networking to Flannel (default), Calico or Multus on top of either Calico or Flannel.

You may optionally enable the deployment of Istio, MetalLB or Gluster (deprecated)
modules. Note that Helm is now installed automatically.

_Note:_ Kata Containers (now deprecated) require Intel hardware virtualization support and
will not work in a VirtualBox guest until nested virtualization support is
released for Intel CPUs.

## Prerequisites

1. Read the [prerequisites in the top level README](../README.md#prerequisites)
to set up Vagrant with either VirtualBox or KVM
2. [vagrant-env](https://github.com/gosuri/vagrant-env) plugin is optional but
makes configuration much easier

## Quick start

1. Clone this repository `git clone https://github.com/oracle/vagrant-projects`
2. Change into the `vagrant-projects/OCNE` directory
3. Run `vagrant up`

Your Oracle Cloud Native Environment is ready!

From any control plane node (e.g. controlplane1) you can check the status of the cluster (as
the `vagrant` user). E.g.:

- `kubectl cluster-info`
- `kubectl get nodes`
- `kubectl get pods --namespace=kube-system`

## Accessing the Kubernetes Dashboard

By default, the Kubernetes Dashboord does not allow non-HTTPS connections from
any source except `localhost`/`127.0.0.1`. If you want to be able to connect
to the Dashboard from a browser on your Vagrant host, you will need to set
`BIND_PROXY` to `true` in your `.env.local` file.

To access the Kubernetes Dashboard, remember to use `localhost` or `127.0.0.1`
in the URL, i.e. <http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/>.
To obtain token from any Control plane node, you may run: `kubectl -n kubernetes-dashboard get secret -o=jsonpath='{.items[?(@.metadata.annotations.kubernetes\.io/service-account\.name=="kubernetes-dashboard")].data.token}' | base64 --decode`

## About the `Vagrantfile`

The VMs communicate via a private network:

- Controller node IP: 192.168.99.100 (if `STANDALONE_OPERATOR=true`)
- Control plane node _i_: 192.168.99.(100+ _i_ ) / controlplane *_i_* .vagrant.vm
- Worker node _i_: 192.168.99.(110+ _i_ ) / worker *_i_* .vagrant.vm
- Control plane Virtual IP: 192.168.99.99 (if `MULTI_CONTROL_PLANE=true`)
- LoadBalancer IPs: 192.168.99.240 - 192.168.99.250 (if `DEPLOY_METALLB=true`)

## Configuration

The `Vagrantfile` can be used _as-is_; there are a couple of parameters you
can set to tailor the installation to your needs.

### How to configure

There are several ways to set parameters:

1. Update the `Vagrantfile`. This is straightforward; the downside is that you
will lose changes when you update this repository.
2. Use environment variables. Might be difficult to remember the parameters
used when the VM was instantiated.
3. Use the `.env`/`.env.local` files (requires
[vagrant-env](https://github.com/gosuri/vagrant-env) plugin). Configure
your cluster by editing the `.env` file; or better copy `.env` to `.env.local`
and edit the latter one, it won't be overridden when you update this repository
and it won't mark your git tree as changed (you won't accidentally commit your
local configuration!)

Parameters are considered in the following order (first one wins):

1. Environment variables
2. `.env.local` (if [vagrant-env](https://github.com/gosuri/vagrant-env) plugin
is installed)
3. `.env` (if [vagrant-env](https://github.com/gosuri/vagrant-env) plugin
is installed)
4. `Vagrantfile` definitions

### VM parameters

- `VERBOSE` (default: `false`): verbose output during VM deployment.
- `WORKER_CPUS` (default: `1`):  Provision Worker Node with 1 vCPU.
- `WORKER_MEMORY` (default: `1024`): Provision Worker Node with 1GB memory.
- `CONTROL_PLANE_CPUS` (default: `2`): At least 2 vCPUS are required for Control plane Nodes.
- `CONTROL_PLANE_MEMORY` (default: `2048`): At least 1700MB are required for Control plane Nodes.
- `OPERATOR_CPUS` (default: `1`): Only applicable if `STANDALONE_OPERATOR=true` or `MULTI_CONTROL_PLANE=true`.
- `OPERATOR_MEMORY` (default: `1024`): Only applicable if `STANDALONE_OPERATOR=true` or `MULTI_CONTROL_PLANE=true`.
- `VB_GROUP` (default: `OCNE`): group all VirtualBox VMs under this label.
- `EXTRA_DISK` (default: `false`): Creates an extra disk (`/dev/sdb`) on Worker nodes that can be used for GlusterFS for Kubernetes Persistent Volumes

### Cluster parameters

- `STANDALONE_OPERATOR` (default: `false` unless `MULTI_CONTROL_PLANE=true`): create
a separate VM for the operator node -- default is to install the operator
components on the (first) control plane node.
- `MULTI_CONTROL_PLANE` (default: `false`): multi-control-plane setup. Deploy 3 control planes in
HA mode.
- `NB_WORKERS` (default: `2`): number of worker nodes to provision.
At least one worker node is required.
- `BIND_PROXY` (default: `false`): bind the kubectl proxy port (8001) from the
(first) control plane to the Vagrant host. This is required if you want to access the
Kubernetes Dashboard from a browser on your host.
__Note__: you only need this if you want to expose the kubectl proxy to other
hosts in your network.
- `DEPLOY_CALICO` (default: `false`): deploys the Calico networking module with custom configuration `calico-config.yaml`.
- `DEPLOY_MULTUS` (default: `false`): deploys the Multus networking module with custom configuration `multus-config.yaml`.
- `DEPLOY_HELM` (default: `false`): deploys the Helm module (deprecated).
- `DEPLOY_ISTIO` (default: `false`): deploys the Istio modules.
- `DEPLOY_METALLB` (default: `false`): deploys the MetalLB module.
- `DEPLOY_GLUSTER` (default: `false`): deploys the Gluster module (deprecated).
__Note__: if `NB_WORKERS` is less than `3`, the `hyperconverged` `storageclass`
is patched to adjust the number of Gluster replicas accordingly.
__Note__: This provisioning script also installs Heketi on the operator node.

### Repositories

- `YUM_REPO` (default: none): additional yum repository to consider
(e.g. local repo)
- `OCNE_DEV` (default: `false`): whether to enable the Oracle Cloud
Native Environment developer channel.
- `REGISTRY_OCNE` (default: `container-registry.oracle.com/olcne`): Container
registry for Oracle Cloud Native Environment images.

For performance reasons, we recommend using the closest Oracle Container Registry mirror to your region. A list of available regions can be found on the [Regions and Availability Domains](https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) page of the Oracle Cloud Infrastructure documentation.

To specify an Oracle Container Registry mirror, either edit the `Vagrantfile` or install the vagrant-env plugin and create a `.env.local` file that specifies the mirror.

The following syntax can be used to specify a mirror:

- `container-registry-<region_name>.oracle.com`, e.g. `container-registry-sydney.oracle.com/olcne`
- `container-registry-<region_identifier>.oracle.com`, e.g. `container-registry-ap-sydney-1.oracle.com/olcne`
- `container-registry-<region_key>.oracle.com`, e.g. `container-registry-syd.oracle.com/olcne`

 All regions are available at <https://docs.cloud.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm>

### Advanced Parameters

Danger zone!
Mainly used for development.

- `NB_CONTROL_PLANES` (default: none): override number of control planes to deploy. Requires `MULTI_CONTROL_PLANE=true` to function properly.
- `SUBNET` (default: `192.168.99`): Set the VM provider host-only / private network subnet.
- `UPDATE_OS` (default: false): Runs `dnf -y update` on the VM.

## Optional plugins

When installed, this `Vagrantfile` will make use of the following third party Vagrant plugins:

- [vagrant-env](https://github.com/gosuri/vagrant-env): loads environment
variables from .env files;
- [vagrant-hosts](https://github.com/oscar-stack/vagrant-hosts): maintains
`/etc/hosts` for the guest VMs;
- [vagrant-proxyconf](https://github.com/tmatilai/vagrant-proxyconf): set
proxies in the guest VMs if you need to access the Internet through proxy. See
plugin documentation for the configuration.

To install Vagrant plugins run:

```shell
vagrant plugin install <name>...
```

## Product Documentation

- [Oracle Cloud Native Environment: Getting Started](https://docs.oracle.com/en/operating-systems/olcne/1.6/start/)
- [Oracle Cloud Native Environment: Container Orchestration](https://docs.oracle.com/en/operating-systems/olcne/1.6/orchestration/)
- [Oracle Cloud Native Environment: Container Runtimes](https://docs.oracle.com/en/operating-systems/olcne/1.6/runtimes/)
- [Oracle Cloud Native Environment: Platform CLI](https://docs.oracle.com/en/operating-systems/olcne/1.6/olcnectl/)

## Feedback

Please provide feedback of any kind via GitHub issues on this repository.

## Contributing

See [CONTRIBUTING](../CONTRIBUTING.md) for details.
