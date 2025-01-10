# Migrating EBS-backed StatefulSet Across Kubernetes Clusters

## Background

TODO

## Demo A: Migrating StatefulSet through EBS Snapshots

TODO

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

You can confirm that your clusters are setup correctly with the following steps:

TODO


### 1: Create StatefulSet on cluster alpha

```
aws eks update-kubeconfig --name ebs-demo-alpha --region us-east-1
kubectl create -f classes.yaml
kubectl create -f statefulset.yaml
```

### 2: Backup your EBS Volumes by creating Kubernetes VolumeSnapshots

Create the snapshots:

```
kubectl -f snapshot-create.yaml
```

Get the EBS Snapshot IDs from the VolumeSnapshotContent status.snapshotHandle fields associated with each VolumeSnapshot.  

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

Edit `snapshot-import.yaml` to restore from EBS Snapshots created in step 2. The following will use the `sed` tool to replace EDITME0 and EDITME1 for you. 

```
sed "s/EDITME0/$SNAP_ID_0/g" ./snapshot-import.yaml -i
sed "s/EDITME1/$SNAP_ID_1/g" ./snapshot-import.yaml -i
```

Statically import VolumeSnapshots from step 2 to cluster bravo, pre-create PVC resources based on those snapshots, and finally create the StatefulSet (which will wait for pre-created PVCs to finish)

```
aws eks update-kubeconfig --name ebs-demo-bravo --region us-east-1
kubectl create -f classes.yaml
kubectl create -f snapshot-import.yaml
kubectl create -f statefulset.yaml
```

Note that the volumes contain timestamps from Step 1:

```
kubectl exec web-0 -- cat /data/out.txt
```

## Cleanup

TODO 