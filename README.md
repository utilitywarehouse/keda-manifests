# keda-manifests
Holds manifests for Keda adapted for UW usage.
Based on the manifests from [Keda releases](https://github.com/kedacore/keda/releases)

## Structure
The high level structure is as follows:
- `/kube-system` - contains cluster level resources that can only be applied by cluster admins in the kube-system namespace. They must be reviewed by the system team.
- `/dev-enablement` - contains the resources that can be applied by namespace admins. The default namespace where keda runs is "dev-enablement"

## Components
Keda runs nowadays as a single operator instance + single metrics API server per cluster, that watch over other selected namespaces.
Both components will run from the `dev-enablement` namespace

### RBAC adjustments 
[Keda releases](https://github.com/kedacore/keda/releases) include:
- a `keda-operator` [cluster role](kube-system/rbac/operator-cluster-role.yaml) with cluster wide permissions used for watching over other namespaces. 
  The permissions in it are quite well curated nowadays, but we went forward and adjusted it more. These adjustments are documented at the beginning of the file.  
- a slim `keda-operator` [role](dev-enablement/rbac/operator-role.yaml) used for accessing local `dev-enablement` resources. We adjusted this one too documented also at the beginning of the file.
- a `keda-operator` service account that is bound to these roles
Both by the metrics API server and the operator run with this service account.

Although the metrics API doesn't need all the permissions in these roles, we want to keep the manifests as close as possible to Keda upstream, to have a leaner upgrade experience.

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
