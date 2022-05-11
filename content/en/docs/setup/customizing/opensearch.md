---
title: "Customize OpenSearch"
description: "Learn how to customize your OpenSearch cluster configuration"
aliases:
    - /docs/setup/customizing/elasticsearch
linkTitle: OpenSearch
weight: 8
draft: false
---

Verrazzano supports two cluster topologies for an OpenSearch cluster:
- A single-node cluster (master, ingest, and data roles performed by a single node).
- A multi-node cluster configuration with separate master, data, and ingest nodes.

[Installation Profiles](/docs/setup/install/profiles/) describes the default OpenSearch cluster
configurations provided by Verrazzano.  

## Configure cluster topology

You can customize the node characteristics of your OpenSearch cluster by using the
[spec.components.elasticsearch.nodes](/docs/reference/api/verrazzano/verrazzano/#opensearch-component)
field in the Verrazzano custom resource.  When installing or upgrading Verrazzano, you can use this field to
define an OpenSearch cluster using node groups.

To support backward compatibility, Helm overrides can be configured using [spec.components.elasticsearch.installArgs](/docs/reference/api/verrazzano/verrazzano/#opensearch-install-args)),
though it is recommended to configure your cluster using `nodes` instead.

The following example overrides the `dev` installation profile, OpenSearch configuration (a single-node cluster with
1Gi of memory and ephemeral storage) to use a multi-node cluster (three master nodes, and three combination data/ingest nodes) with persistent storage.
Note that the public API references Elasticsearch, the API will change to OpenSearch in an upcoming release.

```yaml
apiVersion: install.verrazzano.io/v1alpha1
kind: Verrazzano
metadata:
  name: custom-opensearch-example
spec:
  profile: dev
  components:
    elasticsearch:
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
            
      # Override the default cluster settings, since we are providing our own topology.  
      installArgs:
      - name: nodes.master.replicas
        value: "0"
      - name: nodes.ingest.replicas
        value: "0"
      - name: nodes.data.replicas
        value: "0"
```

Listing the pods and persistent volumes in the `verrazzano-system` namespace for the previous configuration
shows the expected nodes are running with the appropriate data volumes:

```
$ kubectl get pvc,pod -l verrazzano-component=opensearch -n verrazzano-system

# Sample output
NAME                                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/elasticsearch-master-vmi-system-master-0   Bound    pvc-9ace042a-dd68-4975-816d-f2ca0dc4d9d8   50Gi       RWO            standard       5m22s
persistentvolumeclaim/elasticsearch-master-vmi-system-master-1   Bound    pvc-8bf68c2c-235e-4bd5-8741-5a5cd3453934   50Gi       RWO            standard       5m21s
persistentvolumeclaim/elasticsearch-master-vmi-system-master-2   Bound    pvc-da8a48b1-5762-4669-98f0-8479f30043fc   50Gi       RWO            standard       5m21s
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
pod/vmi-system-kibana-7d47f65dfc-zhjxp                 2/2     Running    0          5m21s
pod/vmi-system-master-0                                2/2     Running    0          5m21s
pod/vmi-system-master-1                                2/2     Running    0          5m21s
pod/vmi-system-master-2                                2/2     Running    0          5m21s
pod/vmi-system-prometheus-0-5fd9d66b4c-x57sv           3/3     Running    0          5m21s
pod/weblogic-operator-666b548749-lj66t                 2/2     Running    0          7m48s
```

Running the command `kubectl describe pod -n verrazzano-system vmi-system-data-ingest-0-5485dcd95d-rkhvk` shows the
requested amount of memory:

```
Containers:
  es-data:
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
- Is three or more days old.
- Contains 1,000 documents or more.
- Is 10GB in size or larger.

```yaml
apiVersion: install.verrazzano.io/v1alpha1
kind: Verrazzano
metadata:
  name: custom-opensearch-example
spec:
  profile: dev
  components:
    elasticsearch:
      policies:
        - policyName: my-app
          indexPattern: my-app-*
          minIndexAge: 14d
          rollover:
            minIndexAge: 3d
            minDocCount: 1000
            minSize: 10Gb
```