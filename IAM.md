# Identity and access management with AKS
There are two levels of access needed to fully operate an AKS cluster:
- Access to the AKS as a resource in the Azure subscription which allows to: 
  - control scaling or upgrading the cluster using the AKS APIs,
  - pull the `kubeconfig` file,
- Access to the Kubernetes API controlled either by:
  - Kubernetes RBAC,
  - Azure RBAC integrated with AKS cluster.

Please note this distinction while using instructions below. 

Interacting with Kubernetes (and thus also with an AKS) clusters is provided by `kubectl` - primary interface through which users manage, deploy, and monitor applications and resources. There are a few methods of authenticating and authorization in order to use `kubectl` for the specific cluster, and in case of AKS, these methods are extended with the use of Microsoft Azure authentication solutions. 

### Authentication and authorization for clusters not integrated with AAD
*kubeconfig* is a configuration file utilized by `kubectl` to authenticate and access Kubernetes clusters. It contains the necessary information and credentials required to connect to the cluster's API server and perform operations. Fetching the *kubeconfig* file for the specified AKS cluster and merging it with the local `kubectl` configuration file (~/.kube/config by default) is allowed only for users using the identity with appropriate Azure RBAC permissions over the AKS resource. The roles allowed to access *kubeconfig* file are:

- Azure Kubernetes Service Cluster User Role,
- Azure Kubernetes Service Cluster Admin Role,
- Owner/contributor on the scope, where AKS is provisioned.

> Please note AKS Cluster User role is the minimum role required for pulling *kubeconfig* file, however it grants read-only access to the Kubernetes cluster as an Azure resource. Users with this role can view resources in the cluster but do not have permission to modify or create resources, while AKS Cluster Admin role grants full administrative access over the Kubernetes cluster.

1. Sign in to Azure CLI with using identity with required permissions
	```
	az login
	```
2. Get details about cluster you would like to manage and check its name and resource group
	```
	az aks list
	``` 
3. Get the *kubeconfig* definition for your AKS cluster 
	```
	az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
	```
4. Credentials are merged into your local .kube/config file so that `kubectl` can use them. Now you can use `kubectl` with your cluster
	```
	kubectl get nodes
	```
The example below shows connection with *AKStestCluster* located in *AKStestRG* resource group with checking Azure RBAC role at first:	![kubeconfig merged and tested](https://aksprojectscreens.blob.core.windows.net/screens/3.1.jpg)

The example below shows the error occured when trying to get *kubeconfig* file using identity with insufficient permissions:
![permissions error](https://aksprojectscreens.blob.core.windows.net/screens/4.png)

If your identity has Azure Kubernetes Service Cluster Admin role assigned, you can also fetch the *kubeconfig* file with cluster admin permissions:
	```
	az aks get-credentials --resource-group myResourceGroup --name myAKSCluster --admin
	```

Use the `cat` command with *config* file to check your cluster role and certificate which is the foundation for your authentication in cluster API. 
![enter image description here](https://aksprojectscreens.blob.core.windows.net/screens/5.jpg)
> On clusters that don't use Azure AD, the *clusterUser* role has same effect of *clusterAdmin* role in terms of permissions over the Kubernetes API. 

Azure RBAC roles necessary for fetching *kubeconfig* file can be assigned as any other roles, e.g. via [Azure Portal](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal) or using [Azure CLI](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-cli).

Without any integration with external identity providers like Azure AD, authentication and authorization is provided by Kubernetes cluster itself. Here are the main concepts of Kubernetes authentication and authorization:
 - Kubernetes does not operate classic concept of user; instead, it only trusts certificates as a source of identity and foundation for authentication,
 - Managing identity is based on creation and signing user certificates, embedded in *kubeconfig* file, shared with the end user so that he can authenticate in the cluster,
- *Roles* and *RoleBindings* are defined within the Kubernetes cluster to grant specific permissions to users or service accounts.

In order to manage identities and permissions for other users, you should create, sign and distribute certificates for them. You can find more information in the links below:\
[Kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)\
[Kubernetes RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

### Clusters with Azure AD authentication and Kubernetes RBAC
Azure Kubernetes Service (AKS) can be configured to use Azure Active Directory (Azure AD) for user authentication. In this configuration, you sign in to an AKS cluster using an Azure AD authentication token. Once authenticated, you can use the built-in Kubernetes RBAC to manage access to namespaces and cluster resources based on a user's identity or group membership.

1. Enable Azure AD authentication in Azure Portal, in the *Cluster Configuration* blade:
	![enter image description here](https://aksprojectscreens.blob.core.windows.net/screens/6.jpg)

> Please note that in order to change authentication and authorization method for your AKS cluster you need a permission over the cluster as an Azure resource, e.g. *Azure Kubernetes Service Cluster Admin* role. Screen below shows the example of insufficient permissions error:\
> <img src="https://aksprojectscreens.blob.core.windows.net/screens/7.jpg" width="400">

2. Define the Azure AD group with administrator permissions inside of AKS cluster in order to enable this group members to do administrative tasks, e.g. granting roles and permissions to other users inside of the cluster: 
<img src="https://aksprojectscreens.blob.core.windows.net/screens/8a.jpg" width="700">

3. Now you can see that AKS uses AAD to authenticate user to the cluster:
<img src="https://aksprojectscreens.blob.core.windows.net/screens/9a.jpg" width="700">
Installing *kubelogin* may be necessary for this authentication mechanism to run
	> Regular users, out of administrative AAD group, even having contributor permissions over the AKS resource, cannot perform any actions inside the AKS cluster, because authorization is based on Kubernetes RBAC, not the AAD. 
	<img src="https://aksprojectscreens.blob.core.windows.net/screens/10.jpg" width="700">

4. As a member of *AKSadmins* group you can execute all the operational tasks in created AKS cluster, including creating and assigning roles to the users. Kubernetes RBAC supports lots of built-in roles which you can assign to AAD users. The yaml file below shows the script necessary to assign *cluster-admin* role to *bartlomiej.dylik[...]* user:
	```yaml
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	name: cluster-admin-binding
	subjects:
	- kind: User
	name: bartlomiej.dylik_pwc.com#EXT#@bartekdylikgmail.onmicrosoft.com
	apiGroup: rbac.authorization.k8s.io
	roleRef:
	kind: ClusterRole
	name: cluster-admin
	apiGroup: rbac.authorization.k8s.io
	```
5. Use the command below to apply the yaml file and assign the role (*RoleBinding* in Kubernetes terminology):
    ```
    kubectl apply -f cluster-admin-binding.yaml
    ```
    Now you can check if role is assigned and make sure user *bartlomiej.dylik[...]* has *cluster-admin* permissions:
    <img src="https://aksprojectscreens.blob.core.windows.net/screens/11.png" width="700">
6. In the next steps you can create new [*Roles*](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole) and [*RoleBindings*](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding) to other Azure AD users. 

> Please note the difference between Azure RBAC roles over the AKS cluster as an Azure resource in the subscription and Kubernetes roles existed inside the AKS cluster. For example, you can assign Azure RBAC *Contributor* role on the cluster and the recipient will be able to read and edit AKS as a resource, however he will not be allowed to interact with it using its Kubernetes management API.
    
### Clusters with Azure AD authentication and Azure RBAC
Azure Kubernetes Service (AKS) can be configured to use Azure Active Directory (Azure AD) not only for user authentication, but also for authorization in the cluster. In this case Azure AD is a single source for user account management, credentials and permissions. This is the most convenient option, as it does not require managing two separate identity management platforms. 

1. Enable Azure AD authentication and authorization in Azure Portal, in the *Cluster Configuration* blade:
<img src="https://aksprojectscreens.blob.core.windows.net/screens/12.jpg" width="700">

2. Make sure your role allows to make role assignments and assign proper roles to the users. AKS provides the following built-in roles, assigned as any other RBAC Azure Roles:
<img src="https://aksprojectscreens.blob.core.windows.net/screens/13.jpg" width="600">

3. The example below shows the `get nodes` command executed before and after *Azure Kubernetes Service RBAC Cluster Admin* role assignment:
<img src="https://aksprojectscreens.blob.core.windows.net/screens/14b.jpg" width="700">

### Summary
The examples below show three different methods for authentication and authorization in AKS cluster:
 - Local accounts with Kubernetes RBAC,
 - Azure AD authentication with Kubernetes RBAC,
 - Azure AD authentication with Azure RBAC. 
 
 Choosing the best option depends on the required level of permissions granularity and overall IAM strategy adopted. Please follow [best practises](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-identity) recommended by Microsoft while implementing authentication and authorization method in your AKS cluster.
 
 Useful links:

[Access and identity options for Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/concepts-identity)\
[Use Azure role-based access control to define access to the Kubernetes configuration file in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/control-kubeconfig-access)\
[Use Kubernetes role-based access control with Azure Active Directory in Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/azure-ad-rbac?tabs=portal)\
[Use Azure role-based access control for Kubernetes Authorization](https://learn.microsoft.com/en-us/azure/aks/manage-azure-rbac)\
[Kubernetes: authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)\
[Kubernetes: authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)