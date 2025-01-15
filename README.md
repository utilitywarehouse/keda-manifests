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

### TLS 
Keda uses TLS certificates, and we're managing those TLS assets on our own using cert-manager.

Here are [their docs on this](https://keda.sh/docs/2.16/operate/security/#use-your-own-tls-certificates).

Implementation details:
- [defined custom cert manager issuer](kube-system/clusterissuer/clusterissuer.yaml). This is the issuer used for all the Keda self-signed certificates.
- [defined certificate](dev-enablement/metrics-apiserver/certificate.yaml) for metrics-apiserver. The generated secret is [mounted in the pod](https://github.com/utilitywarehouse/keda-manifests/blob/264e3e5d3cf8285f7b2c1e17b1e23141a20c0903/dev-enablement/metrics-apiserver/deployment.yaml#L111-L125). 
- [defined certificate](dev-enablement/operator/certificate.yaml) for the operator. We're running with [disabled certificate rotation](https://github.com/utilitywarehouse/keda-manifests/blob/264e3e5d3cf8285f7b2c1e17b1e23141a20c0903/dev-enablement/operator/deployment.yaml#L58) and the generated secret is [mounted in the pod](https://github.com/utilitywarehouse/keda-manifests/blob/264e3e5d3cf8285f7b2c1e17b1e23141a20c0903/dev-enablement/operator/deployment.yaml#L108-L123)
- [defined certificate](kube-system/apiservice/certificate.yaml) for the API Service. The `cabundle` in the API service is set by using the annotation [cert-manager.io/inject-ca-from](https://github.com/utilitywarehouse/keda-manifests/blob/264e3e5d3cf8285f7b2c1e17b1e23141a20c0903/kube-system/apiservice/apiservice.yaml#L6)

### RBAC adjustments 
[Keda releases](https://github.com/kedacore/keda/releases) include:
- a `keda-operator` [cluster role](kube-system/rbac/operator-cluster-role.yaml) with cluster wide permissions used for watching over other namespaces. 
  The permissions in it are quite well curated nowadays, but we went forward and adjusted it more. These adjustments are documented at the beginning of the file.  
- a slim `keda-operator` [role](dev-enablement/rbac/operator-role.yaml) used for accessing local `dev-enablement` resources. We adjusted this one too documented also at the beginning of the file.
- a `keda-operator` service account that is bound to these roles
Both by the metrics API server and the operator run with this service account.

Although the metrics API doesn't need all the permissions in these roles, we want to keep the manifests as close as possible to Keda upstream, to have a leaner upgrade experience.

### SclaledJobs limitation
Due to [this bug](https://github.com/kedacore/keda/issues/4740) we currently don't support [scaling jobs](https://keda.sh/docs/2.16/concepts/scaling-jobs/).

We took the scaledjobs.keda.sh Custom Resource Definition out of the manifests.

If you have a good use case for it please contact us.

### Secrets limitation
We are running with [restricting Keda access to secrets](https://keda.sh/docs/2.16/operate/cluster/#restrict-secret-access).

This means that using scalers that use any flavor of [authentication through secrets](https://keda.sh/docs/2.16/concepts/authentication/) requires giving Keda access to the secrets in your namespace.


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
