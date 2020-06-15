# Introduction and Prerequisites

This document provides information to the user about how to deploy and consume CNF related operators and additional objects. The goal is to understand how to deploy those elements and how to properly use them.

The topics covered in this document are:

- performance addon operator
- SR-IOV operator
- PTP operator
- SCTP module
- DPDK s2i

**NOTE:** In each case, we deliberately choose to use plain yaml manifests to deploy the components to provide the user a clear view of what assets are involved.

## General Prerequisites

The following items are needed in order to be able to complete the lab from beginning to end:

* An openshift cluster with at least one physical worker, ideally two.
* In a disconnected environment, imagecontentsources and catalog sources in place.
* oc installed and configured with KUBECONFIG env variable.
* imageregistry properly configured with a valid backend for dpdk s2i push to work.
* git tool and repository `https://github.com/karmab/performance-operators-lab` cloned

**NOTE:** Both masters and workers can be virtual. If using virtual workers, you will only be able to properly use the performance profile operator and sctp module, as the other ones have hardware dependency.

**NOTE:** If running disconnected, you will have to adapt the source field in the install samples to match your environment.

Clone the repo where we host the sample assets:

```
git clone https://github.com/karmab/performance-operators-lab
cd performance-operators-lab
``` 

# Performance addon operator

In this section, we install and configure the performance addon operator, which is an operator aimed at simplifying the configuration of workers needing enhancements in order to run compute sensitive workloads.

## Deployment

Launch the following command:

```
oc create -f performance/install.yml
```

Expected Output

```
namespace/openshift-performance-addon created
operatorgroup.operators.coreos.com/openshift-performance-addon-operatorgroup created
subscription.operators.coreos.com/performance-addon-operator-subscription created
```

You can then wait for operator to show in the openshift-performance-addon namespace with `oc get pod -n openshift-performance-addon`

Expected Output

```
NAME                                    READY   STATUS    RESTARTS   AGE
performance-operator-6694c8c9fb-q64qw   1/1     Running   1          123m
```

## How to use

The performance addon operator deploys a custom resource (CR) named *performanceprofile* that we can use to configure the nodes. Creating such profile will trigger the creation of underlying objects such as machineconfigs or tuned objects, with handling done by more lowlevel operators.

The full list of attributes handled by the operator can be see [here](https://github.com/openshift-kni/performance-addon-operators/blob/master/deploy/crds/performance.openshift.io_performanceprofiles_crd.yaml).

The workflow goes as follows:

- Label the workers that need specific configuration. We will use the commonly used label *worker-cnf*: 

```
oc label node your_worker node-role.kubernetes.io/worker-cnf='' --overwrite
```

- Create machine config pools that matches previously set labels. As previous task, it's admin's responsability to create those assets to match its platforms

for instance we can use the following definition and create it with `oc apply -f`

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker-cnf
  labels:
    machineconfiguration.openshift.io/role: worker-cnf
spec:
  machineConfigSelector:
    matchExpressions:
      - {
          key: machineconfiguration.openshift.io/role,
          operator: In,
          values: [worker-cnf, worker],
        }
  paused: false
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/worker-cnf: ""
```

In This definition, note that:

  * We can only indicate one extra label beyond worker in the machineConfigSelector. Note that it implies that a given node can only belong to a single machine config pool at once.
  * The machineconfig pool has the *paused* field to False. If *paused* were set to True, performance profile operator will create the assets but nothing will effectively be applied.
  * We specify which labels to match in the nodeSelector section.


- Create a performance profile CR that matches the label:

```
apiVersion: performance.openshift.io/v1alpha1
kind: PerformanceProfile
metadata:
  name: worker-cnf
spec:
  cpu:
    isolated: 0-8
    reserved: 9-15
  hugepages:
    defaultHugepagesSize: "1G"
    pages:
    - size: "1G"
      count: 16
      node: 0
  realTimeKernel:
    enabled: true
  nodeSelector:
    node-role.kubernetes.io/worker-cnf: ""
```

In this definition, note the following elements:

- We are setting 16 hugepages of 1GB on numa node 0.  For a testing/virtual env, you'll want to lower this number to 1

- We are enabling realtime kernel, which will effectively add an extra label to one of the created machineconfigs so that the machineconfig operator reboots the node using already installed kernel with realtime.

- We're targeting the machineconfigpool indirectly by matching the proper label in the *nodeSelector* section.

After creating this CR, you can monitor the machineconfigpools master and worker-cnf with `oc get mcp` to see the progress towards enabling the features. 

**NOTE:** All the nodes will initially be rebooted the first time, as a feature gate for the topology manager gets enabled.

# SR-IOV Operator

## Deployment

**NOTE:** In order to connect our pod to a real dhcp network, we need to patch the openshift network operator to Add a dummy dhcp network to start the dhcp daemonset by the operator.

```
oc patch networks.operator.openshift.io cluster --type='merge' -p='{"spec":{"additionalNetworks":[{"name":"dummy-dhcp-network","simpleMacvlanConfig":{"ipamConfig":{"type":"dhcp"},"master":"eth0","mode":"bridge","mtu":1500},"type":"SimpleMacvlan"}]}}'
```

Launch the following command:

```
oc create -f sriov/install.yml
```

Expected Output

```
namespace/openshift-sriov-network-operator created
operatorgroup.operators.coreos.com/sriov-network-operators created
subscription.operators.coreos.com/sriov-network-operator created
```

You can then wait for operators to show in the openshift-sriov-network-operator namespace with `oc get pod -n openshift-sriov-network-operator`

Expected Output

```
NAME                                      READY   STATUS        RESTARTS   AGE
network-resources-injector-hntx4          1/1     Running       0          176m
network-resources-injector-rdgqt          1/1     Running       0          176m
network-resources-injector-zth79          1/1     Running       0          176m
operator-webhook-8npdk                    1/1     Running       0          176m
operator-webhook-hnnz2                    1/1     Running       0          176m
operator-webhook-vqhjg                    1/1     Running       0          176m
sriov-cni-zjff2                           1/1     Running       0          3m50s
sriov-device-plugin-4wf9x                 1/1     Running       0          109s
sriov-network-config-daemon-bwdw9         1/1     Running       0          88m
sriov-network-config-daemon-jhhwp         1/1     Running       1          88m
sriov-network-operator-5f8cb9fb58-ql648   1/1     Running       0          113m
```

Beyond operator, sriov-network-config-daemon pods appear for each node.

## How to use

After the operator gets installed, We have the following CRS:

- SriovNetworkNodeState
- SriovNetwork 
- SriovNetworkNodePolicy

SriovNetworkNodeState CRS are readonly and provide information about SR-IOV capable devices in the cluster. We can list them with `oc get sriovnetworknodestates.sriovnetwork.openshift.io -n openshift-sriov-network-operator  -o yaml`

Expected Output

```
apiVersion: v1
items:
- apiVersion: sriovnetwork.openshift.io/v1
  kind: SriovNetworkNodeState
  metadata:
    creationTimestamp: "2020-05-25T22:08:04Z"
    generation: 19
    name: cnf10-worker-0.xxx.kni.lab.bonka.mad.hendrix.com
    namespace: openshift-sriov-network-operator
    ownerReferences:
    - apiVersion: sriovnetwork.openshift.io/v1
      blockOwnerDeletion: true
      controller: true
      kind: SriovNetworkNodePolicy
      name: default
      uid: 642fc098-d30c-4638-8851-edaf68b00357
    resourceVersion: "426718"
    selfLink: /apis/sriovnetwork.openshift.io/v1/namespaces/openshift-sriov-network-operator/sriovnetworknodestates/cnf10-worker-0.xxx.lab.mad.hendrix.com
    uid: b92037d2-c1bb-43c6-84a0-59973e7815bd
  spec:
    dpConfigVersion: "425914"
    interfaces:
    - name: eno1
      numVfs: 5
      pciAddress: "0000:19:00.0"
      vfGroups:
      - deviceType: netdevice
        resourceName: testresource
        vfRange: 2-4
  status:
    interfaces:
    - Vfs:
      - deviceID: "1016"
        driver: mlx5_core
        mtu: 1500
        pciAddress: "0000:19:00.2"
        vendor: 15b3
        vfID: 0
      - deviceID: "1016"
        driver: mlx5_core
        mtu: 1500
        pciAddress: "0000:19:00.3"
        vendor: 15b3
        vfID: 1
      - deviceID: "1016"
        driver: mlx5_core
        mtu: 1500
        pciAddress: "0000:19:00.4"
        vendor: 15b3
        vfID: 2
      - deviceID: "1016"
        driver: mlx5_core
        mtu: 1500
        pciAddress: "0000:19:00.5"
        vendor: 15b3
        vfID: 3
      - deviceID: "1016"
        driver: mlx5_core
        mtu: 1500
        pciAddress: "0000:19:00.6"
        vendor: 15b3
        vfID: 4
      deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: eno1
      numVfs: 5
      pciAddress: "0000:19:00.0"
      totalvfs: 5
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: eno2
      pciAddress: "0000:19:00.1"
      totalvfs: 5
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: ens1f0
      pciAddress: 0000:3b:00.0
      totalvfs: 5
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: ens1f1
      pciAddress: 0000:3b:00.1
      totalvfs: 5
      vendor: 15b3
    syncStatus: Succeeded
- apiVersion: sriovnetwork.openshift.io/v1
  kind: SriovNetworkNodeState
  metadata:
    creationTimestamp: "2020-05-26T09:21:48Z"
    generation: 2
    name: cnf11-worker-0.xxx.lab.mad.hendrix.com
    namespace: openshift-sriov-network-operator
    ownerReferences:
    - apiVersion: sriovnetwork.openshift.io/v1
      blockOwnerDeletion: true
      controller: true
      kind: SriovNetworkNodePolicy
      name: default
      uid: 642fc098-d30c-4638-8851-edaf68b00357
    resourceVersion: "425937"
    selfLink: /apis/sriovnetwork.openshift.io/v1/namespaces/openshift-sriov-network-operator/sriovnetworknodestates/cnf11-worker-0.xxx.lab.mad.hendrix.com
    uid: fcda2f57-b0bf-444f-ae8d-c9329f574544
  spec:
    dpConfigVersion: "425914"
  status:
    interfaces:
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: eno1
      pciAddress: "0000:19:00.0"
      totalvfs: 5
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: eno2
      pciAddress: "0000:19:00.1"
      totalvfs: 5
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: ens1f0
      pciAddress: 0000:3b:00.0
      totalvfs: 5
      vendor: 15b3
    - deviceID: "1015"
      driver: mlx5_core
      mtu: 1500
      name: ens1f1
      pciAddress: 0000:3b:00.1
      totalvfs: 5
      vendor: 15b3
    syncStatus: Succeeded
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

We can get a given nic configured by the operator by creating a SriovNetworkNodePolicy CR, by specifying it with `nicSelector` and targetting specific nodes with `nodeSelector`, for instance to configure eno1:

```
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: sriov-network-node-policy
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  isRdma: true
  nicSelector:
    pfNames:
      - eno1
  nodeSelector:
    node-role.kubernetes.io/worker-cnf: ""
  numVfs: 5
  resourceName: sriovnic
```

Once the node policy is created, the operator will update the node (its nic) accordingly, which can be viewed using the previous `SriovNetworkNodeState`. Note it might imply that the node gets rebooted as some elements are BIOS specific.

**NOTE:** You might have to adapt the spec depending on your nic model.Consult [https://docs.openshift.com/container-platform/4.4/networking/hardware_networks/about-sriov.html#supported-devices_about-sriov](https://docs.openshift.com/container-platform/4.4/networking/hardware_networks/about-sriov.html#supported-devices_about-sriov) for details

Finally, we create a SriovNetwork CR which refer to the 'resourceName' defined in SriovNetworkNodePolicy. Then a network-attachment-definitions CR will be generated by operator with the same name and namespace, for instance: 

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: sriov-testing
---
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: sriov-network
  namespace: openshift-sriov-network-operator
spec:
  ipam: |
    {
      "type": "dhcp"
    }
  networkNamespace: sriov-testing
  resourceName: sriovnic
  vlan: 0
```

A new network-attachment-definition got created, which we can then use in our pod definition, as we would for other multus backends.

```
oc get network-attachment-definitions -n sriov-testing
```

Expected Output

```
NAME           AGE
sriov-network   3d14h
```

Pods can be created making use of this network attachment definition:

```
apiVersion: v1
kind: Pod
metadata:
  name: sriovpod
  namespace: sriov-testing
  annotations:
    k8s.v1.cni.cncf.io/networks:  sriov-network
spec:
  containers:
  - name: sriovpod
    command: ["/bin/sh", "-c", "trap : TERM INT; sleep 600000& wait"]
    image: alpine
```

# PTP Operator

## Deployment

Launch the following command:

```
oc create -f ptp/install.yml
```

Expected Output

```
namespace/openshift-ptp created
operatorgroup.operators.coreos.com/ptp-operators created
subscription.operators.coreos.com/ptp-operator-subscription created
```

We wait for operators to show in the openshift-ptp namespace with `oc get pod -n openshift-ptp`

Expected Output

```
NAME                           READY   STATUS    RESTARTS   AGE
linuxptp-daemon-9tvk8          1/1     Running   0          18m
linuxptp-daemon-qv9w9          1/1     Running   0          18m
linuxptp-daemon-r6dr4          1/1     Running   0          18m
linuxptp-daemon-sbfgs          1/1     Running   0          18m
linuxptp-daemon-w4tbx          1/1     Running   0          18m
ptp-operator-8844cc676-7d6hc   1/1     Running   0          112m
```

Beyond operator, we can see linuxptp-daemons pods for each node, which encapsulates the ptp4l daemon.

## How to use

The operator deploys the CR PtpConfig that we can use to configure the nodes by matching them with a specific label. For instance, to configure a node as PTP grandmaster, we can inject the following CR

```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: grandmaster
  namespace: openshift-ptp
spec:
  profile:
  - name: "grandmaster"
    interface: "eno1"
    ptp4lOpts: ""
    phc2sysOpts: "-a -r -r"
  recommend:
  - profile: "grandmaster"
    priority: 4
    match:
    - nodeLabel: "ptp/grandmaster"
```

Or, for a slave:

```
apiVersion: ptp.openshift.io/v1
kind: PtpConfig
metadata:
  name: slave
  namespace: openshift-ptp
spec:
  profile:
  - name: "slave"
    interface: "eno1"
    ptp4lOpts: "-s"
    phc2sysOpts: "-a -r"
  recommend:
  - profile: "slave"
    priority: 4
    match:
    - nodeLabel: "ptp/slave"
``` 

The difference between those two CRS lies in :

- the ptp4lOpts and phc2sysOpts attributes of the profile.
- the matching done between a profile and nodeLabel.

We can then Label the workers that need specific configuration. For instance, for the nodes to be used as grandmaster: 

```
oc label node your_worker ptp/grandmaster='' --overwrite
```

We can then monitor the linuxptp-daemon pods of each node to check how the profile gets applied (and sync occurs, if a grandmaster is found).

# SCTP module

Launch the following command:

```
oc create -f sctp/install.yml
```

Expected Output

```
machineconfig.machineconfiguration.openshift.io/load-sctp-module created
```

The SCTP module consists of a single machineconfig, which makes sure that the sctp module is not blacklisted and loaded at boot time. We can inject the following manifest with `oc apply -f`

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker-cnf
  name: load-sctp-module
spec:
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
        - contents:
            source: data:,
            verification: {}
          filesystem: root
          mode: 420
          path: /etc/modprobe.d/sctp-blacklist.conf
        - contents:
            source: data:text/plain;charset=utf-8,sctp
          filesystem: root
          mode: 420
          path: /etc/modules-load.d/sctp-load.conf
```

Once done, and provided there is a matching mcp in the cluster, the node will get rebooted and have the module loaded, which can be checked by sshing in the node (or running `oc debug node/$node`) and running `sudo lsmod | grep sctp`.

# DPDK s2i

This part covers an image containing DPDK framework and built using source to image. It depends on sriov beeing deployed and working on the cluster.

You would use this mechanism to build and package a dpdk based application from a git repository but using a DPDK well known base image.


## Deployment




We launch the following yamls, which will trigger the building of the image and its pushing against image registry

```
oc create -f dpdk/dpdk-network.yml
oc create -f dpdk/scc.yml
```


We will create a secret so that we can pull the dpdk base image from redhat.io


```
SECRET='registrysecret'
REGISTRY='registry.redhat.io'
USERNAME='XXX'
PASSWORD='YYY'
MAIL="ZZZ"
oc create secret docker-registry $SECRET --docker-server=$REGISTRY --docker-username=$USERNAME --docker-password=$PASSWORD --docker-email=$MAIL
oc secrets link default $SECRET --for=pull
oc secrets link builder $SECRET --for=pull
```

Then we launch the building of the image from source code and its pushing against image registry:

```
oc create -f dpdk/build-config.yml
```

**NOTE:** the build config points to https://github.com/openshift-kni/cnf-features-deploy/tree/master/tools/s2i-dpdk/test/test-app as a sample app. In a real world, you would point to the source code where your application lives.


**NOTE:** the build config makes use of the dpdk-base-rhel8 image fetching it from registry.redhat.io. In a disconnected environment, you would edit the yaml so that it points to your disconnected registry.

Once we create thosse assets, we can check the building of the image in the dpdk namespace with `oc get pod -n dpdk`, which will eventuall show as a completed pod.

```
NAME               READY   STATUS    RESTARTS   AGE
s2i-dpdk-1-build   1/1     Running   0          80s
```

## How to use

We can then create a nodepolicy to configure a given nic, the corresponding sriovnetwork and a deployment config to actually launch the resulting application.

```
oc create -f dpdk/sriov-networknodepolicy-dpdk.yml
oc create -f dpdk/deployment-config.yml
```

The app will show as pod named s2i-dpdk-app-* in the dpdk namespace.
One can then oc rsh in the pod and run testpmd commands.

# Additional resources

## Documentation

- [https://docs.openshift.com/container-platform-ocp/4.4/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html](https://docs.openshift.com/container-platform-ocp/4.4/scalability_and_performance/cnf-performance-addon-operator-for-low-latency-nodes.html)
- [https://docs.openshift.com/container-platform/4.4/scalability_and_performance/using-topology-manager.html](https://docs.openshift.com/container-platform/4.4/scalability_and_performance/using-topology-manager.html)
- [https://docs.openshift.com/container-platform/4.4/scalability_and_performance/using-cpu-manager.html](https://docs.openshift.com/container-platform/4.4/scalability_and_performance/using-cpu-manager.html)
- [https://docs.openshift.com/container-platform/4.4/networking/hardware_networks/installing-sriov-operator.html#installing-sriov-operator](https://docs.openshift.com/container-platform/4.4/networking/hardware_networks/installing-sriov-operator.html#installing-sriov-operator)
- [https://docs.openshift.com/container-platform/4.4/networking/using-sctp.html](https://docs.openshift.com/container-platform/4.4/networking/using-sctp.html)
