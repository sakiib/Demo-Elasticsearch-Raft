# AppsCode Demo (28-05-21)
### Manage Elasticsearch credentials using the KubeVault Operator, Vault Database Secret Engine and the Integrated Storage Backend (Raft) to persist Vault's data.

### Install KubeDB Enterprise Edition
* Create kind cluster - 
  ```bash
  $ kind create cluster --image kindest/node:v1.18.15@sha256:5c1b980c4d0e0e8e7eb9f36f7df525d079a96169c8a8f20d8bd108c0d0889cc4
  ```
* Go to the AppsCode License Server & use the cluster ID to get the license of the KubeDB Enterprise Edition & put it in `license.txt`. 
* Get the Cluster ID - `$ kubectl get ns kube-system -o=jsonpath='{.metadata.uid}'`
* Install KubeDB using Helm3 
  ```bash
  $ helm repo add appscode https://charts.appscode.com/stable/
  $ helm repo update
  $ helm search repo appscode/kubedb
  ```

  ```bash
  $ helm install kubedb appscode/kubedb \
      --version v2021.04.16 \
      --namespace kube-system \
      --set-file global.license=/home/sakib/demo-es-raft/license.txt \
      --set kubedb-enterprise.enabled=true \
      --set kubedb-autoscaler.enabled=true
  ```

### Elasticsearch quickstart using KubeDB
* Create the namespace `demo`
`$ kubectl create namespace demo`
* Find available StorageClass
`$ kubectl get storageclass`
* Find available Elasticsearch version
`$ kubectl get elasticsearchversions`
* Create an Elasticsearch Cluster
`$ kubectl apply -f elasticsearch.yaml`
* Use port forwarding to connect with Elasticsearch database
`$ kubectl port-forward -n demo svc/es-quickstart 9200`
* Connection information
Username: `$ kubectl get secret -n demo es-quickstart-elastic-cred -o jsonpath='{.data.username}' | base64 -d`
Password: `$ kubectl get secret -n demo es-quickstart-elastic-cred -o jsonpath='{.data.password}' | base64 -d`
* Check health of the Elasticsearch database
`$ curl -XGET -k -u '<username>:<password>' "https://localhost:9200/_cluster/health?pretty"`

### Install KubeVault Operator
```bash
$ cd go/src/kubevault.dev/operator/ # (branch - raft)
$ export REGISTRY=sakibalamin
$ make push install
```
### Kubevault
KubeVault operator is a Kubernetes controller for HashiCorp Vault. Vault is a tool for secrets management, encryption as a service, and privileged access management. Deploying, maintaining, and managing Vault in Kubernetes could be challenging. KubeVault operator eases these operational tasks so that developers can focus on solving business problems.

### Deploy VaultServer
* Patch the VaultServer version - `$ kubectl apply -f vsversion.yaml`
* Deploy VaultServer - `$ kubectl apply -f vaultserver.yaml`

A VaultServer is a Kubernetes CustomResourceDefinition (CRD) which is used to deploy a HashiCorp Vault server on Kubernetes clusters in a Kubernetes native way.
When a VaultServer is created, the KubeVault operator will deploy a Vault server and create necessary Kubernetes resources required for the Vault server.

* Get all the Secrets - `$ kubectl get secret -n demo`
* Get the vault keys (vault-root-token, unseal-keys) - `$ kubectl get secret -n demo vault-keys -o yaml`
* Base-64 decode the vault-root-token - `$ echo "cy5yenVIQUNsMDdtUzhzemF3QWNuZ202dlk=" | base64 -d`
* Export as VAULT_TOKEN ENV VAR - `$ export VAULT_TOKEN=s.rzuHACl07mS8szawAcngm6vY`
* Export VAULT_ADDR - `$ export VAULT_ADDR='https://127.0.0.1:8200'`
* Export VAULT_SKIP_VERIFY - `$ export VAULT_SKIP_VERIFY=true`
* Use port forwarding to connect with Vault - `$ kubectl port-forward -n demo svc/vault 8200`
* Check Vault Status - `$ vault status` 
```
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            4
Threshold               2
Version                 1.7.0
Storage Type            raft
Cluster Name            vault-cluster-10222fbc
Cluster ID              031056e3-f096-f1ba-931a-38becc8ec94f
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Active Since            2021-05-22T09:10:51.503565207Z
Raft Committed Index    68
Raft Applied Index      68
```

* Check vault's secrets list - `$ vault secrets list`
```
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_ecff0777    per-token private secret storage
identity/     identity     identity_c76ff30a     identity store
sys/          system       system_7e29f09c       system endpoints used for control, policy and debugging
```

* Check vault's policy list - `$ vault policy list`
```
default
k8s.-.demo.vault-auth-method-controller
vault-policy-controller
root
```

* Create the SecretEngine
`$ kubectl apply -f secretengine.yaml`

**SecretEngine:** A SecretEngine is a Kubernetes CustomResourceDefinition (CRD) which is designed to automate the process of enabling and configuring secret engines in Vault in a Kubernetes native way.

When a SecretEngine CRD is created, the KubeVault operator will perform the following operations:

---
* Creates vault policy for the secret engine. 
* Updates the Kubernetes auth role of the default k8s service account created with VaultServer with a new policy. The new policy will be merged with previous policies.
* Enables the secrets engine at a given path. By default, they are enabled at their “type” (e.g. “aws” is enabled at “aws/").
* Configures the secret engine with the given configuration.
---  

* Check vault's secrets list - `$ vault secrets list`
```
Path                     Type         Accessor              Description
----                     ----         --------              -----------
cubbyhole/               cubbyhole    cubbyhole_ecff0777    per-token private secret storage
custom-database-path/    database     database_7fd68639     n/a
identity/                identity     identity_c76ff30a     identity store
sys/                     system       system_7e29f09c       system endpoints used for control, policy and debugging
```

* Check vault's policy list - `$ vault policy list`
```
default
k8s.-.demo.es-quickstart
k8s.-.demo.vault-auth-method-controller
vault-policy-controller
root
```

* Create the Secret Engine Role
`$ kubectl apply -f secretenginerole.yaml`

**SecretEngineRole:** A ElasticsearchRole is a Kubernetes CustomResourceDefinition (CRD) which allows a user to create a Elasticsearch database secret engine role in a Kubernetes native way. When a ElasticsearchRole is created, the KubeVault operator creates a Vault role according to the specification.

* Create a database access request
`$ kubectl apply -f dbaccessrequest.yaml`

**DatabaseAccessRequest:** A DatabaseAccessRequest is a Kubernetes CustomResourceDefinition (CRD) which allows a user to request a Vault server for database credentials in a Kubernetes native way. If DatabaseAccessRequest is approved, then the KubeVault operator will issue credentials and create Kubernetes secret containing credentials.

* To Approve the request - `$ kubectl vault approve databaseaccessrequest es-cred-rqst -n demo`
* To Deny the request - `$ kubectl vault deny databaseaccessrequest es-cred-rqst -n demo`

* Check the database access request status - `$ kubectl get databaseaccessrequest es-cred-rqst -n demo -o json | jq '.status'`
```
{
  "conditions": [
    {
      "lastTransitionTime": "2021-05-22T09:31:43Z",
      "message": "This was approved by: kubectl vault approve databaseaccessrequest",
      "reason": "KubectlApprove",
      "status": "True",
      "type": "Approved"
    },
    {
      "lastTransitionTime": "2021-05-22T09:31:51Z",
      "message": "The requested credentials successfully issued.",
      "reason": "SuccessfullyIssuedCredential",
      "status": "True",
      "type": "Available"
    }
  ],
  "lease": {
    "duration": "1h0m0s",
    "id": "custom-database-path/creds/k8s.-.demo.es-quickstart/gA8pNpinN0Cu95dMGX4ryCQs",
    "renewable": true
  },
  "observedGeneration": 1,
  "phase": "Approved",
  "secret": {
    "name": "es-cred-rqst-z3pdjr"
  }
}
```

* Get the Secret
`$ kubectl get secret -n demo es-cred-rqst-z3pdjr -o yaml -o json`
```
{
    "apiVersion": "v1",
    "data": {
        "password": "MHYtYW5vTW1KZHgta3g3d3RMYzk=",
        "username": "di1rdWJlcm5ldGVzLWRlbW8tazhzLi0uZGVtby5lcy1xLWM2SE5NZU1seFRaZ0Z3a1pCcHFyLTE2MjE2NzU5MDM="
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2021-05-22T09:31:51Z",
        "managedFields": [
            {
                "apiVersion": "v1",
                "fieldsType": "FieldsV1",
                "fieldsV1": {
                    "f:data": {
                        ".": {},
                        "f:password": {},
                        "f:username": {}
                    },
                    "f:metadata": {
                        "f:ownerReferences": {
                            ".": {},
                            "k:{\"uid\":\"49d61d8a-52df-46a2-9dd2-74343c94f722\"}": {
                                ".": {},
                                "f:apiVersion": {},
                                "f:blockOwnerDeletion": {},
                                "f:controller": {},
                                "f:kind": {},
                                "f:name": {},
                                "f:uid": {}
                            }
                        }
                    },
                    "f:type": {}
                },
                "manager": "vault-operator",
                "operation": "Update",
                "time": "2021-05-22T09:31:51Z"
            }
        ],
        "name": "es-cred-rqst-z3pdjr",
        "namespace": "demo",
        "ownerReferences": [
            {
                "apiVersion": "engine.kubevault.com/v1alpha1",
                "blockOwnerDeletion": true,
                "controller": true,
                "kind": "DatabaseAccessRequest",
                "name": "es-cred-rqst",
                "uid": "49d61d8a-52df-46a2-9dd2-74343c94f722"
            }
        ],
        "resourceVersion": "12296",
        "selfLink": "/api/v1/namespaces/demo/secrets/es-cred-rqst-z3pdjr",
        "uid": "443c41a5-8822-4751-9f06-a06e0008ebe0"
    },
    "type": "Opaque"
}
```

`$ kubectl get secret -n demo es-cred-rqst-z3pdjr -o yaml -o json | jq '.data'`

```
{
  "password": "MHYtYW5vTW1KZHgta3g3d3RMYzk=",
  "username": "di1rdWJlcm5ldGVzLWRlbW8tazhzLi0uZGVtby5lcy1xLWM2SE5NZU1seFRaZ0Z3a1pCcHFyLTE2MjE2NzU5MDM="
}
```

### Connect to Elasticsearch using the username & the password
* set ES_PASSWORD ENV var - `$ export ES_PASSWORD=(kubectl get secret -n demo es-cred-rqst-npjlhs -o=jsonpath={.data.password} | base64 -d)`

* set ES_USER ENV var - `$ export ES_USER=(kubectl get secret -n demo es-cred-rqst-npjlhs -o=jsonpath={.data.username} | base64 -d)`

* check health - `$ curl -XGET -k -u "$ES_USER:$ES_PASSWORD"  "https://localhost:9200/_cluster/health?pretty"`
  
* create ES indices
```bash
$ curl -XPUT -k -u "$ES_USER:$ES_PASSWORD"  "https://localhost:9200/demo_index0"
```

* get ES indices
```bash
$ curl -XGET -k -u "$ES_USER:$ES_PASSWORD"  "https://localhost:9200/_cat/indices"
```

### Check the Raft Storage 

* **Delete** the VaultServer - `$ kubectl delete -f vaultserver.yaml`
* **Check** the PVC - `$ kubectl get pvc -n demo`
* **Create** the VaultServer again - `$ kubectl apply -f vaultserver.yaml`
* **Check** the secrets list (e.g: stop & port-forward again) - `$ vault secrets list` (Our enabled path `custom-database-path`/ still should exists)
  
* `$ kubectl get all -n demo`
```
NAME                  READY   STATUS    RESTARTS   AGE
pod/es-quickstart-0   1/1     Running   0          166m
pod/es-quickstart-1   1/1     Running   0          163m
pod/es-quickstart-2   1/1     Running   0          163m
pod/vault-0           3/3     Running   0          83m

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
service/es-quickstart          ClusterIP   10.102.203.75   <none>        9200/TCP                        166m
service/es-quickstart-master   ClusterIP   None            <none>        9300/TCP                        166m
service/es-quickstart-pods     ClusterIP   None            <none>        9200/TCP                        166m
service/vault                  NodePort    10.106.57.102   <none>        8200:30757/TCP,8201:31801/TCP   155m
service/vault-internal         ClusterIP   None            <none>        8200/TCP,8201/TCP               155m
service/vault-stats            ClusterIP   10.106.10.219   <none>        56790/TCP                       155m

NAME                             READY   AGE
statefulset.apps/es-quickstart   3/3     166m
statefulset.apps/vault           1/1     155m

NAME                                               TYPE                       VERSION   AGE
appbinding.appcatalog.appscode.com/es-quickstart   kubedb.com/elasticsearch   7.9.1     166m
appbinding.appcatalog.appscode.com/vault                                                155m

NAME                                                      STATUS     AGE
databaseaccessrequest.engine.kubevault.com/es-cred-rqst   Approved   134m

NAME                                                   STATUS    AGE
elasticsearchrole.engine.kubevault.com/es-quickstart   Success   135m

NAME                                              STATUS    AGE
secretengine.engine.kubevault.com/es-quickstart   Success   141m

NAME                              REPLICAS   VERSION   STATUS    AGE
vaultserver.kubevault.com/vault   1          1.7.0     Running   155m

NAME                                                                   STATUS    AGE
vaultpolicybinding.policy.kubevault.com/vault-auth-method-controller   Success   153m

NAME                                                            STATUS    AGE
vaultpolicy.policy.kubevault.com/vault-auth-method-controller   Success   153m

NAME                                     VERSION          STATUS   AGE
elasticsearch.kubedb.com/es-quickstart   xpack-7.9.1-v1   Ready    166m
```

* `$ kubectl get all  -n kube-system`
```
pod/coredns-66bff467f8-8lb4p                     1/1     Running   0          3h15m
pod/coredns-66bff467f8-x84ss                     1/1     Running   0          3h15m
pod/etcd-kind-control-plane                      1/1     Running   0          3h15m
pod/kindnet-k4bq4                                1/1     Running   0          3h15m
pod/kube-apiserver-kind-control-plane            1/1     Running   0          3h15m
pod/kube-controller-manager-kind-control-plane   1/1     Running   0          3h15m
pod/kube-proxy-mhcvr                             1/1     Running   0          3h15m
pod/kube-scheduler-kind-control-plane            1/1     Running   0          3h15m
pod/kubedb-kubedb-autoscaler-5dd66688db-dwbc2    1/1     Running   0          169m
pod/kubedb-kubedb-community-7566877c7d-jsd7x     1/1     Running   0          169m
pod/kubedb-kubedb-enterprise-7b4b5db7ff-kcs6f    1/1     Running   0          169m
pod/kubevault-community-6649447d44-q6pg2         1/1     Running   0          160m

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns                   ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   3h16m
service/kubedb-kubedb-autoscaler   ClusterIP   10.105.13.6      <none>        443/TCP                  169m
service/kubedb-kubedb-community    ClusterIP   10.102.183.227   <none>        443/TCP                  169m
service/kubedb-kubedb-enterprise   ClusterIP   10.103.179.238   <none>        443/TCP                  169m
service/kubevault-community        ClusterIP   10.106.51.16     <none>        443/TCP                  160m

NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/kindnet      1         1         1       1            1           <none>                   3h16m
daemonset.apps/kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   3h16m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns                    2/2     2            2           3h16m
deployment.apps/kubedb-kubedb-autoscaler   1/1     1            1           169m
deployment.apps/kubedb-kubedb-community    1/1     1            1           169m
deployment.apps/kubedb-kubedb-enterprise   1/1     1            1           169m
deployment.apps/kubevault-community        1/1     1            1           160m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-66bff467f8                    2         2         2       3h15m
replicaset.apps/kubedb-kubedb-autoscaler-5dd66688db   1         1         1       169m
replicaset.apps/kubedb-kubedb-community-7566877c7d    1         1         1       169m
replicaset.apps/kubedb-kubedb-enterprise-7b4b5db7ff   1         1         1       169m
replicaset.apps/kubevault-community-6649447d44        1         1         1       160m
```

* [KubeDB](https://kubedb.com/)
* [KubeVault](https://kubevault.com/)
* [Elasticsearch Secret Engine](https://www.vaultproject.io/docs/secrets/databases/elasticdb)
* [Integrated Storage Backend - Raft](https://www.vaultproject.io/docs/configuration/storage/raft)
