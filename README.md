# SNO with OpenShift Agent Based Installer

## Overview

Sister repo for [Multiple Nodes OpenShift](https://github.com/borball/mno-with-abi): 

Note: This repo only works for OpenShift 4.12+. For 4.9/4.10 please use this [helper script](https://github.com/borball/sno-manual-helper)

This repo provides set of helper scripts to install SNO(Single Node OpenShift) with [OpenShift Agent Based Installer](https://docs.openshift.com/container-platform/4.12/installing/installing_with_agent_based_installer/preparing-to-install-with-agent-based-installer.html) and install operators and apply tunings which are recommended and required by vDU applications.

- sno-iso: Used to generate bootable ISO image based on agent based installer, some operators and node tunings for [vDU applications](https://github.com/openshift-kni/cnf-features-deploy/tree/release-4.12/ztp/gitops-subscriptions/argocd/example/policygentemplates) are enabled as day 1 operations.
- sno-install: Used to mount the ISO image generated by sno-iso to BMC console as virtual media with Redfish API, and boot the node from the image to trigger the SNO installation. Tested on HPE, ZT and KVM with Sushy tools, other servers may/not work, please create issues if not.
- sno-day2: Most of the operators and tunings required by vDU applications are enabled as day1, but some of them can only be done as [day2 configurations](templates/openshift/day2).
- sno-ready: Used to validate if the SNO cluster has all vDU required configuration and tunings.

## Dependencies
Some software and tools are required to be installed before running the scripts:

- nmstatectl: sudo dnf install /usr/bin/nmstatectl -y
- yq: https://github.com/mikefarah/yq#install
- jinja2: pip3 install jinja2-cli, pip3 install jinja2-cli[yaml] 
 
## Configuration

Prepare config.yaml to fit your lab situation, here is an example:

```yaml
cluster:
  domain: outbound.vz.bos2.lab
  name: sno148
  #optional: set ntps servers 
  #ntps:
    #- 0.rhel.pool.ntp.org
    #- 1.rhel.pool.ntp.org
host:
  hostname: sno148.outbound.vz.bos2.lab
  interface: ens1f0
  mac: b4:96:91:b4:9d:f0
  ipv4:
    enabled: true
    dhcp: false
    ip: 192.168.58.48
    dns: 192.168.58.15
    gateway: 192.168.58.1
    prefix: 25
    machine_network_cidr: 192.168.58.0/25
    #optional, default 10.128.0.0/14
    #cluster_network_cidr: 10.128.0.0/14
    #optional, default 23
    #cluster_network_host_prefix: 23
  ipv6:
    enabled: false
    dhcp: false
    ip: 2600:52:7:58::48
    dns: 2600:52:7:58::15
    gateway: 2600:52:7:58::1
    prefix: 64
    machine_network_cidr: 2600:52:7:58::/64
    #optional, default fd01::/48
    #cluster_network_cidr: fd01::/48
    #optional, default 64
    #cluster_network_host_prefix: 64
  vlan:
    enabled: false
    name: ens1f0.58
    id: 58
  disk: /dev/nvme0n1

cpu:
  isolated: 2-31,34-63
  reserved: 0-1,32-33

proxy:
  enabled: false
  http:
  https:
  noproxy:

pull_secret: ./pull-secret.json
ssh_key: /root/.ssh/id_rsa.pub

bmc:
  address: 192.168.13.148
  username: Administrator
  password: dummy

iso:
  address: http://192.168.58.15/iso/agent-148.iso

```

By default following will be enabled during day1(installation phase):

  - Workload partitioning
  - SNO boot accelerate
  - Kdump service/config
  - Local Storage Operator
  - PTP Operator
  - SR-IOV Network Operator

You can turn on/off the day1 operations in the config file under section [day1](samples/usage.md#day1).

In some case you may want to include more customizations for the cluster during day1, you can create folder extra-manifests and put those CRs(Custom Resources) inside before you run sno-iso.sh, the script will copy and include those inside the ISO image. see [advanced usage](samples/usage.md#advanced) of the configurations.

Get other sample [configurations](samples/usage.md#usages).

## Generate ISO image

You can run sno-iso.sh [config file] to generate a bootable ISO image so that you can boot from BMC console to install SNO. By default stable-4.12 will be downloaded and installed if not specified.

```
# ./sno-iso.sh -h
Usage: ./sno-iso.sh [config file] [ocp version]
config file and ocp version are optional, examples:
- ./sno-iso.sh sno130.yaml              equals: ./sno-iso.sh sno130.yaml stable-4.12
- ./sno-iso.sh sno130.yaml 4.12.10

Prepare a configuration file by following the example in config.yaml.sample           
-----------------------------------
# content of config.yaml.sample
...
-----------------------------------
Example to run it: ./sno-iso.sh config-sno130.yaml   

```

[Demo](#sno-iso)

## Boot node from ISO image

Once you generate the ISO image, you can boot the node from the image with your prefered way, OCP will be installed automatically.

A helper script sno-install.sh is available in this repo to boot the node from ISO and trigger the installation automatically, assume you have an HTTP server (http://192.168.58.15/iso in our case) to host the ISO image.

Define your BMC info and ISO location in the configuration file first:

```yaml
bmc:
  address: 192.168.14.130
  username: Administrator
  password: Redhat123!
  node_uuid:

iso:
  address: http://192.168.58.15/iso/agent-130.iso

```

Then run it:

```
# ./sno-install.sh 
Usage : ./sno-install.sh config-file
Example : ./sno-install.sh config-sno130.yaml
```

[Demo](#sno-install)

## Day2 operations

Some CRs are not supported in installation phase as day1 operations including PerformanceProfile(4.13+ will support), those can/shall be done as day 2 operations once SNO is deployed.

```
# ./sno-day2.sh -h
Usage: ./sno-day2.sh [config.yaml]
config.yaml is optional, will use config.yaml in the current folder if not being specified.
```

You can turn on/off day2 operation with configuration in section [day2](samples/usage.md#day2).

[Demo](#sno-day2)

## Validation

After applying all day2 operarions, node may be rebooted once, check if all vDU required tunings and operators are in placed:

```
# ./sno-ready.sh 
NAME                          STATUS   ROLES                         AGE     VERSION
sno148.outbound.vz.bos2.lab   Ready    control-plane,master,worker   3h34m   v1.25.7+eab9cc9

Checking node:
 [+]Node is ready.

Checking all pods:
 [+]No failing pods.

Checking required machine config:
 [+]MachineConfig container-mount-namespace-and-kubelet-conf-master exits.
 [+]MachineConfig 02-master-workload-partitioning exits.
 [+]MachineConfig 04-accelerated-container-startup-master exits.
 [+]MachineConfig 05-kdump-config-master exits.
 [+]MachineConfig 06-kdump-enable-master exits.
 [+]MachineConfig 99-crio-disable-wipe-master exits.
 [-]MachineConfig disable-chronyd is not existing.

Checking machine config pool:
 [+]mcp master is updated and not degraded.

Checking required performance profile:
 [+]PerformanceProfile sno-performance-profile exits.
 [+]topologyPolicy is single-numa-node
 [+]realTimeKernel is enabled

Checking required tuned:
 [+]Tuned performance-patch exits.

Checking SRIOV operator status:
 [+]sriovnetworknodestate sync status is 'Succeeded'.

Checking PTP operator status:
 [+]Ptp linuxptp-daemon is ready.
No resources found in openshift-ptp namespace.
 [-]PtpConfig not exist.

Checking openshift monitoring.
 [+]Grafana is not enabled.
 [+]AlertManager is not enabled.
 [+]PrometheusK8s retention is not 24h.

Checking openshift console.
 [+]Openshift console is disabled.

Checking network diagnostics.
 [+]Network diagnostics is disabled.

Checking Operator hub.
 [+]Catalog community-operators is disabled.
 [+]Catalog redhat-marketplace is disabled.

Checking /proc/cmdline:
 [+]systemd.cpu_affinity presents: systemd.cpu_affinity=0,1,32,33
 [+]isolcpus presents: isolcpus=managed_irq,2-31,34-63
 [+]Isolated cpu in cmdline: 2-31,34-63 matches with the ones in performance profile: 2-31,34-63
 [+]Reserved cpu in cmdline: 0,1,32,33 matches with the ones in performance profile: 0-1,32-33

Checking RHCOS kernel:
 [+]Node is realtime kernel.

Checking kdump.service:
 [+]kdump is active.
 [+]kdump is enabled.

Checking chronyd.service:
 [-]chronyd is active.
 [-]chronyd is enabled.

Checking crio-wipe.service:
 [+]crio-wipe is inactive.
 [+]crio-wipe is not enabled.

Completed the checking.

```

[Demo](#sno-ready)

## Demos

### sno-iso

  [![sno-iso](https://asciinema.org/a/580242.svg)](https://asciinema.org/a/580242?autoplay=1)

### sno-install

  [![sno-install](https://asciinema.org/a/580837.svg)](https://asciinema.org/a/580837?autoplay=1)

### sno-day2

  [![sno-day2](https://asciinema.org/a/580241.svg)](https://asciinema.org/a/580241?autoplay=1)

## sno-ready

  [![sno-ready](https://asciinema.org/a/580250.svg)](https://asciinema.org/a/580250?autoplay=1)
