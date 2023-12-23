# IAM-RA Vault Server

## Vault Server
https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-intro

**Note:** For security reasons the Vault Server will not be exposed beyond the scope of the Kubernetes cluster where it is installed; if exposing any Vault Server endpoints, administrators are strongly cautioned to take great care in security controls limiting network access and properly scoping Role-Based Access Control (RBAC) both on the Kubernetes cluster and within Vault Roles and Policies.

## Vault Install and Initialize

1. [Install Vault](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-raft-deployment-guide#setup-helm-repo):
```bash
kubectl create namespace vault && \
  helm repo add hashicorp https://helm.releases.hashicorp.com && \
  helm repo update && \
  helm search repo hashicorp/vault && \
  helm install vault hashicorp/vault --namespace vault
```

2. Initialize Vault Server:
```bash
kubectl --namespace vault exec --stdin=true --tty=true vault-0 -- vault operator init
```

Expect output similar to:
```bash
Unseal Key 1: ###
Unseal Key 2: ###
Unseal Key 3: ###
Unseal Key 4: ###
Unseal Key 5: ###

Initial Root Token: ###

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

3. Use the keys above to unseal the Vault Server (single quotes aren't required but keys can include special characters):
```bash
for KEY in '###' '###' '###'; do 
kubectl --namespace vault exec --stdin=true --tty=true vault-0 -- vault operator unseal $KEY; 
done
```

Expect output similar to:
```bash
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
...
```

4. Verify Vault Server seal status:
```bash
kubectl --namespace vault exec --stdin=true --tty=true vault-0 -- vault status
```

Expect output similar to:
```bash
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         ###
Build Date      ###
Storage Type    file
Cluster Name    vault-cluster-###
Cluster ID      ###
HA Enabled      false
```

5. Interact with Vault Server:
Choose an interaction model for Vault that best matches the requirements of the organization:

- local client forwarded (this assumes that a vault client is installed locally):
```bash
kubectl --namespace vault port-forward svc/vault 8200:8200 > /dev/null 2>&1 &
export export VAULT_TOKEN=###
export VAULT_ADDR=http://localhost:8200
vault status
```

- shell on vault pod:
```bash
kubectl --namespace vault exec -it vault-0 -- /bin/sh
/ $ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    5
Threshold       3
Version         ###
Build Date      ###
Storage Type    file
Cluster Name    vault-cluster-###
Cluster ID      ###
HA Enabled      false
```

- standalone pod including a vault client:
```bash
/ $ export VAULT_ADDR=http://vault.vault.svc.cluster.local:8200
/ $ export VAULT_TOKEN=###
/ $ vault status
```

- the Vault web interface won’t be captured in this documentation but is a viable option

## Vault Kubernetes Authentication Method
[documentation](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
Although not required, this will enable pods to use the Kubernetes-attached service account for authentication with Vault and streamline the process of acquiring Vault secrets without compromising the trust model. Note that this does implicitly trust any upstream CI/CD processes which are granted authority to assign Kubernetes service accounts to Pods; effectively Vault trusts Kubernetes and Kubernetes trusts CI/CD therefore, in a heavily stratified model, the CI/CD pipeline is responsible for deciding which AWS IAM Anywhere Roles will be available on a per-Pod (or abstracted as a Distribution) level.
Also note that if this authentication method is not configured, then a valid Vault token must be provided to the `gen-cert.py` script directly via an environment variable.

### Vault Kubernetes Auth - Setup
1. Begin with the Kubernetes trust components:
One or more Kubernetes `ServiceAccount` will be required to complete the steps in this document; although not exhaustive or necessarily the best fit for every organization, an example configuration is provided here for illustrative purposes.

The name of the `ServiceAccount` is arbitrary but *must* be consistent throughout in order to properly establish the required authentication and trust; here `vault-auth` is used as the name:
```bash
export VAULT_SERVICE_ACCOUNT='vault-auth'

cat <<EOF | kubectl create --save-config=true --filename -
---
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: frontend
  name: frontend

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${VAULT_SERVICE_ACCOUNT}
  namespace: frontend

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: ${VAULT_SERVICE_ACCOUNT}
  namespace: frontend

---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: vault-auth-secret
  namespace: frontend
  annotations:
    kubernetes.io/service-account.name: "${VAULT_SERVICE_ACCOUNT}"
EOF
```
Note the quotes around the service account name - these are *required*.

2. Enable Kubernetes authentication method in Vault:
```bash
vault auth enable kubernetes
```

3. Gather metadata to configure the Vault Kubernetes authentication method:
```bash
export SA_JWT_TOKEN=$(kubectl get secret vault-auth-secret \
  --namespace frontend \
  --output 'go-template={{ .data.token }}' \
  | base64 --decode)
  
export SA_CA_CRT=$(kubectl config view \
  --raw \
  --minify \
  --flatten \
  --output 'jsonpath={.clusters[].cluster.certificate-authority-data}' \
  | base64 --decode) 

export K8S_HOST=$(kubectl config view \
  --raw \
  --minify \
  --flatten \
  --output 'jsonpath={.clusters[].cluster.server}')
```

4. Using this metadata, configure the Vault Kubernetes authentication method:
```bash
vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT" \
  issuer="https://kubernetes.default.svc.cluster.local"
```

5. Create a [Policy](https://developer.hashicorp.com/vault/docs/concepts/policies) for the Vault Role; the simple policy shown here (which will be expanded on throughout the documentation) provides access to all secrets that begin with path `secret/iamanywhere/`:
```bash
vault policy write iamanywhere - <<EOF
path "secret/iamanywhere/*" {
   capabilities = ["read"]
}
EOF
```
Note that this policy should generally *not* be used for production workloads as it provides access to every iamanywhere secret; this is for testing purposes only and proper scoping of policies and permissions is outside of the scope of this document.

6. Create the Vault Role tied to the Kubernetes authentication method:
```bash
vault write auth/kubernetes/role/iamanywhere \
    bound_service_account_names=${VAULT_SERVICE_ACCOUNT} \
    bound_service_account_namespaces=frontend,backend \
    policies=default,iamanywhere \
    ttl=1h
```
The value provided to `bound_service_account_names` *must* correspond to one or more Kubernetes ServiceAccount names (although these need not exist prior to configuring Vault, as shown here).
The value provided to `bound_service_account_namespaces` *must* correspond to one or more Kubernetes Namespace names

The combination of values configuring bound policies, Kubernetes ServiceAccounts, and Kubernetes Namespaces implements the scoping of permissions provided to each Vault role. These roles may overlap in none, some, many, or all of these options and should reflect the requirements and posture of each respective organization.

Ensuring agreement of these values is crucial to achieving a working configuration; that is, if a future project adds a new Namespace to a Kubernetes cluster then Pods in that Namespace will not be able to retrieve AWS IAM Anywhere Roles until a new Vault role is created or an existing role is updated to reflect that Namespace.



### Vault Kubernetes Auth - Demo Pod

Now that Vault is all setup, run a Pod to verify that Kubernetes Auth is working as expected:

#### Create a Demo Vault Secret
1. Enable the Vault secrets engine (version one is shown throughout) at a path of secrets:
```bash
vault secrets enable -path=secret kv
```

Note that the profile specified above configures a sub-path of secrets - `secret/iamanywhere`


2. Validate the secrets engine:
```bash
vault kv put secret/iamanywhere/test key='hello' value='world'
```

```bash
vault kv get secret/iamanywhere/test
```

Expect output similar to:
```bash
==== Data ====
Key      Value
---      -----
key      hello
value    world
```


#### Launch a Demo Pod


1. Create a Pod for testing:
```yaml
cat <<EOF | kubectl create --save-config=true --filename -
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: demo-pod
  name: demo-pod
  namespace: frontend
spec:
  containers:
  - name: demo
    args:
    - while true; do sleep 30; done;
    command:
    - /bin/bash
    - -c
    - --
    env:
    - name: VAULT_ADDR
      value: http://vault.vault.svc.cluster.local:8200
    - name: VAULT_ROLE
      value: iamanywhere
    image: ubuntu:latest
    imagePullPolicy: Always
  serviceAccount: ${VAULT_SERVICE_ACCOUNT}
  serviceAccountName: ${VAULT_SERVICE_ACCOUNT}
EOF
```
- note: the `Namespace` value *must* match above
- note: the `serviceAccount` and `serviceAccountName` values *must* match above
- note: `VAULT_ADDR` and `VAULT_ROLE` are populated for conveniece and will be used later but are optional

2.  Confirm that the Pod is running:
```bash
kubectl --namespace frontend get pods --selector='app=demo-pod'
```

Expect output similar to:
```bash
NAME       READY   STATUS    RESTARTS   AGE
demo-pod   1/1     Running   0          20s
```


#### Verify Kubernetes Auth

1. Launch an interactive shell on the Pod:
```bash
kubectl --namespace frontend exec -it \
  $(kubectl --namespace frontend get po \
    --selector='app=demo-pod' \
    --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'\
  ) \
    -- /bin/bash
```

2. Install supporting software packages:
```bash
apt update && \
apt install -y curl jq  && \
apt clean all
```

3. Gather authentication data:
```bash
JWT="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" && \
VAULT_TOKEN="$(curl --request POST --data '{"jwt": "'"$JWT"'", "role": "iamanywhere"}' -s -k $VAULT_ADDR/v1/auth/kubernetes/login | jq -r '.auth.client_token')"
echo $VAULT_TOKEN
```

4. Test authentication (and authorization) by requesting the secret created earlier:
```bash
curl -H "X-Vault-Token: $VAULT_TOKEN" -X GET $VAULT_ADDR/v1/secret/iamanywhere/test | jq
```
