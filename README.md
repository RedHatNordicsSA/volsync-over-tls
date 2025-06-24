# volsync-over-tls
Test and example of VolSync based replication of PVC using rsync over TLS.

The purpose of this setup is to enable PersistentVolume replication between two OpenShift clusters with no extra infrastructure depeendency such as MetalLB, Elastic LB or sub mariner.
To do this, volume replication is done via rsync over TLS towards a passthrough Route on the destination cluster.

## Prerequisite 
- Two OpenShift clusters.
- The source cluster can reach the Router IP of the destination cluster on port 443.
- Install VolSync on both clusters via the OperatorHub. If you have ACM, there is also a ManagedClusterAddOn available.

## (Optional) Test data.
Create a PostegreSQL Deployment with a PVC. Make use of the `generate_series()` and `random()` function to insert dummy data in thousands or millions of rows in a DB.

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
  annotations:
    haproxy.router.openshift.io/ip_whitelist: <Egress IP or cluster IP range of the source cluster>
spec:
  host: database-volsync.<Ingress_Domain> 
  port:
    targetPort: <VolSync Service Port>
  to:
    kind: Service
    name: <VolSync Service Name>
  tls:
    termination: passthrough
```

### The Pre Shared Key
The ReplicatioDestination once created will have a Secret containing a PSK, you will need that value. To find the Secret name, look at the ReplicationDestination `.status.rsyncTLS.keySecret`. Secret name should start with `volsync-rsync-dst-src-`.

## Setup of the source cluster
## The PSK Secret
Create in your source Namespace a Secret as found in the previous step in the destination cluster.
## The ReplicationSource CR
```yaml
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: my-database
  namespace: y-database
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
While the replication is running, monitor the network load on the Routers of your destination cluster.
