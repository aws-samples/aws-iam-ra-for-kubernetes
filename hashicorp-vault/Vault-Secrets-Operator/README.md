# IAM-RA Vault Secrets Operator


## Vault Secrets Operator (VSO)
https://developer.hashicorp.com/vault/docs/platform/k8s/vso

## Install
https://developer.hashicorp.com/vault/docs/platform/k8s/vso/installation
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com && \
  helm search repo hashicorp/vault-secrets-operator && \
  helm install --create-namespace --namespace vault-secrets-operator vault-secrets-operator hashicorp/vault-secrets-operator
```

## Configuration
Create a `VaultConnection` resource in the `frontend` `Namespace` that will configure an endpoint for the VSO to use when interacting with Vault.
Apply the manifest:
```bash
cat <<EOF | kubectl create --save-config=true --filename -
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  namespace: frontend
  name: default
spec:
  address: http://vault.vault.svc.cluster.local:8200
EOF
```

Create a `VaultAuth` resource in the `frontend` `Namespace` that will use the Kubernetes authentication method and a `ServiceAccount` trust to authenticate and authorize VSO interactions with Vault.
Apply the manifest:
```bash
cat <<EOF | kubectl create --save-config=true --filename -
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: static-auth
  namespace: frontend
spec:
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: iamanywhere
    serviceAccount: ${VAULT_SERVICE_ACCOUNT}
    audiences:
      - vault
  vaultConnectionRef: default
EOF
```

### Create a Secret
Verify that the VSO is working correclty by creating a `VaultStaticSecret` resource which will create a Kubernetes `Secret` from an existing Vault secret.
Apply the manifest:
```bash
cat <<EOF | kubectl create --save-config=true --filename -
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: test-secret
  namespace: frontend
spec:
  type: kv-v1
  mount: secret
  path: iamanywhere/test
  destination:
    name: test-secret
    create: true
  refreshAfter: 30s
  vaultAuthRef: static-auth
EOF
```

Verify that the Kubernetes `Secret` was created:
```bash
kubectl --namespace frontend get secret test-secret
```

Expect output similar to:
```bash
NAME          TYPE     DATA   AGE
test-secret   Opaque   3      86s
```

### Use the Secret
Reuse the manifest from before, but add an `env` variable referencing the secret create by the Vault Secrets Operator:
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
    - name: TEST_SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: test-secret
          key: key
    - name: TEST_SECRET_VALUE
      valueFrom:
        secretKeyRef:
          name: test-secret
          key: value
    - name: TEST_SECRET
      value: \$(TEST_SECRET_KEY):\$(TEST_SECRET_VALUE)
    image: ubuntu:latest
    imagePullPolicy: Always
  serviceAccount: ${VAULT_SERVICE_ACCOUNT}
  serviceAccountName: ${VAULT_SERVICE_ACCOUNT}
EOF
```

Verify that the secret data is now available within the Pod:
```bash
kubectl --namespace frontend exec -it demo-pod -- /bin/bash

root@demo-pod:/# printenv | grep "TEST_SECRET"
TEST_SECRET_KEY=hello
TEST_SECRET_VALUE=world
TEST_SECRET=hello:world
```
