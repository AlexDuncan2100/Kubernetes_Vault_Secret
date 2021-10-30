# Prerequisites:

For the example configuration, you will be using minikube. You can follow the Get Started docs to install minikube or just make sure you have the required prerequisites:

# Minikube Installation On EC2:

Let’s start with launching EC2 instance and doing SSH into it, I’m starting a micro instance having minimum of two 2 cores of CPU of ubuntu 18+ LTS version for this demo.

# Step 1. Install Kubectl command line client for minikube, which is used to interact with kubernete’s cluster:
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

# Step 2. change permission of the binaries, and move to user folder:

chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

# Step 3. Install docker, there are many ways to do it. We will use yum package.

sudo yum update -y
sudo yum install docker -y

# Step 4. Install minikube, one can combine the commands into one command with &&:

$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

$ chmod +x minikube

$ sudo mv minikube /usr/local/bin/

$ sudo yum install -y conntrack

# Step 5. Become root, for missions.
sudo groupadd docker
sudo usermod -aG docker ec2-user
newgrp docker

# Step 6. Start minikube, with no vm args

minikube start --vm-driver=none

# Steps 7. Check status.

minikube status

# Installlation of Helm in Linux machine:

curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

helm version --short

helm repo add stable https://charts.helm.sh/stable

helm search repo stable

helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)


# Run Vault in Kubernetes:

Getting Vault up and running in Kubernetes is extremely easy when using the provided Helm chart. This is our recommended way to install and configure Vault in Kubernetes.

You will start by adding the Vault Helm chart repository and running the Helm chart. You can enable the CSI provider via the Helm chart with the added csi.enabled=true parameter. If you prefer to install the Vault CSI provider via the manifests, a section below has details.

$ kubectl create ns vault

$ helm repo add hashicorp https://helm.releases.hashicorp.com

$ helm repo update # You may need to update

$ helm install vault hashicorp/vault --namespace=vault \
 --set "server.dev.enabled=true" \
 --set "injector.enabled=false" \
 --set "csi.enabled=true"

# Make sure the Vault container is running and reports “Ready 1/1":

$ kubectl get pods --namespace=vault

NAME               READY  STATUS   RESTARTS  AGE
vault-0            1/1    Running  0         35m
vault-csi-provider 1/1    Running  0         35m # If installed via Helm


# Configure Vault:
Now that you have your Vault cluster pod running, you are going to add a test secret that you will retrieve later. The following Vault commands will be run from the container's shell:

$ kubectl exec -it vault-0 --namespace=vault -- /bin/sh

$ vault kv put secret/db-pass password="db-secret-password"

Key              Value
---              -----
created_time     2021-03-29T21:06:29.46397907Z
deletion_time    n/a
destroyed        false
version          1

# Test that you can retrieve the KV pair you just created:

$ vault kv get secret/db-pass

====== Metadata ======
Key              Value
---              -----
created_time     2021-03-29T21:06:29.46397907Z
deletion_time    n/a
destroyed        false
version          1
====== Data ======
Key         Value
---         -----
password    db-secret-password

# Now you need to enable the authentication method that will be used to verify the identity of the service. This is done by using the Kubernetes auth method:

$ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/

$ vault write auth/kubernetes/config \
issuer="https://kubernetes.default.svc.cluster.local" \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

Success! Data written to: auth/kubernetes/config

# Next, you create the Vault policy that you will use to allow your Kubernetes services access to your created secrets paths. This would be the KV pair you created a few steps back:

$ vault policy write csi - <<EOF
path "secret/data/db-pass" {
 capabilities = ["read"]
}
EOF

# The last step for the Vault configuration is to create the authentication role. This defines the authentication endpoint path, the Kubernetes service account, the Kubernetes namespace, the access policy you created, and a Vault token TTL (time-to-live):

$ vault write auth/kubernetes/role/csi \
   bound_service_account_names=myapp1-sa \
   bound_service_account_namespaces=myapp1 \
   policies=csi \
   ttl=20m

Success! Data written to: auth/kubernetes/role/database

exit

# Install the Secrets Store CSI Driver in Kubernetes:

There are two ways to set up the Secrets Store CSI driver in Kubernetes. The documentation goes into detail on both the Helm install and using the deploy manifests. This example uses the provided Helm chart and you define the namespace.

$ helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
$ helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace vault

$ kubectl describe ds csi-secrets-store --namespace=vault|grep csi-secrets-store/driver
   Image:      k8s.gcr.io/csi-secrets-store/driver:v0.0.21

# Now that you have the CSI driver setup in your Kubernetes cluster, you have a new custom resource for defining the SecretProviderClass. You can see the new custom resources by listing the custom resource definitions:

$ kubectl get crd
NAME                                                        CREATED AT
secretproviderclasses.secrets-store.csi.x-k8s.io            2021-03-29T22:29:24Z
secretproviderclasspodstatuses.secrets-store.csi.x-k8s.io   2021-03-29T22:29:24Z

# You should also have a DaemonSet and a three-container pod running:

$ kubectl get ds --namespace=vault
NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
csi-secrets-store-csi-driver   1         1         1       1            1           kubernetes.io/os=linux   57s
vault-csi-provider             1         1         1       1            1           <none>                   4m44s

$ kubectl get pods --namespace=vault

NAME                                 READY   STATUS    RESTARTS   AGE
csi-secrets-store-csi-driver-b7rjj   3/3     Running   0          81s
vault-0                              1/1     Running   0          5m8s
vault-csi-provider-bcj4q             1/1     Running   0          5m8s

# Application Namespace Setup:

The last thing to do is set up your application namespaces with a ServiceAccount and SecretProviderClass. Each unique Kubernetes namespace will require both. This will allow segmentation of sensitive Vault data between your Kubernetes namespaces via Vault roles and policies. You will define Vault as your provider and point to the Vault pod that you previously set up. You will also define your secret that you created earlier.

# Create the application namespace:
$ kubectl create ns myapp1
namespace/myapp1 created

$ cat <<EOF | kubectl apply --namespace=myapp1 -f -
apiVersion: v1
kind: ServiceAccount
metadata:
 name: myapp1-sa
 namespace: myapp1
EOF

serviceaccount/myapp1-sa created

# Now you will create the SecretProviderClass. The vault-database metadata name is how you will reference this resource in your pod spec in a later step. You can see your provider is set to Vault, your role name matches the role you created previously, and the Vault address is defined.

In this example, you are not using Vault namespaces or TLS. I included these additional parameters in this example below but commented them out for reference. Since you are running Vault in dev mode, TLS is not enabled. This is another reason I wouldn’t recommend dev mode for production.

The array of objects will be the secret paths you want to mount in your pod.

*objectName: The secret will be written to a filename in your pod with this alias.
*secretPath: The path in Vault where the secret should be retrieved.
*secretKey: Optional key to read from .Data. The entire JSON payload will be retrieved if this is omitted.


$ cat <<EOF | kubectl apply --namespace=myapp1 -f -
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
 name: vault-database
spec:
 provider: vault
 parameters:
   roleName: "csi"
   vaultAddress: "http://vault.vault:8200"
   # vaultNamespace: <name of Vault Namespace>
   # vaultCACertPath: <path to CA file for validation>
   # vaultTLSClientCertPath: <path to client cert>
   # vaultTLSClientKeyPath: <path to client key>
   objects: |
     - objectName: "password"
       secretPath: "secret/data/db-pass"
       secretKey: "password"
EOF

# Testing:

# Finally, you can define a basic NGINX pod and have it use the CSI driver and the SecretProviderClass you created. Then you just need to mount the volume at a defined path:

$ cat <<EOF | kubectl apply --namespace=myapp1 -f -
kind: Pod
apiVersion: v1
metadata:
 name: nginx-secrets-store-inline
spec:
 containers:
 - image: nginx
   name: nginx
   volumeMounts:
   - name: secrets-store-inline
     mountPath: "/mnt/secrets-store"
     readOnly: true
 serviceAccountName: myapp1-sa
 volumes:
   - name: secrets-store-inline
     csi:
       driver: secrets-store.csi.k8s.io
       readOnly: true
       volumeAttributes:
         secretProviderClass: "vault-database"
EOF

# You will want to make sure your NGINX pod is running. If it is not, this is a quick sign that something didn’t work. If you don’t already have the NGINX image downloaded, it will take a few minutes longer.

$ kubectl get pods -n myapp1
NAME                         READY   STATUS    RESTARTS   AGE
nginx-secrets-store-inline   1/1     Running   0          22m


$ kubectl describe pod -n myapp1 nginx-secrets-store-inline
...
Events:
Type    Reason     Age   From               Message
----    ------     ----  ----               -------
Normal  Scheduled  34m   default-scheduler  Successfully assigned myapp1/nginx-secrets-store-inline to minikube
Normal  Pulling    33m   kubelet            Pulling image "nginx"
Normal  Pulled     33m   kubelet            Successfully pulled image "nginx" in 6.469146131s
Normal  Created    33m   kubelet            Created container nginx
Normal  Started    33m   kubelet            Started container nginx

# Now that the test pod is up and running, you can read the contents of your mounted volume:


$ kubectl exec -it -n myapp1 nginx-secrets-store-inline -- bash

cat /mnt/secrets-store/password

db-secret-password


# Conclusion and Looking Ahead:
We are excited to offer the CSI method alongside our sidecar injection option to help secure your Kubernetes application secrets with Vault. In this post, you learned how both options use different approaches to accomplish this goal.

The sidecar injection method requires additional sidecar containers and is set up via pod annotations. With this method the secret stays within the context of the pod. The CSI method does not require sidecar containers and is set up via a CRD. This method uses Kubernetes hostPath volumes, which puts the secret into the context of the node. Understanding these differences and how they impact your architecture can help you choose the best approach to fit both your security and automation requirements.




