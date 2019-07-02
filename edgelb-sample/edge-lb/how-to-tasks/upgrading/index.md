---
layout: layout.pug
navigationTitle:  Upgrading Edge-LB
title: Upgrading Edge-LB
menuWeight: 45
excerpt: Upgrading an Edge-LB installation
enterprise: true
---
In general, you should regularly update or upgrade the Edge-LB package you have installed to the latest version available to ensure you can take advantage of any fixes and new features that are included in the most recent release.

For information about what's new or fixed in any release, see the Edge-LB [release notes](/services/edge-lb/related-documentation/release-notes/) and [related documentation](/services/edge-lb/related-documentation/).

# Before you begin
* You must have Edge-LB installed as described in the Edge-LB [installation instructions](/services/edge-lb/getting-started/installing).
* You must have the core DC/OS command-line interface (CLI) installed and configured to communicate with the DC/OS cluster.
* You must have the `edgelb` command-line interface (CLI) installed.
* You must have an active and properly-configured DC/OS Enterprise cluster.

# To upgrade Edge-LB packages

1. Uninstall the `apiserver` package.

    ```bash
    dcos package uninstall edgelb --yes
    ```

1. Remove the old package repositories.

    ```bash
    dcos package repo remove edgelb-aws
    dcos package repo remove edgelb-pool-aws
    ```

1. Add the new package repositories.

    ```bash
    dcos package repo add --index=0 edgelb-aws \
      https://<insert download link>/stub-universe-edgelb.json
    dcos package repo add --index=0 edgelb-pool-aws \
      https://<insert download link>/stub-universe-edgelb-pool.json
    ```

1. Install the new `apiserver`. 

    Use the service account you created when you installed the previous version. For more information about creating and configuring permissions for the service account, see [Installing Edge-LB](/services/edge-lb/getting-started/installing) and [Service account permissions](/services/edge-lb/reference/permissions/#service-account-permission).
    
    The configuration file below matches the one created by following the installation instructions.

    ```bash
    tee edgelb-options.json <<EOF
    {
      "service": {
        "secretName": "dcos-edgelb/edge-lb-secret",
        "principal": "edge-lb-principal",
        "mesosProtocol": "https"
      }
    }
    EOF
    dcos package install --options=edgelb-options.json edgelb
    ```

    EdgeLB requires the following options to be specified based on the security mode of the cluster:
    - `service.mesosProtocol`
        - `"https"` for permissive or strict security
        - `"http"` (default) for disabled security

    - `service.mesosAuthNZ`
        - `true` (default) for permissive or strict security
        - `false` for disabled security mode

1. Upgrade each pool.

    ```bash
    dcos edgelb update <pool-file>
    ```
