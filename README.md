# keda-manifests
Holds manifests for Keda adapted for UW usage.
Based on the manifests from [Keda releases](https://github.com/kedacore/keda/releases)

## Structure
The high level structure is as follows:
- /cluster folder - contains cluster level resources that can only be applied by cluster admins. They must be reviewed by the system team.
- /namespaced folder - contains the resources that can be deployed in regular namespaces and can be applied by namespace admins.

