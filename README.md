# VolSync over TLS
Test and example of VolSync based replication of PVC using rsync over TLS.

The purpose of this setup is to enable PersistentVolume replication between two OpenShift clusters with no extra infrastructure dependency such as MetalLB, Elastic LB or sub mariner.
To do this, volume replication is done via rsync over TLS towards a passthrough Route on the destination cluster.

## Prerequisite 
- Two OpenShift clusters.
- The source cluster can reach the Router IP of the destination cluster on port 443.
- Install VolSync on both clusters via the OperatorHub. If you have ACM, there is also a ManagedClusterAddOn available.

## (Optional) Test data.
If you need a Deployment and PersistentVolumeClaim to validate this setup before moving on to actual workload.

Create a PostegreSQL Deployment with a PVC. Make use of the `generate_series()` and `random()` function to insert dummy data in thousands or millions of rows in a DB.

- On both clusters:
```yaml
oc create project my-database
```

- On the source cluster:

In that Namespace, deploy a PostgreSQL 15 (`15-el9` tag from postegrsql ImageStream) with a 10Gi PVC on the source cluster using the OpenShift template (https://github.com/sclorg/postgresql-container/).

Inside a Terminal in the PostgreSQL Pod or via `oc rsh`, generate some random data:
```shell
CREATE TEMP TABLE t AS SELECT generate_series(1, 6e7) x;
```
This should generate about 2GB of data on the PersistentVolumeClaim.

## Setup the destination cluster.
### The ReplicationDestination CR
```yaml
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: database-destination
  namespace: my-database
spec:
  rsyncTLS:
    accessModes:
    - ReadWriteOnce
    capacity: 10Gi
    copyMethod: Direct
    serviceType: ClusterIP
    storageClassName: <StorageClass name>
```
The Service with type: ClusterIP generated there will be used by the passthrough Route.

### The Route
```yaml
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: database-volsync
  namespace: my-database
spec:
  host: database-volsync.<Ingress_Domain> 
  port:
    targetPort: 8000
  to:
    kind: Service
    name: <VolSync Service Name>
  tls:
    termination: passthrough
```
The TLS PSK ensures authentication and if you want an added layer of security, you can add an ACL to the Route:
```yaml
metadata:
  annotations:
    haproxy.router.openshift.io/ip_whitelist: <Egress IP or cluster IP range of the source cluster>
```

### The Pre Shared Key
The ReplicatioDestination once created will have a Secret containing a PSK, you will need that value. To find the Secret name, look at the ReplicationDestination `.status.rsyncTLS.keySecret`. Secret name should start with `volsync-rsync-dst-src-`.

## Setup of the source cluster
## The PSK Secret
Create in your source Namespace a Secret copy of the one in the previous step in the destination cluster.
## The ReplicationSource CR
```yaml
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: my-database
  namespace: my-database
spec:
  sourcePVC: my-database-pvc
  trigger:
    schedule: "*/10 * * * *"
  rsyncTLS:
    keySecret: <Secret with PSK>
    address: <Host value of the Route created on the destination cluster>
    port: 443
    copyMethod: Clone    
```

## Monitor
While the replication is running, monitor the network load on the Routers of your destination cluster. One way to do that is by going in the OpenShift Console under Observe and choosing the `Kubernetes / Networking / Namespace (Pods)` then selecting the `openshift-ingress` Namespace.

The ReplicationSource and ReplicationDestination CRs `.status.latestMoverStatus` section will be updated after each sync.
