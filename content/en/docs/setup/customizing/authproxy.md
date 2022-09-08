---
title: Customize AuthProxy
description: Customize Verrazzano AuthProxy installation settings
linkTitle: AuthProxy
Weight: 5
draft: false
---

The Verrazzano AuthProxy component enables authentication and authorization for Keycloak users accessing Verrazzano resources.  You can customize the AuthProxy component using settings in the Verrazzano custom resource.

The following table describes the fields in the Verrazzano custom resource pertaining to the [AuthProxy component]({{< relref "/docs/reference/API/Verrazzano/v1beta1.md#authproxy-component" >}}).

| Path to Field | Description |
| --- | --- |
| `spec.components.authProxy.kubernetes.replicas`    | The number of pods to replicate.  The default is `2` for the `prod` profile and `1` for all other profiles. |
| `spec.components.authProxy.kubernetes.affinity`    | The pod affinity definition expressed as a standard Kubernetes [affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity) definition.  The default configuration spreads the AuthProxy pods across the available nodes. <pre>spec:<br>  components:<br>    authProxy:<br>      kubernetes:<br>        affinity:<br>          podAntiAffinity:<br>            preferredDuringSchedulingIgnoredDuringExecution:<br>              - weight: 100<br>                podAffinityTerm:<br>                  labelSelector:<br>                    matchExpressions:<br>                      - key: app<br>                        operator: In<br>                        values:<br>                          - verrazzano-authproxy<br>                  topologyKey: kubernetes.io/hostname</pre> |

The following example customizes a Verrazzano `prod` profile as follows:
* Increases the replicas count to `3`
* Changes the `podAffinity` configuration to use `requiredDuringSchedulingIgnoredDuringExecution`
```
apiVersion: install.verrazzano.io/v1beta1
kind: Verrazzano
metadata:
  name: example-verrazzano
spec:
  profile: prod
  components:
    authProxy:
      overrides:
      - values:
          replicas: 3
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchExpressions:
                      - key: app
                        operator: In
                        values:
                          - verrazzano-authproxy
                  topologyKey: kubernetes.io/hostname
```