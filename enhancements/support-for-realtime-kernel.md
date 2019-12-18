---
title: support-for-real-time-kernels
authors:
  - "@miabbott"
reviewers:
  - "@ashcrow"
  - "@cgwalters"
  - "@crawford"
  - "@darkmuggle"
  - "@davidvossel"
  - "@imcleod"
  - "@jlebon"
  - "@MarSik"
  - "@mrguitar"
  - "@runcom"  
  - "@simon3z"
  - "@sinnykumari"
approvers:
  - "@ashcrow"
  - "@cgwalters"
  - "@crawford"
  - "@imcleod"
  - "@runcom"
creation-date: 2019-12-18
last-updated: 2019-12-18
status: provisional
see-also:
replaces:
superseded-by:
---

# Support For Real-time Kernels

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)
 
## Open Questions

1.  **Will the `rpm-ostree override` command use filenames or rely on a temp repo for package names?**
2. **How will the upgrade path be handled?**
Nodes that have had the `kernel-rt` packages deployed may have them deployed via filename which would complicate keeping the overridden packages up-to-date. (Follow-on to #1)
3.  **Can the real-time kernel be used with FIPS?**

## Summary

Many workloads associated with telcos and other verticals like FSIs require a high degree of determinism. While Linux is not a real-time operating system, the real-time kernel provides a preemptive scheduler providing the OS with real-time characteristics. We want to provide customers the ability to select the real-time kernel for nodes in the cluster.

## Motivation

The real-time kernel, along with realtime-friendly hardware and BIOS settings, is a requirement to meet the low latency needs that Verizon, Ericsson, Nokia, along with other NEPs, are looking to deploy as part of the 5G roll-out.

Unfortunately, the additional overhead from the scheduler makes it a poor fit for the majority of “general purpose” workloads; therefore, real-time is not suited to be the default kernel in RHCOS. OCP clusters and MachineConfigPools will need to be configurable to run either kernel.

### Goals

- Provide the ability to select the real-time kernel for a set of nodes in the cluster
- Provide the `kernel-rt` packages in the `machine-os-content` image
- Provide initial tuning of the RHCOS nodes for real-time workloads

### Non-Goals

- Providing the `kernel-rt` packages as part of the boot images
- Additional tuning of the nodes after the real-time kernel is selected
- Provide real-time kernel support to older versions (pre-4.4) of OCP
- Support real-time kernel selection on non-RHCOS nodes

## Proposal

1.  Include the `kernel-rt` packages in the `machine-os-content` image
2.  Provide tunable in MachineConfig that selects the real-time kernel

### User Stories 

#### Story 1 - Fresh Install

Cyberdyne Systems wants to deploy OCP at their radio tower to handle the processing
of the radio signals.  Processing the signals requires the determinism and guarantees
that come with a real-time kernel.  They deploy OCP and then create a MachineConfig to
select the real-time kernel on their RHCOS nodes.

#### Story 2 - Upgrade Path

Nakatomi Corp. has an older OCP cluster deployed using all RHCOS nodes.  They plan on
introducing a workload that would benefit from the use of the real-time kernel.
They upgrade their OCP cluster to the latest version with real-time kernel support
and create a MachineConfig that selects the real-time kernel on their nodes.

#### Story 3 - Node Replacement

Tyrell Corp. has an existing OCP cluster deployed with RHEL 7 worker nodes using
the real-time kernel.  They want to upgrade to the latest version of OCP and switch
their worker nodes to RHCOS nodes.  They remove the RHEL 7 worker nodes from the
cluster, upgrade the cluster, add in RHCOS nodes, and create a MachineConfig to
select the real-time kernel on their RHCOS nodes.

#### Story 4 - Unsupported Version

Corellian Engineering Corp. has an existing OCP cluster that does not support
real-time kernels.  They learn of support for real-time kernels in the new version
of OCP and try to create a MachineConfig that selects the real-time kernel.  The
MachineConfig is ignored.

### Implementation Details/Notes/Constraints

This proposal only covers providing the real-time kernel to RHCOS nodes and
selecting it via MachineConfig.  It does not cover any changes that may be
required by the container runtime or other components of OCP.

### Risks and Mitigations

#### Performance Impact

Selecting the real-time kernel for OCP nodes without properly configuring the
underlying hardware **MAY** incur an undesirable performance impact.  It is
recommended that only users who control their hardware (i.e. not cloud users)
attempt to use the real-time kernel.  If users enable real-time kernel support
without properly configured hardware, they can always remove the offending 
MachineConfig to return to the previous state.

#### Lack of Full Support

If there are additional changes required across the OCP product to properly support
real-time kernel (i.e. container runtime, etc) which cannot be delivered at the
same time, the real-time kernel packages can still be included as part of the 
`machine-os-content` image.  Additionally, exposing the tunable in the MachineConfig
spec can be removed.  Once support has been fully committed across the product,
we can expose the tunable again.

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

## Design Details

### Providing the Packages

The proposal is to include the `kernel-rt` packages in the `machine-os-content`
image.  They **WILL NOT** be included as part of the ostree commit, but rather
as separate files within the image.  Example:

```
$ sudo podman pull quay.io/openshift-release-dev/ocp-v4.0-art-dev:realtime
Trying to pull quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:realtime...
Getting image source signatures
Copying blob 3c141bccb4e4 skipped: already exists
Copying config 485b845970 done
Writing manifest to image destination
Storing signatures
485b845970f19098b662f85a9e22bf4db7aabea83fe5d9c155d798dffcd6f2e9

$ ctr=$(sudo podman create quay.io/openshift-release-dev/ocp-v4.0-art-dev:realtime)

$ mnt=$(sudo podman mount $ctr)

$ sudo ls -l $mnt
total 50036
-rw-r--r--. 1 miabbott miabbott 26061020 Dec 16 15:58 kernel-rt-core-4.18.0-147.0.3.rt24.95.el8_1.x86_64.rpm
-rw-r--r--. 1 miabbott miabbott 22869480 Dec 16 15:58 kernel-rt-modules-4.18.0-147.0.3.rt24.95.el8_1.x86_64.rpm
-rw-r--r--. 1 miabbott miabbott  2285240 Dec 16 15:58 kernel-rt-modules-extra-4.18.0-147.0.3.rt24.95.el8_1.x86_64.rpm
-rw-rw-r--. 1 root     root        13764 Dec 16 15:59 pkglist.txt
drwxrwxr-x. 3 root     root           18 Dec 16 15:58 srv

```

### Selecting the Kernel

The proposal is to have the MachineConfig spec changed to support a boolean named
`realtime` which will determine the which kernel should be used on the RHCOS nodes.
If `realtime: true`, the `kernel-rt` packages will be used.  If `realtime: false`,
the vanilla kernel will be used.  In the absence of any `realtime` field, the default
choice is for the vanilla kernel to be used.

Example MachineConfig:

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: "worker"
  name: 99-worker-realtime-kernel
spec:
  realtime: true
```

### Installing the Real-time Kernel

When the MCO parses a MachineConfig with `realtime: true`, it shall instruct 
`rpm-ostree` on the RHCOS node to override the installed kernel with the `kernel-rt`
package from the `machine-os-content` image.

### Removing the Real-time Kernel

When the MCO parses a MachineConfig with `realtime: false`, it shall instruct `rpm-ostree`
on the RHCOS node to remove any `kernel-rt` packages and use the vanilla kernel.  If the
`kernel-rt` packages are not present, it should be a no-op. 

### Upgrades

After the real-time kernel has been selected and deployed to nodes, it will remain in
place throughout upgrades of the cluster.  **(See open question at beginning of doc)**

### Test Plan

- Test install of `kernel-rt` packages on single RHCOS node
- Test removal of `kernel-rt` packages on single RHCOS node
- Test `realtime: true` on OCP cluster with RHCOS nodes
- Test `realtime: false` on OCP cluster with RHCOS nodes already using `kernel-rt`
- Test `realtime: false` on OCP cluster with RHCOS nodes on vanilla kernel
- Test `realtime: true` on OCP cluster with RHEL nodes
- Test `realtime: false` on OCP cluster with RHEL nodes
- Test `realtime: true` on older OCP cluster

### Graduation Criteria

For this feature to be considered stable:
- `kernel-rt` packages available in `machine-os-content` images
- MachineConfig spec has `realtime` boolean
- Cluster with nodes using `kernel-rt` is upgraded successfully


### Upgrade / Downgrade Strategy

- Nodes using `kernel-rt` packages should continue to use `kernel-rt` packages through upgrades
- Nodes using `kernel-rt` packages should continue to use `kernel-rt` packages after a downgrade.
    - **WARNING**: downgrading a cluster to a previous version that does not have real-time kernel support is unsupported.

### Version Skew Strategy

There should be no implications for realtime kernel support if other components in the cluster are at a different version.

## Implementation History

- 2019-12-11: RT kernel included in `machine-os-content` image - https://url.corp.redhat.com/rhcos-rt-kernel
- 2019-12-12: PR for MCO support - https://github.com/openshift/machine-config-operator/pull/1330

## Drawbacks

If there are additional changes required to cluster components for full support of the realtime kernel, this proposal has not been scoped for that and should be pushed to a later release.

## Alternatives

The alternative model for delivering the `kernel-rt` packages would be to create an additional ostree commit in the `machine-os-content` image which the nodes would rebase to.  Currently, there is a single ostree repo with a single ostree commit in the `machine-os-content` image.  It is possible to create a second commit with the `kernel-rt` packages which would be used instead of the vanilla kernel.  While this is a cleaner, more pure ostree implementation of delivering a different set of packages, it was determined that the work required in the build pipeline would be non-trivial.

The intent is to pursue the current proposal and perhaps revisit this alternative as we gain experience handling different package sets for nodes in the cluster.