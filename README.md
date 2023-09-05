# EKS with VSO and HCP Vault using JWT/OIDC

*How to deploy and configure the Vault Secrets Operator (VSO) on EKS and consume secrets from HCP Vault, using the JWT/OIDC authentication method.*

At the end of this demo, a Kubernetes pod running on EKS will be able to consume static secrets from HCP Vault as environment variables. The pod will have no knowledge about Vault, no sidecar: just consuming native Kubernetes secrets, while on the other hand, the secrets lifecycle is managed with Vault.

## prerequisites

Have these pieces of infrastructure ready:

- hcp hvn,
- hcp vault,
- eks cluster.

### Configure Kubectl CLI

Update your kubectl config for the newly created AWS EKS cluster

```SHELL
aws eks --region eu-central-1 update-kubeconfig \
  --name eks-vso-demo \
  --alias eks-vso-demo
```

Verify that `kubectl` use the new context and can connect to the cluster's API.

```SHELL
> kubectl config get-contexts
CURRENT   NAME           CLUSTER                                                      AUTHINFO                        NAMESPACE
*         eks-vso-demo   arn:aws:eks:xxx:209xxx:cluster/eks-vso-demo   arn:aws:eks:xxx:209xxxcluster/eks-vso-demo

> kubectl get nodes
NAME                                          STATUS   ROLES    AGE     VERSION
ip-10-0-0-143.eu-central-1.compute.internal   Ready    <none>   4h14m   v1.27.3-eks-a5565ad
ip-10-0-1-49.eu-central-1.compute.internal    Ready    <none>   4h14m   v1.27.3-eks-a5565ad
```

### Configure Vault CLI

Set these environment variables to access your HCP cluster from vault CLI:

- VAULT_ADDR
- VAULT_TOKEN
- VAULT_NAMESPACE

You can gather this information from the HCP portal.

**Note: The Vault namespace should be set to `admin` when working with HCP Vault, as the root namespace is not accessible on HCP Vault.**

Verify that `vault` CLI can reach your HCP Vault cluster.

```SHELL
> vault status
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
...

> vault auth list
Path      Type        Accessor                  Description                              Version
----      ----        --------                  -----------                              -------
token/    ns_token    auth_ns_token_347f3941    token based credentials                  n/a
```

## Prepare the EKS cluster

This role binding ensures that the OIDC discovery URLs do not require authentication.

```SHELL
kubectl create clusterrolebinding oidc-reviewer  \
   --clusterrole=system:service-account-issuer-discovery \
   --group=system:unauthenticated
```

## Prepare HCP Vault

### Configure a static secret engine

Here, we configure a Vault KV secret engine and create sample secrets. Later, we will try to make these Vault static secrets available as Kubernetes secrets, using the Vault Secrets Operator (VSO).

```SHELL
vault secrets enable -path kvv2 kv-v2
vault kv put kvv2/myapp/config username='appuser' password='suP3rsec(et!' ttl='30s'
vault kv put kvv2/webserver db_username='dbadmin' db_password='Xu*e87fRs/9'
```

### Configure the JWT/OIDC auth method on HCP Vault

When using VSO, the Kubernetes pods require no knowledge of Vault to consume Vault secrets. It is VSO that will authenticate with Vault and retrieve secrets according to your configuration.

In this demo, VSO will authenticate with HCP Vault using the Vault JWT/OIDC auth method. The commands below retrieve your EKS Issuer URL using OIDC discovery and enable the JWT auth method.

```SHELL
ISSUER="$(kubectl get --raw /.well-known/openid-configuration | jq -r '.issuer')"
vault auth enable jwt
vault write auth/jwt/config oidc_discovery_url="${ISSUER}"
```

Now that the auth method is configured, we need to create a role and the corresponding policies, so an identified client can request the authorizations corresponding to its role.

```SHELL
vault policy write vso - <<EOF
path "kvv2/*" {
   capabilities = ["read", "list"]
}
EOF

vault write auth/jwt/role/vso \
   role_type="jwt" \
   bound_audiences="https://kubernetes.default.svc" \
   user_claim="sub" \
   policies="default,vso" \
   ttl="1h"
```

Here, we configured a role named `vso`, and attached the policy of the same name in addition to the `default` policy.

A Vault client able to authenticate with the JWT/OIDC method and requesting the role `vso` will be able to list and read any secrets in the KV secret engine mounted under the path `kvv2`. This is a simplistic example to show how VSO works, in a production context, you should plan and design a roles and policies strategy to fit your requirements.

## Deploy and configure VSO

To deploy the Vault Secrets Operator, we will use the official helm chart.

*Note: The `address` parameter in `vso_values.yaml` should match your HCP Vault's API address (in the form `https://<hcp-vaul-fqdn>:8200`)*

```SHELL
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  --version 0.1.0 \
  --namespace vault-secrets-operator-system --create-namespace \
  --values vso_values.yaml
```

Now that VSO is deployed, you need to declare how VSO will connect to HCP Vault to read secrets, using two Custom Resources:

- `VaultConnection` targets a specific Vault cluster
- `VaultAuth` defines the vault auth method, the vault role, and optionally a VaultConnection to use

The Helm chart creates a default VaultConnection object that will be used if no VaultConnection object is declared using `vaultConnectionRef`. But in this example, we will declare a specific VaultAuth object to illustrate what it would look like if different auth methods were needed.

```SHELL
kubectl apply -f vault-connection.yaml
kubectl apply -f vault-auth-jwt.yaml
```

At this stage, the EKS cluster is ready to consume secrets from the HCP Vault cluster. The next section will show how to tell VSO to synchronize specific Vault secrets with Kubernetes secrets.

## Configure Static Secrets synchronization

VSO can synchronize several types of Vault secrets. In the following example, we will configure static secrets.

The sample YAML file below synchronize the content under `myapp/config` from the `kvv2` mount, as a Kubernetes secret named `secret-webapp`, using the `eks-jwt-oidc` VaultAuth Custom Resource.

```YAML
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: vault-kvv2-myapp
  namespace: app
spec:
  type: kv-v2

  # mount path for the kv secrets engine
  mount: kvv2

  # path of the secret
  path: myapp/config

  # dest k8s secret
  destination:
    name: secret-myapp
    create: true

  # static secret refresh interval
  refreshAfter: 30s

  # Name of the CRD to authenticate to Vault
  vaultAuthRef: eks-jwt-oidc
```

Applying this file will configure the two Vault secrets we created earlier for synchronization with Kubernetes secrets.

```SHELL
kubectl apply -f static-secret-app.yaml
```

You can now verify that the Kubernetes secrets were automatically created by VSO

```SHELL
> kubectl get secrets -n app
NAME               TYPE     DATA   AGE
secret-myapp       Opaque   4      3s
secret-webserver   Opaque   3      3s
```

## Consume secrets from a pod

Now that the Vault secrets are created as Kubernetes secrets and that they are kept in sync by VSO, you can consume them from pods, as usual with Kubernetes secrets.

Deploy a sample pod that adds Kubernetes secrets as environment variables.

```SHELL
kubectl apply -f nginx-deployment.yaml
```

Verify the environment variables are present inside the pod.

```SHELL
> kubectl exec -it $(kubectl get pods -n app -o jsonpath='{.items[0].metadata.name}') -n app -- /bin/bash

root@nginx-deployment-55899c86ff-bxsn9:/# env |grep DB
DB_PASSWORD=Xu*e87fRs/9
DB_USERNAME=dbadmin
```
