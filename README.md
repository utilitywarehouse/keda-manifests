# keda-manifests
Holds manifests for Keda adapted for UW usage.
Based on the manifests from [Keda releases](https://github.com/kedacore/keda/releases)

## Structure
The high level structure is as follows:
- `/cluster` folder - contains cluster level resources that can only be applied by cluster admins. They must be reviewed by the system team.
- `/namespaced` folder - contains the resources that can be deployed in regular namespaces and can be applied by namespace admins.

## Components
Considering our Kubernetes cluster setup we'll deploy:
- the metrics server can only be one per cluster, as it needs to register as an
  [APIService](https://github.com/utilitywarehouse/kubernetes-manifests/blob/master/dev-merit/kube-system/keda/metrics-api-service.yaml)
  that handles external metrics. Until wider adoption, this sits in a normal namespace like billing.
- until wider adoption the operator will be deployed in each namespace that wants to leverage Keda.

### RBAC split
At the moment of writing this doc, [Keda releases](https://github.com/kedacore/keda/releases) come with a single global role and a single service account that is used both by the metrics server and by the operator and the permissions are quite loose.

For adapting the RBAC to our clusters, we split it into several pieces:
- the metrics api server will run under the `service account` [keda-metrics-apiserver](https://github.com/utilitywarehouse/keda-manifests/blob/main/namespaced/metrics-apiserver/rbac.yaml#L2)
- the operator in each namespace will run under the `service account` [keda-operator](https://github.com/utilitywarehouse/keda-manifests/blob/main/namespaced/operator/rbac.yaml#L2) 
- the `cluster role` [keda-operator](https://github.com/utilitywarehouse/keda-manifests/blob/main/cluster/operator-role.yaml#L2) should be bound to each `keda-operator` service account for accessing cluster wide resources.  
- the `role` [keda-operator](https://github.com/utilitywarehouse/keda-manifests/blob/main/namespaced/operator/rbac.yaml#L10) is [bound](https://github.com/utilitywarehouse/keda-manifests/blob/main/namespaced/operator/rbac.yaml#L92) to each `keda-operator` service account for accessing namespaced resources.  
- the `cluster role` [keda-metrics-apiserver](https://github.com/utilitywarehouse/keda-manifests/blob/main/cluster/metrics-apiserver-role.yaml#L2) is bound to the `keda-metrics-apiserver` service account for accessing cluster wide keda objects.
- the `cluster role` [keda-external-metrics-reader](https://github.com/utilitywarehouse/keda-manifests/blob/main/cluster/hpa-rbac.yaml#L2) bound to the existing `horizontal-pod-autoscaler` service account for HPA to be able to access external metrics exposed by the keda metrics-apiserver. This is kept as is from the released keda manifests 

### Secrets limitation
Using scalers with any means of [authentication through secrets](https://keda.sh/docs/2.7/concepts/authentication/) is `NOT SUPPORTED`. 

#### Explanation
In its default RBAC configuration, Keda gives `cluster wide access` for its components to `secrets`. 

This is not acceptable to our cluster setup as it would break our namespace isolation approach, and it would pose a serious security risk.

In version 1.7.1, we tried limiting the secrets access to only namespace through a role, but the `keda metrics-apiserver` wants cluster wide access, so this strategy failed.

#### Workaround
So far we've been using the kafka & prometheus scalers which don't require authentication.

If you do need a scaler with authentication here are the proposed solutions:
1. Use a prometheus exporter that exposes the needed metric as a prometheus metric and handles the authentication. After this, just use the prometheus scaler.
2. Use credentials from HashiCorp Vault through [TriggerAuthentication](https://keda.sh/docs/2.7/concepts/authentication/#re-use-credentials-and-delegate-auth-with-triggerauthentication). This requires more setup as we didn't try it yet.

## Upgrade procedure
Due to the RBAC split, the upgrade is not straight forward.

We used the following procedure:
1. diff the Keda manifests between [Keda releases](https://github.com/kedacore/keda/releases). Example: keda [v5.1.0](https://github.com/kedacore/keda/releases/download/v2.5.0/keda-2.5.0.yaml) vs [v.2.7.1](https://github.com/kedacore/keda/releases/download/v2.7.1/keda-2.7.1.yaml)
2. adjust the manifests in this repo, based on the diffs.
3. test the upgrade in the exp1-merit environment where there are 2 namespaces using it: `billing-keda` and `finance-keda`. 

Things to check:
- The metrics-apiserver & operators are running fine. Check the logs for this.
- The Horizontal Pod Autoscalers created by Keda can read the external metrics and scale the deployments based on them.