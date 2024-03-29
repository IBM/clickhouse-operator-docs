# Install ClickHouse Operator

## Prerequisites

1. OpenShift version from 4.10.x to 4.13.x
1. The `oc` command, downloadable from [here](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/).
1. Properly configured `kubectl`
1. `curl`

## Coexistence expectations

1. IBM ClickHouse Operator is best to use ```All Namespaces``` install mode.
    This reduces overall footprints of the operator in one cluster.
    It also guarantees that operator version and CRD version are matched.
1. The ```Own Namespace``` install mode is supported.
    However, it has a potential risk of CRD version collision, if two different versions of the operator are installed in one cluster.
1. Mixing ```All Namespaces``` and ```Own Namespace``` install mode in one cluster is unsupported.
    These setup will have the risk of overwriting a CR status from different operators, which may cause unexpected operator behavior.
1. Installing two ```Own Namespace``` operators in the same namespace is not supported either.
    This can't be done through the OpenShift's GUI. However, it might be possible through manipulating subscriptions directly.

## ZooKeeper

If you need data replication, please install a ZooKeeper cluster on the same OpenShift by following [ZooKeeper Setup](./zookeeper_setup_openshift.md). Or, if you want to use ClickHouse Keeper instead, see [ClickHouse Keeper Setup](./clickhouse_keeper_setup_openshift.md) instead.

## Configure username and password for the ```clickhouse operator``` user

Note: This is a required step for the v1.0.0 release. 
If you are installing v1.1.0 and above, the operator will generate this secret for you with ```clickhouse_operator``` and ```clickhouse_operator_password```.

There is an important role called ```clickhouse-operator``` to be configured.
Its main purpose is to manage ClickHouse instances from the ClickHouse Operator pod.
Follow the instructions below to configure it:

1. Make sure you are in the project (namespace) the clickhouse-operator pod to be deployed.
1. Click the **plus** button near the top right corner\
    ![Plus button](./img/plus_button.png)
1. The following template shows secret fields to be configured.
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: clickhouse-operator
    type: Opaque
    stringData:
      username: <operator-name>
      password: <operator-password>
    ```
1. Copy and paste the manifest above, replace desire name and password, and click the **Create** button.

## Install via OperatorHub GUI

### Create a new project

Note: If you're going to use the **All namespaces** install mode, you can skip the following steps.

1. Click **Home > Projects** on the left, then
1. Click the **Create Project** button on the right
1. Type a proper name in the **Name**, such as `my-clickhouse`
1. Click the **Create** button

### Add Catalog Source

1. Click the **plus** button near the top right corner\
    ![Plus button](./img/plus_button.png)
1. Paste the following yaml file to the text box, and click the **Create** button at the bottom.
    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: ibm-clickhouse-operator-catalog
      namespace: openshift-marketplace 
    spec:
      sourceType: grpc
      image: icr.io/cpopen/ibm-clickhouse-operator-catalog:<version>
      displayName: IBM ClickHouse Operator Catalog
    ```
    Note: Please specify the right image version tag while adding the catalog source.
    All available releases can be found [here](../releases).
    
    After a while, you will see the catalog is ready as follows:
    ![Catalog ready](./img/catalog_ready.png)

### Install operator from OperatorHub

1. Go to **Operators > OperatorHub** on the left
1. Type `clickhouse` into **Filter by keyword...** input box, the **IBM ClickHouse Operator** tile will be shown as follows:\
    ![Filter clickhouse](./img/filter_clickhouse.png)
1. Click the tile and the installation page will appear
1. Click the **Install** button to go to the configuration page:
    ![Install clickhouse operator](./img/install_clickhouse_operator.png)
1. If you choose to install the operator for all namespaces, the **openshift-operators** namespace will automatically selected.
   Otherwise, select the previously created namespace, such as `my-clickhouse` and click the **Install** button.
   After a while, you will see the message: Installed operator - ready for use.

### Verify operator is running

1. Click **Operators > Installed Operator** on the left, and switch project to the desired one, such as `my-clickhouse` as follows:\
    ![Switch to my-clickhouse ns](./img/switch_to_my-clickhouse_ns.png)
1. After you switch to the desired namespace, you can see it is in the Installed Operators list.
    ![Installed clickhouse operator](./img/installed_clickhouse_operator.png)