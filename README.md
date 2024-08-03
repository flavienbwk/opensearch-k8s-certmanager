# OpenSearch on K8s with cert-manager

Add certs autorenewal capabilities to your OpenSearch cluster with [cert-manager](https://cert-manager.io/docs/installation/kubectl/).

This repo exists because neither the official [OpenSearch Kubernetes Helm chart](https://github.com/opensearch-project/helm-charts) nor the operator [currently support auto cert renewal](https://github.com/opensearch-project/opensearch-k8s-operator/issues/399).

We want to easily automate it using cert-manager for certs management and [Reloader](https://github.com/stakater/Reloader?tab=readme-ov-file#how-to-use-reloader) for rolling update through annotations.

Pre-requisites:

- A Kubernetes cluster up and running, accessible through `kubectl` ;
- (optionally) An OpenSearch cluster deployed with Helm.

## Install

1. Install cert-manager if you don't have it already

    ```bash
    # Get the latest version at https://cert-manager.io/docs/installation/kubectl/
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.2/cert-manager.yaml
    ```

2. Generate certificates

    ```bash
    kubectl apply -f ./os-certs-cm.yml
    ```

    To further test the renewal of a certificate, delete its associated secret.

3. Install Reloader

    ```bash
    helm repo add stakater https://stakater.github.io/stakater-charts
    helm install reloader stakater/reloader --set reloader.watchGlobally=true --set reloader.reloadOnCreate=true
    ```

4. Setup your OpenSearch cluster with the appropriate cert-manager secrets and Reloader annotations

    <details>
    <summary>👉 Deploy a new OpenSearch cluster with Helm...</summary>

    Just copy paste the following commands or read the configuration to adapt yours:

    ```bash
    helm repo add opensearch https://opensearch-project.github.io/helm-charts/

    kubectl apply -f ./os-config.yml  # OS configuration (users, roles...)
    helm install opensearch-nodes opensearch/opensearch --version 2.21.0 -f "./opensearch-example.yaml"
    helm install opensearch-dashboards opensearch/opensearch-dashboards --version 2.19.1 -f "./opensearch-dashboards-example.yaml"

    kubectl exec -it opensearch-cluster-master-0 -- /usr/share/opensearch/plugins/opensearch-security/tools/securityadmin.sh \
        -cd /usr/share/opensearch/config/opensearch-security/ \
        -icl -nhnv \
        -cacert /usr/share/opensearch/config/certs/ca.crt \
        -cert /usr/share/opensearch/config/admin-certs/tls.crt \
        -key /usr/share/opensearch/config/admin-certs/tls.key \
        -t config.yml \
        -t roles.yml \
        -t roles_mapping.yml \
        -t internal_users.yml \
        -t action_groups.yml \
        -t nodes_dn.yml \
        -t whitelist.yml \
        -t allowlist.yml \
        -t audit.yml \
        -t tenants.yml
    ```

    :information_source: Each time you edit the `os-config.yml` file, you'll need to run the `securityadmin.sh` command.

    </details>

    <details>
    <summary>👉 Manually update your existing OpenSearch cluster...</summary>

    Whether it is deployed with Helm or the Operator, you want to understand the basic principales of cert-manager and Reloader **so you can update your own manifests/Helm values**.

    At step 2, we've created cert-manager certificates : `os-certs`, `os-admin-certs` and `os-dashboards-certs`. These certs are created as _secrets_ in our cluster. Those secrets must be mounted to our cluster.

    Each secret includes a `ca.crt`, `tls.crt` and `tls.key` field we'll need to map in our `opensearch.yml` and `opensearch-dashboards.yml` configurations.

    Take example on [`opensearch-example.yaml#L67`](./opensearch-example.yaml#L67) to mount these secrets appropriately.

    Take example on [`opensearch-example.yaml#L28-L38`](./opensearch-example.yaml#L28) to configure the right paths.

    When cert-manager renews our certificates, we want our OpenSearch pods to reload so they use the new certs. That's where Reloader comes into play. We want our OpenSearch _StatefulSet_ and OpenSearch Dashboard _Deployment_ to add the proper annotations.

    You might want to use the following command as example to patch your current deployment :

    ```bash
    kubectl patch statefulset opensearch-cluster-master -p '{"spec":{"template":{"metadata":{"annotations":{"secret.reloader.stakater.com/reload": "os-certs"}}}}}'
    kubectl patch deployment opensearch-dashboards -p '{"spec":{"template":{"metadata":{"annotations":{"secret.reloader.stakater.com/reload": "os-dashboards-certs"}}}}}'
    ```

    Make it permanent by taking example on [`opensearch-example.yaml#L9`](./opensearch-example.yaml#L9) to configure the right annotations.

    <details>
