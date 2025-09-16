# Keycloak Cluster Configuration: Comprehensive Guide

### Overview

This guide explains how Keycloak clustering works, how discovery is configured, and provides production‑ready configuration files for the most common environments. You can copy these files as-is and adapt hostnames, ports, and credentials.

---

### How Keycloak Clustering Works

- Keycloak shares login sessions and other state through an Infinispan distributed cache.
- Nodes “discover” peers using a JGroups stack. If nodes can discover each other and share the same cluster identity, they will form a single cluster and share sessions.
- The cluster you get depends on four things:
    
    1) cache mode: local or ispn
    
    2) discovery stack: tcp, udp, or kubernetes
    
    3) cluster identity: cluster name and the discovery endpoints
    
    4) network reachability: whether the JGroups ports can reach other nodes
    

---

### Quick Decision Matrix

- Don’t cluster at all: cache=local
- VMs/bare‑metal where multicast is not available: TCP stack
- VMs where multicast exists and is isolated: UDP stack
- Kubernetes: kubernetes stack (or Keycloak Operator)

---

### 1) No Clustering (Single Node Semantics)

Use this if you want nodes NOT to join any cluster.

keycloak.conf

```
# Core
hostname=[keycloak.example.com](http://keycloak.example.com)
http-enabled=false
https-port=8443

# Database
db=postgres
db-url=jdbc:postgresql://db:5432/keycloak
db-username=keycloak
db-password=strong-password

# Cache (disable clustering)
cache=local

# TLS
https-certificate-file=/opt/keycloak/conf/tls/tls.crt
https-certificate-key-file=/opt/keycloak/conf/tls/tls.key
```

Environment variable equivalents

```
KC_CACHE=local
```

---

### 2) One Cluster on VMs/Bare‑metal Using TCP (Recommended)

TCP works reliably through firewalls, NAT, and networks without multicast. All nodes use the same jgroups-tcp.xml.

keycloak.conf (same on all nodes)

```
hostname=[auth.example.com](http://auth.example.com)
http-enabled=false
https-port=8443

# Database
db=postgres
db-url=jdbc:postgresql://db:5432/keycloak
db-username=keycloak
db-password=strong-password

# Infinispan (enable clustering)
cache=ispn
cache-stack=tcp
cache-config-file=/opt/keycloak/conf/jgroups-tcp.xml

# Optional logical cluster name (should match JGroups config expectations)
kc.spi.connections-infinispan.cluster-name=kc-prod-cluster

# TLS
https-certificate-file=/opt/keycloak/conf/tls/tls.crt
https-certificate-key-file=/opt/keycloak/conf/tls/tls.key
```

/opt/keycloak/conf/jgroups-tcp.xml (shared by all nodes)

```xml
<config xmlns="urn:org:jgroups">
  <!-- Transport: TCP on port 7800 -->
  <TCP bind_addr="${jgroups.bind_addr:0.0.0.0}" bind_port="7800"
       recv_buf_size="20M" send_buf_size="640K"
       max_bundle_size="64K" max_bundle_timeout="30"/>

  <!-- Static peer discovery. Replace hostnames with your nodes. -->
  <TCPPING async_discovery="true"
           initial_hosts="${jgroups.tcpping.initial_hosts:kc1:7800,kc2:7800,kc3:7800,kc4:7800}"
           port_range="0" num_initial_members="2"/>

  <MERGE3 min_interval="10000" max_interval="30000"/>
  <FD_SOCK2/>
  <FD_ALL3 timeout="12000" interval="3000"/>
  <VERIFY_SUSPECT timeout="1500"/>
  <BARRIER/>
  <pbcast.NAKACK2 use_mcast_xmit="false"/>
  <UNICAST3/>
  <pbcast.STABLE stability_delay="1000"/>
  <pbcast.GMS print_local_addr="false" join_timeout="20000"/>
  <MFC max_credits="2M" min_threshold="0.4"/>
  <FRAG3 frag_size="60K"/>
  <RSVP timeout="60000"/>
</config>
```

Runtime options and firewall

- Open TCP 7800 between cluster nodes.
- Optionally pass -Djgroups.tcpping.initial_hosts=kc1:7800,kc2:7800,... per node instead of editing the XML.

---

### 3) One Cluster on VMs Using UDP Multicast

Use only if multicast is supported and properly isolated. Isolate clusters by changing mcast_addr and/or mcast_port.

keycloak.conf

```
hostname=[auth.example.com](http://auth.example.com)
http-enabled=false
https-port=8443

# Database
db=postgres
db-url=jdbc:postgresql://db:5432/keycloak
db-username=keycloak
db-password=strong-password

# Infinispan
cache=ispn
cache-stack=udp
cache-config-file=/opt/keycloak/conf/jgroups-udp.xml

kc.spi.connections-infinispan.cluster-name=kc-prod-cluster
```

/opt/keycloak/conf/jgroups-udp.xml

```xml
<config xmlns="urn:org:jgroups">
  <UDP mcast_addr="${jgroups.udp.mcast_addr:230.0.0.1}"
       mcast_port="${jgroups.udp.mcast_port:45566}"
       bind_addr="${jgroups.bind_addr:0.0.0.0}"
       ucast_recv_buf_size="20M" ucast_send_buf_size="640K"
       mcast_recv_buf_size="25M" mcast_send_buf_size="640K"
       max_bundle_size="64K" max_bundle_timeout="30"/>

  <PING/>
  <MERGE3 min_interval="10000" max_interval="30000"/>
  <FD_SOCK2/>
  <FD_ALL3 timeout="12000" interval="3000"/>
  <VERIFY_SUSPECT timeout="1500"/>
  <BARRIER/>
  <pbcast.NAKACK2 use_mcast_xmit="true"/>
  <UNICAST3/>
  <pbcast.STABLE stability_delay="1000"/>
  <pbcast.GMS print_local_addr="false" join_timeout="20000"/>
  <MFC max_credits="2M" min_threshold="0.4"/>
  <FRAG3 frag_size="60K"/>
  <RSVP timeout="60000"/>
</config>
```

Create a second isolated cluster (Cluster B)

- Set jgroups.udp.mcast_addr=230.0.0.2 and/or a different mcast_port.
- Optionally change kc.spi.connections-infinispan.cluster-name to kc-prod-b.

Firewall

- Allow the chosen multicast group/port between nodes; block others.

---

### 4) Two Isolated Clusters on VMs Using TCP (A and B)

Give Cluster A and B separate initial hosts and logical names.

Cluster A keycloak.conf

```
hostname=[keycloak-a.example.com](http://keycloak-a.example.com)
cache=ispn
cache-stack=tcp
cache-config-file=/opt/keycloak/conf/jgroups-tcp-a.xml
kc.spi.connections-infinispan.cluster-name=kc-cluster-a
```

/opt/keycloak/conf/jgroups-tcp-a.xml

```xml
<config xmlns="urn:org:jgroups">
  <TCP bind_addr="${jgroups.bind_addr:0.0.0.0}" bind_port="7800"/>
  <TCPPING initial_hosts="kca1:7800,kca2:7800" async_discovery="true" port_range="0"/>
  <FD_SOCK2/>
  <FD_ALL3 timeout="12000" interval="3000"/>
  <VERIFY_SUSPECT timeout="1500"/>
  <pbcast.NAKACK2/>
  <UNICAST3/>
  <pbcast.STABLE/>
  <pbcast.GMS/>
  <FRAG3/>
</config>
```

Cluster B keycloak.conf

```
hostname=[keycloak-b.example.com](http://keycloak-b.example.com)
cache=ispn
cache-stack=tcp
cache-config-file=/opt/keycloak/conf/jgroups-tcp-b.xml
kc.spi.connections-infinispan.cluster-name=kc-cluster-b
```

/opt/keycloak/conf/jgroups-tcp-b.xml

```xml
<config xmlns="urn:org:jgroups">
  <TCP bind_addr="${jgroups.bind_addr:0.0.0.0}" bind_port="7800"/>
  <TCPPING initial_hosts="kcb1:7800,kcb2:7800" async_discovery="true" port_range="0"/>
  <FD_SOCK2/>
  <FD_ALL3 timeout="12000" interval="3000"/>
  <VERIFY_SUSPECT timeout="1500"/>
  <pbcast.NAKACK2/>
  <UNICAST3/>
  <pbcast.STABLE/>
  <pbcast.GMS/>
  <FRAG3/>
</config>
```

Network isolation

- Ensure A nodes cannot connect to B nodes on TCP 7800 (and vice versa).

---

### 5) Kubernetes: One Cluster Using Kubernetes Stack

Use a headless Service for discovery. Pods that match the same Service/selector join the same cluster.

keycloak.conf (mounted via ConfigMap or passed as env)

```
hostname=[auth.example.com](http://auth.example.com)
http-enabled=false
https-port=8443

# Database
db=postgres
db-url=jdbc:postgresql://postgresql:5432/keycloak
db-username=keycloak
db-password=strong-password

# Infinispan discovery on K8s
cache=ispn
cache-stack=kubernetes
kc.spi.connections-infinispan.cluster-name=kc-k8s-cluster
```

Headless Service (discovery)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak-headless
  labels:
    app: keycloak
spec:
  clusterIP: None
  selector:
    app: keycloak
  ports:
    - name: jgroups
      port: 7800
      targetPort: 7800
```

StatefulSet (excerpt)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: keycloak
spec:
  serviceName: keycloak-headless
  replicas: 3
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: [quay.io/keycloak/keycloak:latest](http://quay.io/keycloak/keycloak:latest)
          args: ["start"]
          env:
            - name: KC_CACHE
              value: "ispn"
            - name: KC_CACHE_STACK
              value: "kubernetes"
            - name: KC_SPI_CONNECTIONS_INFINISPAN_CLUSTER_NAME
              value: "kc-k8s-cluster"
          ports:
            - containerPort: 7800   # JGroups
```

Isolation

- To create a second independent cluster, define a second headless Service with different labels and a second StatefulSet that selects them.

---

### 6) Kubernetes with Keycloak Operator (HA Pattern)

- The Operator simplifies HA deployments and can integrate an external Infinispan service for robust caching.
- High-level values are set in the Keycloak custom resource; the Operator configures discovery and cache for you.

Keycloak CR (conceptual example)

```yaml
apiVersion: [k8s.keycloak.org/v2alpha1](http://k8s.keycloak.org/v2alpha1)
kind: Keycloak
metadata:
  name: kc-prod
spec:
  instances: 3
  http:
    enabled: false
  hostname:
    hostname: [auth.example.com](http://auth.example.com)
  db:
    vendor: postgres
    host: postgresql
    usernameSecret:
      name: kc-db-user
    passwordSecret:
      name: kc-db-pass
  features:
    - clustering
  additionalOptions:
    - name: cache
      value: ispn
    - name: cache-stack
      value: kubernetes
```

---

### 7) Operational Checklist

- Load balancer: enable sticky sessions if possible for smoother UX, but failover works due to distributed cache.
- Health checks: enable readiness/liveness probes.
- Ports:
    - TCP stack: open TCP 7800 between nodes
    - UDP stack: open multicast port and group
    - K8s stack: Service/selector determines peers
- Keep identical cache-config-file per cluster.
- Back up keycloak.conf and JGroups XMLs; keep them in source control with environment overlays.

---

### 8) Troubleshooting

- Nodes aren’t clustering:
    - Verify KC_CACHE=ispn and KC_CACHE_STACK chosen correctly
    - Check discovery: TCPPING initial_hosts, UDP multicast reachability, or K8s headless Service/labels
    - Firewalls and SELinux/AppArmor can block ports
- Too many nodes in one cluster:
    - They share discovery endpoints or multicast group
    - Use different initial_hosts or multicast address/port or K8s Services
- Sessions not shared:
    - Confirm all nodes are actually in the same JGroups view
    - Check for mixed cache stacks or mismatched XML files

---

### 9) Env Var Quick Reference

- KC_CACHE=local|ispn
- KC_CACHE_STACK=tcp|udp|kubernetes
- KC_CACHE_CONFIG_FILE=/opt/keycloak/conf/jgroups.xml
- KC_SPI_CONNECTIONS_INFINISPAN_CLUSTER_NAME=kc-name

---

### 10) Validation Commands

- Check cluster view in logs:

```
kubectl logs statefulset/keycloak | grep -i "cluster"   # K8s
journalctl -u keycloak | grep -i "view"                 # systemd
```

- Curl stickiness sanity check (behind LB):

```
for i in {1..5}; do curl -s -I [https://auth.example.com/realms/master](https://auth.example.com/realms/master) | grep -i server; done
```

- Change node set, observe new JGroups view formed.

---

### 11) Security Notes

- Prefer TCP stack unless multicast is well‑managed.
- Limit JGroups ports to the intended peers only.
- Rotate DB credentials and TLS certificates regularly.
- Keep Keycloak versions consistent across the cluster to avoid protocol mismatches.