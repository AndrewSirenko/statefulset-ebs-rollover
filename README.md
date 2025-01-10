# Migrating EBS-backed StatefulSet Across Kubernetes Clusters

## Background

Roadmap:

- clusters: Contains cluster configurations that sets up EBS CSI Driver and Snapshot Controller add-ons `eksctl`. Should automatically setup required IAM roles too. 
- `classes.yaml`: Adds StorageClass and VolumeSnapshotClass to your cluster
- `statefulset.yaml`: Simple stateful workload with 2 replicas that current time to EBS volumes. Will create EBS-Backed PVCs if they do not exist on cluster already. 
- `snapshot-create.yaml`: Creates VolumeSnapshot resources from the stateful workload's EBS volumes.
- `snapshot-import.yaml`: Creates VolumeSnapshot resources from manually imported Snapshot ID, as well as PVCs that will restore from those snapshots. 

## Demo A: Migrating StatefulSet through EBS Snapshots

### Pre-requisites

First, clone this git repository. You will execute all demo commands from the root directory:

```
git clone https://github.com/AndrewSirenko/statefulset-ebs-rollover.git
cd statefulset-ebs-rollover
```

For this demo, you will need two separate EKS clusters, each with the EBS CSI Driver and Snapshot Controller add-ons installed.

If you have `eksctl` installed, you can use our sample cluster configurations in the `clusters` directory:

```
eksctl create -f ./cluster/cluster_alpha.yaml
eksctl create -f ./cluster/cluster_bravo.yaml
```

### 1: Create StatefulSet on cluster alpha

```
aws eks update-kubeconfig --name ebs-demo-alpha --region us-east-1
kubectl create -f classes.yaml
kubectl create -f statefulset.yaml
```

Within a few minutes, you should see that two Pods are `Running` and that 2 PersistentVolumeClaims have been `Bound`:

```
> kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          2m
web-1   1/1     Running   0          4m

> kubectl get pvc
NAME        STATUS   VOLUME                                     ...
ebs-web-0   Bound    pvc-b9eb698a-4b62-49e5-9f0a-2282a6d536d7   ...
ebs-web-1   Bound    pvc-bd893e5a-1c8e-4a7b-a2d4-cf034d601c81   ...
```

If your Pods are stuck in `Status: Pending` for more than 5 minutes, and your PVCs are not `Bound`, your cluster or EBS CSI Driver setup is wrong.  

### 2: Backup your EBS Volumes by creating Kubernetes VolumeSnapshots

Create the snapshots:

```
kubectl -f snapshot-create.yaml
```

Within a few minutes, 2 VolumeSnapshot resources should be `ReadyToUse` on your cluster:

```
> kubectl get vs
NAME                    READYTOUSE   SOURCEPVC  ...
ebs-volume-snapshot-0   true         ebs-web-0  ...
ebs-volume-snapshot-1   true         ebs-web-1  ...
```

Get the EBS Snapshot IDs from the VolumeSnapshotContent status.snapshotHandle fields associated with each VolumeSnapshot. You can try using the following commands, or look inside your AWS account EC2 > Snapshots. 

```
export SNAP_ID_0 SNAP_ID_1

tmp0=$(kubectl get vs | awk '$1 == "ebs-volume-snapshot-0" {print $6}')
SNAP_ID_0=$(kubectl get vsc "$tmp0" -o json | jq ".status.snapshotHandle")

tmp1=$(kubectl get vs | awk '$1 == "ebs-volume-snapshot-1" {print $6}')
SNAP_ID_1=$(kubectl get vsc "$tmp1" -o json | jq ".status.snapshotHandle")

echo $SNAP_ID_0
echo $SNAP_ID_1
```

### 3: Restore StatefulSet on cluster bravo 

Edit `snapshot-import.yaml`'s "EDITME0" and "EDITME1" to include the Snapshot IDs found in step 2. The following will use the `sed` tool to do this for you. 

```
sed "s/EDITME0/$SNAP_ID_0/g" ./snapshot-import.yaml -i
sed "s/EDITME1/$SNAP_ID_1/g" ./snapshot-import.yaml -i
```

Then you can statically import VolumeSnapshots from step 2 to cluster bravo, pre-create PVC resources based on those snapshots, and finally create the StatefulSet (which will wait for pre-created PVCs to finish)

```
aws eks update-kubeconfig --name ebs-demo-bravo --region us-east-1
kubectl create -f classes.yaml
kubectl create -f snapshot-import.yaml
kubectl create -f statefulset.yaml
```

After a few minutes, you should see that two Pods are `Running` and that 2 PersistentVolumeClaims have been `Bound`:

```
> kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          2m
web-1   1/1     Running   0          4m

> kubectl get pvc
NAME        STATUS   VOLUME                                     ...
ebs-web-0   Bound    pvc-b9eb698a-4b62-49e5-9f0a-2282a6d536d7   ...
ebs-web-1   Bound    pvc-bd893e5a-1c8e-4a7b-a2d4-cf034d601c81   ...
```

Validate that these volumes contain timestamps from Step 1:

```
kubectl exec web-0 -- cat /data/out.txt

❯ kubectl exec web-0 -- cat /data/out.txt
Fri Jan 10 19:09:23 UTC 2025
Fri Jan 10 19:09:28 UTC 2025
Fri Jan 10 19:09:33 UTC 2025
Fri Jan 10 19:25:53 UTC 2025
Fri Jan 10 19:25:58 UTC 2025
Fri Jan 10 19:26:03 UTC 2025
...
```

## Cleanup

Make sure you cleanup all Volumes, Snapshots, and EKS clusters to avoid fees. 

The following commands will delete all StatefulSets, volumes, and snapshots from your clusters:

```
aws eks update-kubeconfig --name ebs-demo-alpha --region us-east-1
kubectl delete sts --all
kubectl delete pvc --all
kubectl delete vs --all
aws eks update-kubeconfig --name ebs-demo-bravo --region us-east-1
kubectl delete sts --all
kubectl delete pvc --all
kubectl delete vs --all
```

Check EC2 console to ensure no volumes/snapshots remain.