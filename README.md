# Appscode Demo (28-05-21)
### Manage Elasticsearch credentials using the KubeVault Operator, Secret Engine and the Integrated Storage Backend (Raft) to persist Vault's data.

### Install KubeDB Enterprise Edition
* Create kind cluster - `kind create cluster --image kindest/node:v1.18.15@sha256:5c1b980c4d0e0e8e7eb9f36f7df525d079a96169c8a8f20d8bd108c0d0889cc4`
* Go to the AppsCode License Server & use the cluster ID to get the license of the KubeDB Enterprise Edition & put it in `license.txt`. Get the Cluster ID - `kubectl get ns kube-system -o=jsonpath='{.metadata.uid}'`
* Install KubeDB using Helm3 
`helm repo add appscode https://charts.appscode.com/stable/` 
`helm repo update`
`helm search repo appscode/kubedb`

```
helm install kubedb appscode/kubedb \
    --version v2021.04.16 \
    --namespace kube-system \
    --set-file global.license=/home/sakib/demo-es-raft/license.txt \
    --set kubedb-enterprise.enabled=true \
    --set kubedb-autoscaler.enabled=true
```

### Elasticsearch quickstart using KubeDB
* Create the namespace `demo`
`kubectl create namespace demo`
* Find available StorageClass
`kubectl get storageclass`
* Find available Elasticsearch version
`kubectl get elasticsearchversions`
* Create an Elasticsearch Cluster
`kubectl apply -f elasticsearch.yaml`
* Use port forwarding to connect with Elasticsearch database
`kubectl port-forward -n demo svc/es-quickstart 9200`
* Connection information
Username: `kubectl get secret -n demo es-quickstart-elastic-cred -o jsonpath='{.data.username}' | base64 -d`
Password: `kubectl get secret -n demo es-quickstart-elastic-cred -o jsonpath='{.data.password}' | base64 -d`
* Check health of the Elasticsearch database
`curl -XGET -k -u '<username>:<password>' "https://localhost:9200/_cluster/health?pretty"`

### Install KubeVault Operator
* `cd cd go/src/kubevault.dev/operator/` branch - `raft`
* `export REGISTRY=sakibalamin`
* `make push install`

### Deploy VaultServer
* Patch the VaultServer version - `kubectl apply -f vsversion.yaml`
* Deploy VaultServer - `kubectl apply -f vaultserver.yaml`
* Get all the Secrets - `kubectl get secret -n demo`
* Get the vault keys (vault-root-token, unseal-keys) - `kubectl get secret -n demo vault-keys -o yaml`
* Base-64 decode the vault-root-token - `echo "cy5yenVIQUNsMDdtUzhzemF3QWNuZ202dlk=" | base64 -d`
* Export as VAULT_TOKEN ENV VAR - `export VAULT_TOKEN=s.rzuHACl07mS8szawAcngm6vY`
* Export VAULT_ADDR - `export VAULT_ADDR='https://127.0.0.1:8200'`
* Export VAULT_SKIP_VERIFY - `export VAULT_SKIP_VERIFY=true`
* Use port forwarding to connect with Vault - `kubectl port-forward -n demo svc/vault 8200`
* Check Vault Status - `vault status` 
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

* Check vault's secrets list - `vault secrets list`
```
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_ecff0777    per-token private secret storage
identity/     identity     identity_c76ff30a     identity store
sys/          system       system_7e29f09c       system endpoints used for control, policy and debugging
```

* Check vault's policy list - `vault policy list`
```
default
k8s.-.demo.vault-auth-method-controller
vault-policy-controller
root
```

* Enable `custom-database-path` of `database` secret
`vault secrets enable -path=custom-database-path database`

* Create the SecretEngine
`kubectl apply -f secretengine.yaml`

* Check vault's secrets list - `vault secrets list`
```
Path                     Type         Accessor              Description
----                     ----         --------              -----------
cubbyhole/               cubbyhole    cubbyhole_ecff0777    per-token private secret storage
custom-database-path/    database     database_7fd68639     n/a
identity/                identity     identity_c76ff30a     identity store
sys/                     system       system_7e29f09c       system endpoints used for control, policy and debugging
```

* Check vault's policy list - `vault policy list`
```
default
k8s.-.demo.es-quickstart
k8s.-.demo.vault-auth-method-controller
vault-policy-controller
root
```

* Create the Secret Engine Role
`kubectl apply -f secretenginerole.yaml`

* Create a database access request
`kubectl apply -f dbaccessrequest.yaml`

* To Approve the request - `kubectl vault approve databaseaccessrequest es-cred-rqst -n demo`
* To Deny the request - `kubectl vault deny databaseaccessrequest es-cred-rqst -n demo`

* Check the database access request status - `kubectl get databaseaccessrequest es-cred-rqst -n demo -o json | jq '.status'`
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
`kubectl get secret -n demo es-cred-rqst-z3pdjr -o yaml -o json`
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

`kubectl get secret -n demo es-cred-rqst-z3pdjr -o yaml -o json | jq '.data'`

```
{
  "password": "MHYtYW5vTW1KZHgta3g3d3RMYzk=",
  "username": "di1rdWJlcm5ldGVzLWRlbW8tazhzLi0uZGVtby5lcy1xLWM2SE5NZU1seFRaZ0Z3a1pCcHFyLTE2MjE2NzU5MDM="
}
```