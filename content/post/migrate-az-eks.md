---
title: Migrating K8s PVs from AZs in EKS
date: 2024-03-08T17:20:00+00:00
tags:
- K8s
- AWS
- EKS
categories:
- Systems & Infrastructure
---
In the midst of our beta stage, we realized we didn't really need a multi-AZ EKS cluster in a beta stage.  

{{<admonition title="This post is AWS-specific"/>}}

Long story short there is cross-AZ traffic, which gets very expensive very quickly when you have many databases/services talking with each other.  
Which was our case, of course.  

There are better solutions of course, like instructing k8s to deploy all pods talking with each other in the same AZ (and reducing the possible damage in case an entire AZ goes down).  
In our case we wanted to keep the number of instance manually managed.  
Scaling up multiple node groups and handling the scheduling decision was something we just didn't want to do as it was necessary at the time.  

So I wrote a script to migrate the PVs across availabiliy zones.  
The script might actually be adapted to work cross-region (with the exception that the EC2 volume needs to be copied over the other region)

{{<admonition title="This migration is offline">}}
This means that there will be downtime during the migration period.  
It is also advised to disable/suspend any operation which might attempt to bring the workload related to the PVs back up (it is supposed to be scaled down)
{{</admonition>}}

## Edit the storageclass to default 
Before starting the migration, I've edited the storageclass, adding a rule to limit the available AZs where I want to deploy stuff (just in one AZ)

Specifically, I've added the `allowedTopologies` snippet
```yaml
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - eu-west-1a
```

This is what the (neat) storage class looks like
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: gp3
provisioner: kubernetes.io/aws-ebs
allowVolumeExpansion: true
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - eu-west-1a
parameters:
  fsType: ext4
  type: gp3
reclaimPolicy: Delete
```


## The script

The script has some dependencies. It could work without, I just didn't want to.  
Specifically, `jq` (which I hope you already know), and `kubectl-neat` (`neat` from now on).  
`neat` is a great plugin, as it allows us to remove stuff from a K8s object manifest, allowing us to have a __neater__ YAML structure.  
You can install it with `kubectl-krew` (which is basically an extension manager for kubectl).  
How to install those is out of scope for now

### Usage   
The idea is to paste the script in a file, edit the source and destination AZs (the first lines of the script), and just run it.  

The script prints out all the executed commands and set variables. This means that if for some reason the script crashes (and it will at the first command which does not return successfully), it's possible to set all the previously set variables and run the remaining commands manually.  

Not ideal, but at least it's possible to recover some data.  

### Concept

The script lists all the Persistent Volumes in a cluster which are also located in the source availability zone (and whose name does not contain `prometheus`). Then, for each persistent volume:  

Gets the EBS volume ID, and the relative PVC. It also tries to find a workload related to the PV.  
If a workload was found, it tries to scale it down to 0, so that the migration does not corrupt/lose data.  

Of course this will cause downtime.  

Then the script proceeds to create a snapshot of the volume and dump it in a new volume in the destination AZ.  
Once the volume is created in the new region, the old PV (and thus PVC) is deleted and recreated, pointing to the new EBS volume.  

Finally the the deployment is scaled up again.

### The code

```bash
#!/bin/bash

SOURCE_AZ="eu-west-1b"
DESTINATION_AZ="eu-west-1a"

set -euxo pipefail

which jq
which kubectl-krew
which kubectl-neat

# TODO: Filter out RWM

# for PV_NAME in "$(kubectl get pv -l topology.kubernetes.io/zone=${SOURCE_AZ} -o custom-columns=name:.spec.claimRef.name --no-headers)"; do
for PV_NAME in $(kubectl get pv -l topology.kubernetes.io/zone=${SOURCE_AZ} --no-headers | grep -v prometheus- | awk '{print $1}'); do
   # VID=$(aws ec2 describe-volumes | jq -r ".Volumes[] | select((.Tags != null) and (select(.Tags[] | (.Key==\"CSIVolumeName\") and (.Value==\"$PV\")))) | .VolumeId")
    VID=$(kubectl get pv $PV_NAME -o jsonpath="{.spec.awsElasticBlockStore.volumeID}" | cut -d '/' -f3)
    if [[ "$VID" == "" ]]; then
        echo "Could not get VolumeId";
        continue;
    fi
    echo "Matched $PV_NAME with EBS Volume: $VID"

    read -r PVC_NAMESPACE PVC_NAME < <(kubectl get pv $PV_NAME -o jsonpath="{.spec.claimRef.namespace} {.spec.claimRef.name}" | rev | cut -c 1- | rev)
    # kubectl get pods -n $PVC_NAMESPACE -o=json | jq -c ".items[] | {name: .metadata.name, claimName: .spec | select(has(\"volumes\")).volumes[] | select(has(\"persistentVolumeClaim\")).persistentVolumeClaim.claimName} | select(.claimName == \"$PVC_NAME\")"
    SCALING=1
    read -r KIND DEPLOY_NAME < <(kubectl get pods -n $PVC_NAMESPACE -o=json | jq -j ".items[] | 
        select(            
            select( 
                .spec | has(\"volumes\") 
            ).spec.volumes[] 
                | 
                    select(
                        select(
                            has(\"persistentVolumeClaim\")
                        ).persistentVolumeClaim.claimName == \"$PVC_NAME\" 
                    )
        ) | .metadata.ownerReferences[] | .kind,\" \", .name" ) || test -n "$KIND" || SCALING=0
    
    if [[ $SCALING != 0 ]]; then
        PREVIOUS_REPLICA_COUNT=$(kubectl get $KIND $DEPLOY_NAME -n $PVC_NAMESPACE -o jsonpath='{.status.replicas}')

        # This is an hack; We don't care about RSs
        if [[ "$KIND" == "ReplicaSet" ]]; then
            KIND="Deployment"
            REGEX_DEPLOY_NAME="(.+)-[0-9a-z]+$"
            if [[ $DEPLOY_NAME =~ $REGEX_DEPLOY_NAME ]]
            then
              DEPLOY_NAME="${BASH_REMATCH[1]}"
            else
            	# Apparently in another cluster the replicaset had 8 chars, not 12, so this might not work out
            	DEPLOY_NAME=$(echo $DEPLOY_NAME | rev | cut -c 12- | rev)
            	echo "Got deploy name in a legacy way. Stuff might not work out" 
            fi
        fi

        echo "Scaling down the $KIND $DEPLOY_NAME tied to the PV from $PREVIOUS_REPLICA_COUNT to 0"
        kubectl scale -n $PVC_NAMESPACE $KIND $DEPLOY_NAME --replicas 0
        kubectl rollout status -n $PVC_NAMESPACE $KIND $DEPLOY_NAME 
    fi

    echo "Creating snapshot of the PV $PV_NAME"
    SNAPSHOT_ID=$(aws ec2 create-snapshot --volume-id $VID --query SnapshotId --output text --description "Backup of pv $PV_NAME of PVC $PVC_NAME in namespace $PVC_NAMESPACE for $KIND $DEPLOY_NAME")
    aws ec2 wait snapshot-completed --snapshot-ids $SNAPSHOT_ID
    echo "Created snapshot: $SNAPSHOT_ID of the EBS Volume: $VID"

    echo "Creating the new volume in the destination AZ ($DESTINATION_AZ)"
    NEW_VID=$(aws ec2 create-volume --availability-zone $DESTINATION_AZ --volume-type gp3 --snapshot-id $SNAPSHOT_ID --query VolumeId --output text)
    aws ec2 wait volume-available --volume-ids $NEW_VID
    echo "New volume $NEW_VID created"

    echo "Deleting old PV $PV_NAME "
    # spec.persistentvolumesource is immutable after creation
    OLD_PVC=$(kubectl get pvc -n $PVC_NAMESPACE $PVC_NAME -o yaml | kubectl neat)
    NEW_PV_YAML=$(kubectl get pv $PV_NAME -o yaml | sed "s/$SOURCE_AZ/$DESTINATION_AZ/" | sed "s/$VID/$NEW_VID/" | kubectl neat)
    # kubectl patch pv $PV_NAME --type merge -p '{"spec":{"persistentVolumeReclaimPolicy": "Delete"}}'
    kubectl delete pvc -n $PVC_NAMESPACE $PVC_NAME 
    # kubectl delete pv $PV_NAME
    echo "$NEW_PV_YAML" | kubectl apply -f -
    kubectl patch pv $PV_NAME -p '{"spec": {"claimRef":null}}'
    echo "$OLD_PVC" | grep -v bind-completed | grep -v volume.kubernetes.io/selected-node | grep -v pv.kubernetes.io/bound-by-controller | sed "s/ gp2/ gp3/" | kubectl apply -f -


    if [[ $SCALING != 0 ]]; then
        echo "Scaling $KIND back up"
        kubectl scale -n $PVC_NAMESPACE $KIND $DEPLOY_NAME --replicas $PREVIOUS_REPLICA_COUNT 
        kubectl rollout status -n $PVC_NAMESPACE $KIND $DEPLOY_NAME 
    fi

    # Uncommenting the following lines allows for deletion of the snapshot and/or previous volume
    # echo "Deleting the snapshot"
    # aws ec2 delete-snapshot --snapshot-id $SNAPSHOT_ID
    # echo "Deleting the old PV (if it was not in "delete"  reclaim policy)" 
    # aws ec2 delete-volume --volume-id $VID
done
```

Needless to say, I take no responsability with the use of this script, and possible conseguence (e.g. dataloss, loss of account, or anything else).
