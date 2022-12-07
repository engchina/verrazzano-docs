---
title: "OpenSearch"
description: "Learn how to customize your OpenSearch cluster configuration"
aliases:
    - /docs/customize/elasticsearch
linkTitle: OpenSearch
weight: 10
draft: false
---

Verrazzano supports two cluster topologies for an OpenSearch cluster:
- A single-node cluster (master, ingest, and data roles performed by a single node).
- A multi-node cluster configuration with separate master, data, and ingest nodes.

[Installation Profiles]({{< relref "/docs/setup/install/profiles.md" >}}) describes the default OpenSearch cluster
configurations provided by Verrazzano.  
## Plan cluster topology

  You can start with an initial estimate on your hardware needs. These recommendations will serve you with the initial educated estimates but for ideal sizing, you need to test them with representative workloads and monitor their performance and then reiterate.

- `Storage Requirements`

   Minimum Storage Requirement = Source data(log size per day * retention period(days to store your data) * (1 + shard replicas) )* 1.45


- `Memory`

   8 GiB of memory for every 100 GiB of your storage requirement for lighter workloads.


- `JVM heap memory`

   The heap size is the amount of RAM allocated to the Java Virtual Machine of an OpenSearch node. The OpenSearch process is very memory intensive and close to 50% of the memory available on a node should be allocated to JVM. The JVM machine uses memory because the Lucene process needs to know where to look for index values on disk. The other 50% is required for the file system cache which keeps data that is regularly accessed in memory.
   As a general rule, you should set `-Xms` and `-Xmx` to the SAME value, which should be 50% of your total available RAM subject to a maximum of (approximately) 31GB.


- `CPU`
   
  Hardware requirements vary dramatically by workload, but 2 vCPU cores for every 100 GiB of your storage requirement is good for lighter workloads. But in case you have heavier workloads, this might not be sufficient.


- `Shard Size`

  For logging, shard sizes between 10 and 50 GB usually perform well.
  For search-intensive operations, 10-25 GB is usually a good shard size. Overall, it is a best practice that OpenSearch shard size should not go above 50GiB for a single shard. You will have to reindex your data when the shards exceed 50 GiB.


- `Primary shards Count`

   Approximate Number of Primary Shards = (Source Data + Room to Grow) * 1.1) / Desired Shard Size
   
   This equation helps compensate for data growth over time.



## Recommended Alarms

- `OSDataNodeFilesystemSpaceFillingUp` To indicate that the OpenSearch average disk usage has crossed a specific threshold. It will help you decide to add more data nodes or delete old indices if required to free up some space.

   ```yaml
   alert: OSDataNodeFilesystemSpaceFillingUp
  annotations:
    runbook_url: <link to runbook>
    summary: Opensearch average disk usage reached over 75%.
  expr: |-
    1 - (opensearch_fs_total_available_bytes{node=~".*data.*"}/ opensearch_fs_total_total_bytes) > .75
  for: 30m
  labels:
     severity: warning
  ```
    

## Configure cluster topology

You can customize the node characteristics of your OpenSearch cluster by using the
[spec.components.opensearch.nodes](/docs/reference/api/vpo-verrazzano-v1beta1/#install.verrazzano.io/v1beta1.OpenSearchNode)
field in the Verrazzano custom resource.  When installing or upgrading Verrazzano, you can use this field to
define an OpenSearch cluster using node groups.

The following example overrides the `dev` installation profile, OpenSearch configuration (a single-node cluster with
1Gi of memory and ephemeral storage) to use a multi-node cluster (three master nodes, and three combination data/ingest nodes) with persistent storage.

```yaml
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: custom-opensearch-example
spec:
  profile: dev
  components:
    opensearch:
      nodes:
        - name: master
          replicas: 3
          roles:
            - master
          storage:
            size: 50Gi
          resources:
            requests:
              memory: 1.5Gi
        - name: data-ingest
          replicas: 3
          roles:
            - data
            - ingest
          storage:
            size: 100Gi
          resources:
            requests:
              memory: 1Gi
        # Override the default node groups because we are providing our own topology.
        - name: os-master
          replicas: 0
        - name: os-data
          replicas: 0
        - name: os-ingest
          replicas: 0
```

Listing the pods and persistent volumes in the `verrazzano-system` namespace for the previous configuration
shows the expected nodes are running with the appropriate data volumes.

```
$ kubectl get pvc,pod -l verrazzano-component=opensearch -n verrazzano-system

# Sample output
NAME                                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/opensearch-master-vmi-system-master-0      Bound    pvc-9ace042a-dd68-4975-816d-f2ca0dc4d9d8   50Gi       RWO            standard       5m22s
persistentvolumeclaim/opensearch-master-vmi-system-master-1      Bound    pvc-8bf68c2c-235e-4bd5-8741-5a5cd3453934   50Gi       RWO            standard       5m21s
persistentvolumeclaim/opensearch-master-vmi-system-master-2      Bound    pvc-da8a48b1-5762-4669-98f0-8479f30043fc   50Gi       RWO            standard       5m21s
persistentvolumeclaim/vmi-system-data-ingest                     Bound    pvc-7ad9f275-632b-4aac-b7bf-c5115215937c   100Gi      RWO            standard       5m23s
persistentvolumeclaim/vmi-system-data-ingest-1                   Bound    pvc-8a293e51-2c20-4cae-916b-1ce46a780403   100Gi      RWO            standard       5m23s
persistentvolumeclaim/vmi-system-data-ingest-2                   Bound    pvc-0025fcef-1d8c-4307-977c-3921545c6730   100Gi      RWO            standard       5m22s

NAME                                                   READY   STATUS     RESTARTS   AGE
pod/coherence-operator-6ffb6bbd4d-bpssc                1/1     Running    1          8m2s
pod/fluentd-ndshl                                      2/2     Running    0          5m51s
pod/oam-kubernetes-runtime-85cfd899d8-z9gv6            1/1     Running    0          8m14s
pod/verrazzano-application-operator-5fbcdf6655-72tw9   1/1     Running    0          7m49s
pod/verrazzano-authproxy-5f9d479455-5bvvt              2/2     Running    0          7m43s
pod/verrazzano-console-5b857d7b47-djbrk                2/2     Running    0          5m51s
pod/verrazzano-monitoring-operator-b4b446567-pgnfw     2/2     Running    0          5m51s
pod/vmi-system-data-ingest-0-5485dcd95d-rkhvk          2/2     Running    0          5m21s
pod/vmi-system-data-ingest-1-8d7db6489-kdhbv           2/2     Running    1          5m21s
pod/vmi-system-data-ingest-2-699d6bdd9c-z7nzx          2/2     Running    0          5m21s
pod/vmi-system-grafana-7947cdd84b-b7mks                2/2     Running    0          5m21s
pod/vmi-system-kiali-6c7bd6658b-d2zq9                  2/2     Running    0          5m37s
pod/vmi-system-opensearchDashboards-7d47f65dfc-zhjxp   2/2     Running    0          5m21s
pod/vmi-system-master-0                                2/2     Running    0          5m21s
pod/vmi-system-master-1                                2/2     Running    0          5m21s
pod/vmi-system-master-2                                2/2     Running    0          5m21s
pod/weblogic-operator-666b548749-lj66t                 2/2     Running    0          7m48s
```

Running the command `kubectl describe pod -n verrazzano-system vmi-system-data-ingest-0-5485dcd95d-rkhvk` shows the
requested amount of memory.

```
Containers:
  os-data:
    ...
    Requests:
      memory:   1Gi
```

## Configure Index State Management policies

[Index State Management](https://opensearch.org/docs/1.3/im-plugin/ism/index/) policies configure OpenSearch to manage the data in your indices.
Policies can be used to automatically rollover and prune old data, preventing your OpenSearch
cluster from running out of disk space.

The following policy example configures OpenSearch to manage indices matching the pattern `my-app-*`. The data in these indices will be
automatically pruned every 14 days, and will be rolled over if an index meets at least one of the following criteria:
- Is three or more days old
- Contains 1,000 documents or more
- Is 10GB in size or larger

```yaml
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: custom-opensearch-example
spec:
  profile: dev
  components:
    opensearch:
      policies:
        - policyName: my-app
          indexPattern: my-app-*
          minIndexAge: 14d
          rollover:
            minIndexAge: 3d
            minDocCount: 1000
            minSize: 10Gb
```