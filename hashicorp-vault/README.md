# IAM Roles Anywhere on Kubernetes with Hashicorp Vault

**Note:** Sample code, software libraries, command line tools, proofs of concept, templates, or other related technology are provided as AWS Content or Third-Party Content under the AWS Customer Agreement, or the relevant written agreement between you and AWS (whichever applies). You should not use this AWS Content or Third-Party Content in your production accounts, or on production or other critical data. You are responsible for testing, securing, and optimizing the AWS Content or Third-Party Content, such as sample code, as appropriate for production grade use based on your specific quality control practices and standards. Deploying AWS Content or Third-Party Content may incur AWS charges for creating or using AWS chargeable resources, such as running Amazon EC2 instances or using Amazon S3 storage.

There are many different possible models for implementing [IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html) using [Hashicorp Vault](https://www.vaultproject.io/) as a Certificate Authority. In this project, certificate data will be considered sensitive and therefore secured away from applications running at the Pod-level. Because this configuration requires additional layers of abstraction in order to support the security model, some sections and components included here may be optional for other, less security-focused implementations.

This project will use the Vault KV Secrets Engine version 1, however, all operations are compatible with version 2 although commands and syntax will change slightly.

**Note:** On Permissions - It's important to always follow best practices when providing permissions for AWS IAM and Vault Policies ([documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege))
The steps detailed here establish three trusts that are important to security:
1. An AWS IAM Role, including the permissions granted to that Role,  that trusts IAM Roles Anywhere; effectively extending those permissions to any entity trusted by IAM-RA 
2. An IAM-RA Trust Anchor that trusts a Certificate Signing Authority - Vault PKI in this case - which extends the trust above to mean that any arbitrary actor trusted by the CA can receive the IAM permissions assigned by IAM-RA
3. The Vault Kubernetes Authentication Method which allows Vault RBAC to apply Vault Policy rules to a trusted Kubernetes Service Account
A misconfiguration anywhere in this chain of trust could unintentionally assign unintended permissions to an actor, so it is crucial that, at each layer of trust, least privilege standards be implemented

**Note:** This documentation assumes that a working Kubernetes cluster is already available and configured as the current `kubectl` context (`kubectl config current-context`).

**Note:** On environment variables: shell variables are sprinkled throughout the documentation and are required for copy / paste functionality, however, the primary goal of including these variables is to highlight where values *must* be consistent (such as when to use the `VAULT_CERT_CN` value vs the `VAULT_CERT_NAME` value)

Environment Variables that will be consumed by one or more sub-sections:
- `VAULT_ADDR` configures the URL for connecting to Vault with a command-line client
- `VAULT_TOKEN` configures the authentication token for connecting to Vault with a command-line client
- `VAULT_SERVICE_ACCOUNT` configures the string to use for the name of the Kubernetes `ServiceAccount` trusted by Vault
- `VAULT_CERT_NAME` configures the string to use when creating the root SSL Certificate
- `VAULT_CERT_CN` configures the string to use when creating signed (Common Name) SSL Certificates
- `ACCOUNT_NUMBER` configures the AWS account number corresponding to the Elastic Container Registry for the Docker image
- `IMAGE_TAG` configures the string to use when tagging and referencing the Docker image


Sub-Sections are intended to be followed in order:

1. [Vault Server Setup](Vault-Server/README.md)

2. [Vault Secrets Operator Setup](Vault-Secrets-Operator/README.md)

3. [Docker Container](Docker/README.md)

4. [Vault Certificate Authority](Vault-Certificate-Authority/README.md)

5. [Kubernetes](Kubernetes/README.md)
