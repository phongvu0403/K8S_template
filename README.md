**Kubernetes Template**

The template has 3 additional components installed than default:


*   Dashboard
*   Ingress
*   CEPH persistent volume

# **Dashboard**


1.**Deploy the latest Kubernetes dashboard**


The first thing to know about the web UI is that it can only be accessed using localhost address on the machine it runs on. This means we need to have an SSH tunnel to the server. For most OS, you can create an SSH tunnel using this command. Replace the `<user>` and `<master_public_IP>` with the relevant details to your Kubernetes cluster.

`ssh -L localhost:8001:127.0.0.1:8001 <user>@<master_public_IP>`

After you’ve logged in, you can deploy the dashboard itself with the following single command.

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`

2.**Creating Admin user**

The Kubernetes dashboard supports a few ways to manage access control. In this example, we’ll be creating an admin user account with full privileges to modify the cluster and using tokens.

Start by making a new directory for the dashboard configuration files.

`mkdir ~/dashboard && cd ~/dashboard`

Create the following configuration and save it as dashboard-admin.yaml file. Note that indentation matters in the YAML files which should use two spaces in a regular text editor.

`nano dashboard-admin.yaml`

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Once set, save the file and exit the editor.

Then deploy the admin user role with the next command.

`kubectl apply -f dashboard-admin.yaml`

Get the admin token using the command below.

`kubectl get secret -n kubernetes-dashboard $(kubectl get serviceaccount admin-user -n kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode`

The token is created each time the dashboard is deployed and is required to log into the dashboard. Note that the token will change if the dashboard is stopped and redeployed.



3.**Creating Read-Only user**

If you wish to provide access to your Kubernetes dashboard, for example, for demonstrative purposes, you can create a read-only view for the cluster.

Similarly to the admin account, save the following configuration in `dashboard-read-only.yaml`

`nano dashboard-read-only.yaml`

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: read-only-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
  name: read-only-clusterrole
  namespace: default
rules:
- apiGroups:
  - ""
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only-binding
roleRef:
  kind: ClusterRole
  name: read-only-clusterrole
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: read-only-user
  namespace: kubernetes-dashboard

  ```
Once set, save the file and exit the editor.

Then deploy the read-only user account with the command below.

`kubectl apply -f dashboard-read-only.yaml`

To allow users to log in via the read-only account, you’ll need to provide a token which can be fetched using the next command.

`kubectl get secret -n kubernetes-dashboard $(kubectl get serviceaccount read-only-user -n kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode`

4.**Accessing the dashboard**

Before we can log in to the dashboard, it needs to be made available by creating a proxy service on the localhost. Run the next command on your Kubernetes cluster.

`kubectl proxy`

Now, assuming that we have already established an SSH tunnel binding to the localhost port 8001 at both ends, open a browser to the link below.

`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

5.**Stopping the dashboard**

User roles that are no longer needed can be removed using the delete method.

```
kubectl delete -f dashboard-admin.yaml
kubectl delete -f dashboard-read-only.yaml
```
Likewise, if you want to disable the dashboard, it can be deleted just like any other deployment.

`kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`







# Ingress

There are many ingress controller. Here, I choose NGINX Ingress Controller.

Firsly, we clone source code of NGINX Ingress Controller follow this command:
` git clone https://github.com/nginxinc/kubernetes-ingress/`

`cd kubernetes-ingress/deployments`

1.***Create a Namespace, a SA, the Default Secret, the Customization Config Map, and Custom Resource Definitions***

Create a namespace and a service account for the Ingress controller:

`kubectl apply -f common/ns-and-sa.yaml`

Create a secret with a TLS certificate and a key for the default server in NGINX:

`kubectl apply -f common/default-server-secret.yaml`

Create a config map for customizing NGINX configuration

`kubectl apply -f common/nginx-config.yaml`

Create a ingress class

`kubectl apply -f common/ingress-class.yaml`


2.***Configure RBAC***

If RBAC is enabled in your cluster, create a cluster role and bind it to the service account, created in Step 1:

`kubectl apply -f rbac/rbac.yaml`

3.***Deploy the Ingress Controller***

We include two options for deploying the Ingress controller:


*   *Deployment*: Use a Deployment if you plan to dynamically change the number of Ingress controller replicas.
*   *DaemonSet*: Use a DaemonSet for deploying the Ingress controller on every node or a subset of nodes.

**Create a DaemonSet**:

For NGINX, run:

`kubectl apply -f daemon-set/nginx-ingress.yaml`

**Create a Deployment**

For NGINX, run:

`kubectl apply -f deployment/nginx-ingress.yaml`

4.**Get Access to the Ingress Controller**

If you created a *daemonset*, ports 80 and 443 of the Ingress controller container are mapped to the same ports of the node where the container is running. To access the Ingress controller, use those ports and an IP address of any node of the cluster where the Ingress controller is running.

If you created a *deployment*, below are two options for accessing the Ingress controller pods.

4.1 *Service with the Type NodePort*

Create a service with the type NodePort:

`kubectl create -f service/nodeport.yaml`

Kubernetes will randomly allocate two ports on every node of the cluster. To access the Ingress controller, use an IP address of any node of the cluster along with the two allocated ports

4.2 *Service with the Type LoadBalancer*

Create a service with the type LoadBalancer. Kubernetes will allocate and configure a cloud load balancer for load balancing the Ingress controller pods.


**Uninstall the Ingress Controller**

Delete the nginx-ingress namespace to uninstall the Ingress controller along with all the auxiliary resources that were created:

`kubectl delete namespace nginx-ingress`

**Note**: If RBAC is enabled on your cluster and you completed step 2, you will need to remove the ClusterRole and ClusterRoleBinding created in that step:

```
 kubectl delete clusterrole nginx-ingress
 kubectl delete clusterrolebinding nginx-ingress
```



# **CEPH persistent volume**

**Rook**

we will be using Rook as our storage orchestrator.
Clone source code follow this command:

`git clone --single-branch --branch release-1.3 https://github.com/rook/rook.git`

After cloning the repo, navigate to the right folder with:

`cd rook/cluster/examples/kubernetes/ceph`

First, we got to create the the RoleBindings. Run the command:

`kubectl create -f common.yaml`

Deploy the Rook operator:

`kubectl create -f operator.yaml`

Create data host part default of rook ceph:

```
cd /var/lib
mkdir rook
```


 Deploy ceph cluster:

`kubectl create -f cluster.yaml`

**Ceph Persistent Volume for Kubernetes**
Block storage allows a single pod to mount storage. In this section, you will create a storage block that you can use later in your applications.

Before Ceph can provide storage to your cluster, you first need to create a storageclass and a cephblockpool. This will allow Kubernetes to interoperate with Rook when creating persistent volumes:

`kubectl apply -f ./csi/rbd/storageclass.yaml`

After successfully deploying the storageclass and cephblockpool, you will continue by defining the PersistentVolumeClaim (PVC) for your application. A PersistentVolumeClaim is a resource used to request storage from your cluster.

For that, you first need to create a YAML file:

`nano pvc-rook-ceph-block.yaml`

Add the following for your PersistentVolumeClaim:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Now that you have defined the PersistentVolumeClaim, it is time to deploy it using the following command:

`kubectl apply -f pvc-rook-ceph-block.yaml`


