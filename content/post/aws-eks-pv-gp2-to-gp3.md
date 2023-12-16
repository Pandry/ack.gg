---
title: "Migrating EKS EBS PVs from GP2 to GP3"
date: 2023-12-16T15:34:23+02:00
tags:
- aws
- cloud
- k8s
categories:
- K8s
---

For older cluster (I did not check yet with new ones), AWS used to default to GP2 for the Persistent Volumes.  
In most of the cases in my opinion, it makes sense to migrate to GP3 volumes, as those are cheaper and with higher performance in most of my usecases.  
This might not apply to everybody, so please check in the [AWS documentation](https://aws.amazon.com/blogs/storage/migrate-your-amazon-ebs-volumes-from-gp2-to-gp3-and-save-up-to-20-on-costs/) if that's the case for you too!  

## The default StorageClass
In my case, my cluster(s) were updated from K8s 1.24 to 1.28 over time, but the migration didn't add the GP3 storage class, so I need to add it, but first I want to make the GP2 storage class **not** the default:  

```bash
kubectl patch sc gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
``` 

Then, I proceeded to add the GP3 Storage Class (I just dumped the gp2 storage class, patched it and cleaned the YAML file):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: gp3
parameters:
  fsType: ext4
  type: gp3
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```


## The migration script
I wrote a really simple script to migrate the volumes.  
The script also changes the Storage Class of the persistent volumes. This has no real effect, but I just find it handy to have it written on the PV.  
Keep in mind that the Storage Class of the PVC is an immutable field, and thus cannot (shouldn't) be changed.  
The operation is online, and thus doesn't create any downtime (in my experience, test it in a test cluster first).  

The requirements for the script are:  
- `aws` CLI installed and configured for the current account you want to access to. The credentials/role need to have access to the volume list and needs to be able to update the type of the EBS volume.
- `kubectl` configured to work on the cluster you want to work on.
- `jq`

```bash
# Uncomment the set command to see what operations are being executed
# set -x

# List all the `gp2` volumes in the cluster. PVs are cluster-wide (non-namespaced resources), so this will update the whole cluster
for PV in $(kg pv | grep gp2 | awk '{print $1}'); do
  kubectl patch pv $PV -p '{"spec":{"storageClassName":"gp3"}}';
  VID=$(aws ec2 describe-volumes | jq -r ".Volumes[] | select((.Tags != null) and (select(.Tags[] | (.Key==\"CSIVolumeName\") and (.Value==\"$PV\")))) | .VolumeId");
  aws ec2 modify-volume --volume-id $VID --volume-type gp3 > /dev/null;
done
# set +x
```

That's it!  
You can monitor the status of the migration on the AWS CLI or on the AWS dashboard. It will take time for bigger or highly used volumes
