+++
title = "KubeVirt"
date = 2024-10-01T14:46:15+02:00
weight = 5

+++

## Architecture

![KubeVirt Cloud Provider Architecture](images/architecture.png?classes=shadow,border "KubeVirt Cloud Provider Architecture")

## Installation And Configuration

### Requirements

A Kubernetes cluster (KubeVirt infrastructure cluster), which consists of nodes that **have a hardware virtualization support** with at least:
* 3 Bare Metal Server
* CPUs: Minimum 8-core for testing; minimum 16-core or more for production
* Memory: Minimum 32 GB for testing; minimum 64 GB or more for production
* Storage: Minimum 100 GB for testing; minimum 500 GB or more for production

Software requirement:
* KubeOne = 1.7 or higher
* KubeOVN = 1.12 or higher or Canal = 3.26 or higher
* KubeVirt = 1.2.2
* Containerized Data Importer (CDI) = v1.60

The cluster version must be in the scope of [supported KKP Kubernetes clusters]({{< ref "../../../tutorials-howtos/operating-system-manager/compatibility/#kubernetes-versions" >}})
and it must be in the [KubeVirt Support Matrix](https://github.com/kubevirt/sig-release/blob/main/releases/k8s-support-matrix.md).

### Kubernetes And KubeVirt Installation

First use KubeOne, to set up a highly-available Kubernetes cluster on bare metal.
Refer to the [KubeOne documentation](/kubeone/latest/tutorials/creating-clusters-baremetal/) for details on how to use it.

Follow [KubeVirt](https://kubevirt.io/user-guide/operations/installation/#installation) and [Containerized Data Importer](https://kubevirt.io/user-guide/operations/containerized_data_importer/#install-cdi)
documentation to find out how to install them.

We require the following KubeVirt configuration:
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    developerConfiguration:
      featureGates:
      - ExperimentalIgnitionSupport
      - DataVolumes
      - LiveMigration
      - CPUManager
      - CPUNodeDiscovery
      - Sidecar
      - Snapshot
      - HotplugVolumes
```

It is not required to have any specific Containerized Data Importer configuration as long the main storage is **not** local disks.
Otherwise, CDI must be configured with `HonorWaitForFirstConsumer` feature gate.

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: CDI
metadata:
  name: cdi
spec:
  config:
    featureGates:
    - HonorWaitForFirstConsumer
```

{{% notice note %}}
Refer to this [document](https://github.com/kubevirt/kubevirt/blob/main/docs/localstorage-disks.md)
to learn more about how KubeVirt handles local disks storage.
{{% /notice %}}

{{% notice warning %}}
Currently, it is not recommended to use local or any topology constrained storage due to [the issue with kubevirt csi driver](https://github.com/kubevirt/csi-driver/issues/66)
{{% /notice %}}

### Configure KKP With KubeVirt

Once you have Kubernetes with all needed components, the last thing is to configure KubeVirt datacenter on seed.

We allow to configure:
* `customNetworkPolicies` - Network policies that are deployed on the infrastructure cluster (where VMs run).
  * Check [Network Policy documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource) to see available options in the spec.
  * Also check a [common services connectivity issue](#i-created-a-load-balancer-service-on-a-user-cluster-but-services-outside-cannot-reach-it) that can be solved by a custom network policy.
* `ccmZoneAndRegionEnabled` - Indicates if region and zone labels from the cloud provider should be fetched. This field is enabled by default and should be disabled if the infra kubeconfig that is provided for KKP has no permission to access cluster role resources such as node objects.
* `dnsConfig` and `dnsPolicy` - DNS config and policy which are set up on a guest. Defaults to `ClusterFirst`.
  * You should set those fields when you suffer from DNS loop or collision issue. [Refer to this section for more details.](#i-discovered-a-dns-collision-on-my-cluster-why-does-it-happen)
* `images` - Images for Virtual Machines that are selectable from KKP dashboard.
  * Set this field according to [supported operating systems]({{< ref "../../compatibility/os-support-matrix/" >}}) to make sure that users can select operating systems for their VMs.
* `infraStorageClasses` - Storage classes that are initialized on user clusters that end users can work with.
  * `isDefaultClass` - If true, the created StorageClass in the tenant cluster will be annotated with.
  * `labels` - Is a map of string keys and values that can be used to organize and categorize (scope and select) objects. May match selectors of replication controllers and services.
  * `regions` - Represents a larger domain, made up of one or more zones. It is uncommon for Kubernetes clusters to span multiple regions.
  * `volumeBindingMode` - indicates how PersistentVolumeClaims should be provisioned and bound. When unset, VolumeBindingImmediate is used.
  * `volumeProvisioner` - The field specifies whether a storage class will be utilized by the infra cluster csi driver where the Containerized Data Importer (CDI) can use to create VM disk images or by the KubeVirt CSI Driver to provision volumes in the user cluster. If not specified, the storage class can be used as a VM disk image or user clusters volumes.
    * `infra-csi-driver` - When set in the infraStorageClass, the storage class can be listed in the UI while creating the machine deployments and won't be available in the user cluster.
    * `kubevirt-csi-driver` - When set in the infraStorageClass, the storage class won't be listed in the UI and will be available in the user cluster.
  * `zones` - Represent a logical failure domain. It is common for Kubernetes clusters to span multiple zones for increased availability.
* `namespacedMode(experimental)` - Represents the configuration for enabling the single namespace mode for all user-clusters in the KubeVirt datacenter.
* `vmEvictionStrategy` - Indicates the strategy to follow when a node drain occurs. If not set the default value is External and the VM will be protected by a PDB. Currently, we only support two strategies, `External` or `LiveMigrate`.
  * `LiveMigrate`: the VirtualMachineInstance will be migrated instead of being shutdown.
  * `External`: the VirtualMachineInstance will be protected by a PDB and `vmi.Status.EvacuationNodeName` will be set on eviction. This is mainly useful for machine-controller which needs a way for VMI's to be blocked from eviction, yet inform machine-controller that eviction has been called on the VMI, so it can handle tearing the VMI down.
* `csiDriverOperator` - Contains the KubeVirt CSI Driver Operator configurations, where users can override the default configurations of the csi driver.
  *  `overwriteRegistry`: overwrite the images registry for the csi driver daemonset that runs in the user cluster. 

{{% notice note %}}
The `infraStorageClasses` pass names of KubeVirt storage classes that can be used from user clusters.
{{% /notice %}}

{{% notice warning %}}
The `namespacedMode` feature is highly experimental and should never be used in production environments. Additionally, enabling this mode in an existing KubeVirt setup utilized by KKP can cause serious issues, such as storage and networking incompatibilities.
{{% /notice %}}

Refer to this [document](https://github.com/kubermatic/kubermatic/blob/release/v2.26/docs/zz_generated.seed.ce.yaml#L115)
for more details and configuration example.

{{% notice warning %}}
By default, each user cluster is deployed with the **cluster-isolation** Network Policy that allows network communication
only inside the cluster. You should use `customNetworkPolicies` to customize the network rules to your needs.
**Remember that new rules will affect all user clusters.**
{{% /notice %}}

### Setup Monitoring

Install [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) on the KubeVirt cluster.
Then update `KubeVirt` configuration with the following spec:
```yaml
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  monitorNamespace: "<<PROMETHEUS_NAMESPACE>>"
  monitorAccount: "<<PROMETHEUS_SERVICE_ACCOUNT_NAME>>"
```

For more details please refer to this [document](https://kubevirt.io/user-guide/operations/component_monitoring/).

After completing the above setup, you can import the [KubeVirt Dashboard](https://github.com/kubevirt/monitoring/tree/main/dashboards/grafana) to Grafana.
Follow the official [Grafana documentation](https://grafana.com/docs/grafana/latest/dashboards/manage-dashboards/#export-and-import-dashboards
) to learn how to import the dashboard.

## Advanced Settings

### Virtual Machine Templating

We provide a Virtual Machine templating functionality over [Instance Types and Preferences](https://kubevirt.io/user-guide/virtual_machines/instancetypes/).

![Instance Types and Preferences](images/instance-type.png?classes=shadow,border "Instance Types and Preferences")

You can use our standard

Instance Types:
* standard-2 - 2 CPUs, 8Gi RAM
* standard-4 - 4 CPUs, 16Gi RAM
* standard-8 - 8 CPUs, 32Gi RAM

and Preferences (which are optional):
* sockets-advantage - cpu guest topology where number of cpus is equal to number of sockets

or you can just simply adjust the amount of CPUs and RAM of our default template according to your needs.

Additionally, if our templates will not fulfill your requirements then a KubeVirt cluster admin can create customized
instance types and preferences that users can select later. [Read how to add new Instance Types and Preferences.](#how-can-i-add-a-new-virtual-machine-template)

### Virtual Machine Scheduling

KubeVirt can take advantage of Kubernetes inner features to provide an advanced scheduling mechanism to virtual machines (VMs):
- [Kubernetes topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes node affinity/anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

Since KubeVirt VMs are wrapped in pods, the Kubernetes scheduling rules applicable to pods are completely valid for KubeVirt VMs.
This allows you to restrict KubeVirt VMs ([see architecture](#architecture)) to run only on specific KubeVirt infra nodes.

{{% notice note %}}
Note that topology spread constraints and node affinity presets are applicable to KubeVirt infra nodes.
{{% /notice %}}
#### Default Scheduling Behavior

Each Virtual Machine you create has default [topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) applied:

```yaml
maxSkew: 1
topologyKey: kubernetes.io/hostname
whenUnsatisfiable: ScheduleAnyway
```

this allows us to spread Virtual Machine equally across a cluster.

#### Customize Scheduling Behavior

It is possible to change the default behaviour and create your own topology combined with Node Affinity Presets.
You can do it by expanding *ADVANCED SCHEDULING SETTINGS* on the initial nodes dashboard page.

![Instance Types and Preferences](images/scheduling-form.png?classes=shadow,border "Advanced Scheduling Settings")

- `Node Affinity Preset Key` refers to the key of KubeVirt infra node labels.
- `Node Affinity Preset Values` refers to the values of KubeVirt infra node labels.

Node Affinity Preset type can be `hard` or `soft` and refers to the same notion of [Pod affinity/anti-affinity types](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#types-of-inter-pod-affinity-and-anti-affinity):
- `hard`: the scheduler can't schedule the VM  unless the rule is met.
- `soft`: the scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler still schedules the VM.

It gives you a possibility to create your own unique scheduling options that override ours.
For instance, you could avoid creation of Virtual Machines on database nodes etc.

{{% notice note %}}
Note that you can specify a `Node Affinity Preset Key` and leave `Node Affinity Preset Values` empty to constrain the VM to run on KubeVirt infra nodes that have a specific label key (whatever the values are).
{{% /notice %}}

## Network Policy Mode

KKP supports two modes of network policies created in the kubevirt infra cluster when namespaced mode is enabled in enterprise edition. This mode can be configured per provider network in the seed spec `.spec.kubevirt.providerNetwork`. Supported values are `allow` and `deny`.

For instance, if you want to configure `deny` mode then you would have to set:

```yaml
providerNetwork:
  - name: default
    vpcs:
    - name: ovn-cluster
    networkPolicy:
      enabled: true
      mode: deny
```

{{% notice warning %}}
Since all traffic except between worker nodes and traffic between them and the control plane is denied by default when using `deny` mode please take care to configure required nameservers and sources for images and packages for the virtual machines properly by using `customNetworkPolicies` and `nameservers` in the kubevirt datacenter spec.
{{% /notice %}}

## Frequently Asked Questions

### How can I add a new Virtual Machine template?

You can do it by simply creating a new `VirtualMachineClusterInstancetype` and `VirtualMachineClusterPreference` on the KubeVirt infrastructure cluster.
Those resources are cluster scoped meaning all users will see them.

Refer to the [InstanceTypes and Preferences](https://kubevirt.io/user-guide/virtual_machines/instancetypes/#virtualmachineinstancetype) guide for details on how to use it.

### How can I safely drain a bare metal node?

You can do it as with every standard k8s cluster, over `kubectl drain` command.

We implemented a mechanism that will allow you to safely drain a bare-metal node without losing the VM workload.
After running a drain command the VMs running on the node along with their workload will be evicted to different nodes.

{{% notice note %}}
More details on the eviction implementation can be found [here](https://github.com/kubermatic/kubermatic/blob/release/v2.26/docs/proposals/kubevirt-workload-eviction.md).
{{% /notice %}}

{{% notice warning %}}
Remember, the responsibility of making sure that the workload can be evicted lies on you.
Invalid `PodDisruptionBudget` configuration may block the eviction.
{{% /notice %}}

{{% notice warning %}}
Additionally consider [skipEvictionAfter](https://github.com/kubermatic/machine-controller/blob/main/cmd/machine-controller/main.go#L125-L126)
parameter of Machine Controller that sets the timeout for workload eviction.
**Once exceeded, the VMs will simply be deleted.**
{{% /notice %}}

### I discovered a DNS collision on my cluster. Why does it happen?

Usually it happens when both infrastructure and user clusters points to the same address of NodeLocal DNS Cache servers, even if they have separate server instances running.

Let us imagine that:
* On the infrastructure cluster there is a running NodeLocal DNS Cache under 169.254.20.10 address.
* Then we create a new user cluster, start a few Virtual Machines that finally gives a fully functional k8s cluster that runs on another k8s cluster.
* Next we observe that on the user cluster there is another NodeLocal DNS Cache that has the same 169.254.20.10 address.
* Since Virtual Machine can have access to subnets on the infra and user clusters (depends on your network policy rules) having the same address of DNS cache leads to conflict.

One way to prevent that situation is to set a `dnsPolicy` and `dnsConfig` rules that Virtual Machines do not copy DNS configuration from their pods and points to different addresses.

Follow [Configure KKP With KubeVirt](#configure-kkp-with-kubevirt) to learn how set DNS config correctly.

### I created a load balancer service on a user cluster but services outside cannot reach it.

In most cases it is due to `cluster-isolation` network policy that is deployed as default on each user cluster.
It only allows in-cluster communication. You should adjust network rules to your needs by adding [customNetworkPolicies configuration]({{< ref "../../../tutorials-howtos/project-and-cluster-management/seed-cluster/" >}})).

For instance, if you need to allow all ingress traffic from `10.200.10.0/24` CIDR to each user cluster then you would have to set:

```yaml
customNetworkPolicies:
  - name: allow-external-traffic
    spec:
      policyTypes:
        - Ingress
      ingress:
        - from:
          - ipBlock:
            cidr: 10.200.10.0/24
```

## Known Issues

### Support of Block Volume Mode

Currently, the KubeVirt CSI driver does not support volumes with block mode therefore you should avoid using this option to mount a PVC to a user cluster.

### Topology Constrained Storage

Due to [the issue](https://github.com/kubevirt/csi-driver/issues/66), it is not recommended to use local or any storage that is constrained by some topology.
You can find more details in the linked issue.

## Migration from KKP 2.21

Kubermatic Virtualization graduates to GA from KKP 2.22!
On the way, we have changed many things that improved our implementation of KubeVirt Cloud Provider.

Just to highlight the most important:
* Safe Virtual Machine workload eviction has been implemented.
* Virtual Machine templating is based on InstanceTypes and Preferences.
* KubeVirt CSI controller has been moved to control plane of a user cluster.
* Users can influence scheduling of VMs over topology spread constraints and node affinity presets.
* KubeVirt Cloud Controller Manager has been improved and optimized.
* Cluster admin can define the list of supported OS images and initialized storage classes.

Additionally, we removed some features that didn't leave technology preview stage, those are:
* Custom Local Disks
* Secondary Disks

{{% notice warning %}}
The official upgrade procedure will not break clusters that already exist, however, **scaling cluster nodes will not lead to expected results**.
We require to update Machine Deployment objects and rotate machines right after the upgrade.
{{% /notice %}}

### Required Migration Steps

Here we are going to cover manual steps that are required for smooth transition from technology preview to GA.

#### Upgrade of KubeVirt Infrastructure Cluster

{{% notice warning %}}
Updating of Kubernetes, KubeVirt and Containerized Data Imported should be done from N-1 to N release.
{{% /notice %}}

The k8s cluster and KubeVirt components must be in [scope of our supported versions](#requirements).
You can update k8s version by following [the official guide](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/).
Or if you provisioned the cluster over KubeOne please follow [the update procedure](/kubeone/v1.5/tutorials/upgrading-clusters/).

Next you can update KubeVirt control plane and Containerized Data Importer by executing:

```shell
export RELEASE=<SUPPORTED_RELEASE>
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
```

```shell
export RELEASE=<SUPPORTED_RELEASE>
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${RELEASE}/cdi-operator.yaml
```

{{% notice note %}}
Refer to [the KubeVirt control update documentation](https://kubevirt.io/user-guide/operations/updating_and_deletion/#updating-kubevirt-control-plane)
for further details about the update.
{{% /notice %}}

{{% notice warning %}}
It is highly recommended to first test the upgrade on a staging environment.
It can happen that hosts where KubeVirt runs might need reconfiguration or packages update.
{{% /notice %}}

#### Update Machine Deployment

{{% notice warning %}}
If user clusters have working Load Balancers those can be unreachable after Machine Deployment rotation due to LB selector key change.
After this step, it is required to follow the [Update LoadBalancer selectors](#update-loadbalancer-selectors) guide to fix it.
{{% /notice %}}

Right after the upgrade it is required to update Machine Deployment object (this will trigger Machines rotation).
You can do it from the KKP Dashboard which is recommended approach as you will be guided with the possible options.

![Machine Deployment Edit](images/mc-edit.png?classes=shadow,border "Machine Deployment Edit")

The alternative is to directly change Machine Deployment objects over `kubectl apply`.
Take a look into [the example](https://github.com/kubermatic/machine-controller/blob/v1.56.0/examples/kubevirt-machinedeployment.yaml) to see what has been changed.

#### Update LoadBalancer Selectors

{{% notice note %}}
This step is only required if Load Balancers on user clusters have been created, and the previous [Update Machine Deployment](#update-machine-deployment) guide has been accomplished.
{{% /notice %}}

From v0.3.0 KubeVirt Cloud Controller Manager changed Load Balancer selector. You have to edit Load Balancer(s) in
user cluster namespace on KubeVirt infrastructure and change its selector from
`cloud.kubevirt.io/<id>: <val>`
to
`cluster.x-k8s.io/cluster-name: <cluster-id>`.

For instance the selector of LB that exists in KubeVirt infrastructure cluster in the `cluster-xyz` namespace,
would have to be replaced to
`cluster.x-k8s.io/cluster-name: xyz`
