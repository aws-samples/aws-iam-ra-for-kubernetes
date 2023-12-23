# IAM-RA Kubernetes


## Kubernetes Pod Setup
All of these components are combined to provide a Pod that requests on-the-fly temporary credentials from AWS IAM and hands off to an application Container:
```bash
cat <<EOF | kubectl create --save-config=true --filename -
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: demo
  name: demo
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
    image: public.ecr.aws/aws-cli/aws-cli:latest
    imagePullPolicy: Always
    volumeMounts:
    - name: aws-config
      mountPath: /root/.aws
  initContainers:
  - name: aws-iam-ra
    args:
    - python3 ./gen-cert.py ; read AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN < <(echo \$(./aws_signing_helper credential-process --certificate cert.pem --private-key private.pem --intermediates chain.pem --trust-anchor-arn \$ANCHOR_ARN --profile-arn \$PROFILE_ARN --role-arn \$ROLE_ARN | jq -r '.AccessKeyId, .SecretAccessKey, .SessionToken')) ; echo -e "[default]\naws_access_key_id=\${AWS_ACCESS_KEY_ID}\naws_secret_access_key=\${AWS_SECRET_ACCESS_KEY}\naws_session_token=\${AWS_SESSION_TOKEN}" > /home/nobody/.aws/credentials
    command:
    - /bin/bash
    - -c
    - --
    env:
    - name: VAULT_ADDR
      value: http://vault.vault.svc.cluster.local:8200
    - name: VAULT_ROLE
      value: iamanywhere
    - name: VAULT_CERT_NAME
      value: ${VAULT_CERT_CN}
    - name: VAULT_CERT_CN
      value: ${VAULT_CERT_CN}
    - name: ROLE_ARN
      valueFrom:
        secretKeyRef:
          key: arn
          name: role-arn
    - name: ANCHOR_ARN
      valueFrom:
        secretKeyRef:
          key: arn
          name: anchor-arn
    - name: PROFILE_ARN
      valueFrom:
        secretKeyRef:
          key: arn
          name: profile-arn
    image: ${ACCOUNT_NUMBER}.dkr.ecr.${REGION}.amazonaws.com/iamra:${IMAGE_TAG}
    imagePullPolicy: Always
    securityContext:
      runAsUser: 65534
    volumeMounts:
    - name: aws-config
      mountPath: /home/nobody/.aws
  serviceAccount: vault-auth
  serviceAccountName: vault-auth
  volumes:
  - name: aws-config
    emptyDir: {}
EOF
```

### Verify
Confirm that all is successful by executing a shell inside of the `demo` Pod:
```bash
kubectl --namespace frontend exec -it \
  $(kubectl --namespace frontend get po \
    --selector='app=demo' \
    --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'\
  ) \
    -- aws s3 ls
```

During Kubernetes `Pod` init the `gen-cert.py` script interacts with Vault to receive a signed Certificate, which is sent to IAM Roles Anywhere by the `aws_signing_helper` script. The outputs of the `aws_signing_helper` script are captured into the file `~/.aws/credentials` (`~` is the `root` user in this example). Also during `Pod` init, the `Container` consumes Kubernetes `Secret` resources that are created by the Vault Secrets Operator.
Using this model, none of the `Secret` values nor the Certificate contents nor the scripts to request Certificate contents are included in the resulting `Pod` (often termed the "Application Pod") and so this sensitive data is protected from accidental (or malicious) compromise at the application layer.
