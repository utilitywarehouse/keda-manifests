# keda-manifests
Holds manifests for Keda adapted for UW usage.
Based on the manifests from [Keda releases](https://github.com/kedacore/keda/releases)

## Structure
The high level structure is as follows:
- /cluster folder - contains cluster level resources that can only be applied by cluster admins. They must be reviewed by the system team.
- /namespaced folder - contains the resources that can be deployed in regular namespaces and can be applied by namespace admins.

## Components
Considering our Kubernetes cluster setup we'll deploy:
- the metrics server can only be one per cluster, as it needs to register as an
  [APIService](https://github.com/utilitywarehouse/kubernetes-manifests/blob/master/dev-merit/kube-system/keda/metrics-api-service.yaml)
  that handles external metrics
- until wider adoption the operator will be deployed in each namespace that wants to leverage Keda.
