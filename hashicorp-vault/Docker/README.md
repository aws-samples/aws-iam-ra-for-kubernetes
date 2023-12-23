# IAM-RA Docker Container

## Docker Container Setup
In order to keep from exposing sensitive data - such as ARN values or SSL Certificates - a sidecar container will be used to request a Certificate from Vault, request temporary IAM credentials using the Certificate, and pass the credentials securely to the application container before exiting.

### gen-cert.py Script
This Python3 script will do the heavy lifting of interfacing with Vault to request a signed Certificate and format the contents into files expected by the AWS helper script to sent to IAM:
```python
import json

from hvac import Client
from os import environ
from hvac.api.auth_methods import Kubernetes

def write( filename, data ):
  FH = open( filename, 'w' )
  FH.writelines( data )
  FH.close

# Vault login with provided token
if environ['VAULT_TOKEN']:
    client = Client(
            url=environ['VAULT_ADDR'],
            token=environ['VAULT_TOKEN'],
    )
# Vault login with Kubernetes ServiceAccount token
elif not environ['VAULT_TOKEN']:
    client = Client(
            url=environ['VAULT_ADDR'],
    )
    f = open('/var/run/secrets/kubernetes.io/serviceaccount/token')
    jwt = f.read()
    f.close()
    role = environ['VAULT_ROLE']
    Kubernetes(client.adapter).login(
            role=role,
            jwt=jwt
    )
else:
    print( "Something has gone horribly wrong: VAULT_TOKEN" )


generate_certificate_response = client.secrets.pki.generate_certificate(
   name=environ['VAULT_CERT_CN'],
   common_name=environ['VAULT_CERT_CN'],
   mount_point='pki-aws'
)

cert = json.dumps( generate_certificate_response['data'] )

write( 'chain.pem', generate_certificate_response['data']['ca_chain'] )
write( 'cert.pem', [ generate_certificate_response['data']['certificate'],"\n",generate_certificate_response['data']['issuing_ca'] ] )
write( 'private.pem', generate_certificate_response['data']['private_key'] )
```

### Docerfile
This Dockerfil will copy the `gen-cert.py` script above into the container and dynamically download the AWS helper script from a URL (note that the version of the script is included in the URL):
```dockerfile
FROM public.ecr.aws/docker/library/python:slim-bullseye

ENV VAULT_ADDR=""
ENV VAULT_TOKEN=""
RUN mkdir -p /home/nobody && \
WORKDIR /home/nobody
COPY gen-cert.py .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir hvac

RUN apt update && apt install -y jq && \
    apt clean && rm -rf //var/lib/apt/lists/*

ADD --chmod=0755 https://rolesanywhere.amazonaws.com/releases/1.0.4/X86_64/Linux/aws_signing_helper ./

RUN mkdir -p /home/nobody/.aws/ && \
    chown -R nobody: /home/nobody
```

This Dockerfile simply installs the necessary components - such as Python libraries and the AWS Roles Anywhere helper - which leaves Vault requests calling the helper up to the Pod manifest. These activities could be internalized into the Container via the Dockerfile without losing functionality.

### Login to AWS ECR
In order to push the built container to AWS Elastic Container Registry, Docer must authenticate to AWS:
```bash
aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACCOUNT_NUMBER}.dkr.ecr.${REGION}.amazonaws.com
```

### Build and Push the Docker Container to AWS ECR
Build a container from the Dockerfile above and, once authenticated, push to ECR:
```bash
docker build \
  --file Dockerfile . \
  --push \
  --tag ${ACCOUNT_NUMBER}.dkr.ecr.${REGION}.amazonaws.com/iamra:${IMAGE_TAG}
```

If building from a different architecture - such as a Mac - use the `--platform` argument to specify the target architecture, which should match the EKS cluster; here that cluster is `amd64`
```bash
docker buildx build \
  --provenance=false \
  --platform linux/amd64 \
  --file Dockerfile . \
  --push \
  --tag ${ACCOUNT_NUMBER}.dkr.ecr.${REGION}.amazonaws.com/iamra:${IMAGE_TAG}
```
