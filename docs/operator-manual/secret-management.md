# Secret Management

There are two general ways to populate secrets when doing GitOps: on the destination cluster, or in Argo CD during 
manifest generation. We strongly recommend the former, as it is more secure and provides a better user experience.

For further discussion, see [#1364](https://github.com/argoproj/argo-cd/issues/1364).

## Destination Cluster Secret Management

In this approach, secrets are populated on the destination cluster, and Argo CD does not need to directly manage them.
[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets), [External Secrets Operator](https://github.com/external-secrets/external-secrets), [Kubernetes Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) and [SOPS-Operator](https://github.com/peak-scale/sops-operator) are examples of this style of secret management.

This approach has two main advantages:

1) Security: Argo CD does not need to have access to the secrets, which reduces the risk of leaking them.
2) User Experience: Secret updates are decoupled from app sync operations, which reduces the risk of unintentionally
   applying Secret updates during an unrelated release.

We strongly recommend this style of secret management.

Other examples of this style of secret management include:
* [aws-secret-operator](https://github.com/mumoshu/aws-secret-operator)
* [Vault Secrets Operator](https://developer.hashicorp.com/vault/docs/platform/k8s/vso)

## Argo CD Manifest Generation-Based Secret Management

In this approach, Argo CD's manifest generation step is used to inject secrets. This may be done using a 
[Config Management Plugin](config-management-plugins.md) like [argocd-vault-plugin](https://github.com/argoproj-labs/argocd-vault-plugin).

**We strongly caution against this style of secret management**, as it has several disadvantages:

1) Security: Argo CD needs access to the secrets, which increases the risk of leaking them. Argo CD stores generated 
   manifests in plaintext in its Redis cache, so injecting secrets into the manifests increases risk.
2) User Experience: Secret updates are coupled with app sync operations, which increases the risk of unintentionally
   applying Secret updates during an unrelated release.
3) Rendered Manifests Pattern: This approach is incompatible with the "Rendered Manifests" pattern, which is 
   increasingly becoming a best practice for GitOps.

Many users have already adopted generation-based solutions, and we understand that migrating to an operator-based 
solution can be a significant effort. Argo CD will continue to support generation-based secret management, but we will 
not prioritize new features or improvements that solely support this style of secret management.

### Mitigating Risks of Secret-Injection Plugins

Argo CD caches the manifests generated by plugins, along with the injected secrets, in its Redis instance. Those
manifests are also available via the repo-server API (a gRPC service). This means that the secrets are available to
anyone who has access to the Redis instance or to the repo-server.

Consider these steps to mitigate the risks of secret-injection plugins:

1. Set up network policies to prevent direct access to Argo CD components (Redis and the repo-server). Make sure your
   cluster supports those network policies and can actually enforce them.
2. Consider running Argo CD on its own cluster, with no other applications running on it.
