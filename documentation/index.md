---
title: Documentation
section: Overview
layout: doc
---

Skydive is an open source real-time network topology and protocols analyzer.
It aims to provide a comprehensive way of understanding what is happening in
the network infrastructure.

Skydive agents collect topology informations and flows and forward them to a
central agent for further analysis. All the informations are stored in an
Elasticsearch database.

Skydive is SDN-agnostic but provides SDN drivers in order to enhance the
topology and flows informations.

## Topology Probes

Topology probes are used to construct the graph comprising:
* Graph nodes: depicted as cycles (in contracted form) or areas (in expanded form)
* Graph edges: depicted as strait lines (with arrowheads when directional)

### Agent Topology Probes

Probes which extract topological information from the host about host residing entities are agent probes:
* Docker (docker)
* Ethtool (ethtool)
* LibVirt (libvirt)
* LLDP (lldp)
* Lxd (lxd)
* NetLINK (netlink)
* NetNS (netns)
* Neutron (neutron)
* OVSDB (ovsdb)
* Opencontrail (opencontrail)
* runC (runc)
* Socket Information (socketinfo)
* VPP (vpp)

### Analyzer Topology Probes

Probes which extract topological information from a remote global entity are analyzer probes:
* Istio (istio)
* Kubernetes (k8s)
* OVN (ovn)

### K8s

The k8s probe provides topological information.

Graph nodes:
* general: cluster, namespace
* compute: node, pod, container
* storage: persistentvolumeclaim (pvc), persistentvolume (pv), storageclass
* network: networkpolicy, service, endpoints, ingress
* deployment: deployment, statefulset, replicaset, replicationcontroller, cronjob, job
* configuration: configmap, secret

Graph edges:
* k8s-k8s ownership (e.g. k8s.namespace -- k8s.pod)
* k8s-k8s relationship (e.g. k8s.service -- k8s.pod)
* k8s-physical relationship (e.g. k8s.node -- host)

Graph node metadata:
* indexed fields: standard fields such as `Type`, `Name` plus k8s specific such as `K8s.Namespace`
* stored-only fields: the entire content of k8s resource stored under `K8s.Extra`

Graph node status:
* the `Status` node metadata field
* with values Up (white) / Down (red)
* currently implemented for resources: pod, persistentvolumeclaim (pvc) and persistentvolume (pv)

## Flow Probes supported

Flow probes currently implemented :

* sFlow
* AFPacket
* PCAP
* PCAP socket
* DPDK
* eBPF
* OpenvSwitch port mirroring


## Architecture

<p>
  <a href="/assets/images/documentation/architecture.png" data-lightbox="WebUI-1" data-title="Skydive Architecture">
    <center>
      <img src="/assets/images/documentation/architecture.png" width="500"/>
    </center>
  </a>
</p>

### Graph engine

Skydive relies on a event based graph engine, which means that notifications
are sent for each modification. Graphs expose notifications over WebSocket
connections. Skydive support multiple graph backends for the Graph. The `memory`
backend will be always used by agents while the backend for analyzers can be
choosen. Each modification is kept in the datastore so that we have a full
history of the graph. This is really useful to troubleshoot even if
interfaces do not exist anymore.

### Topology probes

Fill the graph with topology informations collected. Multiple probes fill the
graph in parallel. As an example there are probes filling graph with
network namespaces, netlink or OVSDB information.

### Flow table

Skydive keep a track of packets captured in flow tables. It allows Skydive to
keep metrics for each flows. At a given frequency or when the flow expires
(see the config file) flows are forwarded from agents to analyzers and then
to the datastore.

### Flow enhancer

Each time a new flow is received by the analyzer the flow is enhanced with
topology informations like where it has been captured, where it originates from,
where the packet is going to.

### Flow probes

Flow probes capture packets and fill agent flow tables. There are different
ways to capture packets like sFlow, afpacket, PCAP, etc.

### Gremlin engine

Skydive uses Gremlin language as its graph traversal language. The Skydive
Gremlin implementation allows to use Gremlin for flow traversal purpose.
The Gremlin engine can either retrieve informations from the datastore or from
agents depending whether the request is about something is the past or for live
monitoring/troubleshooting.

### Etcd

Skydive uses Etcd to store API objects like captures. Agents are watching Etcd
so that they can react on API calls.

### On-demand probes

This component watches Etcd and the graph in order to start captures. So when a
new capture is created by the API on-demande probe looks for graph nodes
matching the Gremlin expression, and if so, start capturing traffic.
