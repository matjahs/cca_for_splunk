cca.splunk.onboarding
=========

Splunk apps deployment or `data onboarding` as we like to call it can be a daunting task to manage, especially in a larger Splunk environment. Within CCA for Splunk this has been solved by collecting all Splunk apps in a structure that will ease your management of all apps and enforce control at the same time.

Apps are split into several different types and directories in CCA, all names matching those you find in Splunk. We have the 4 standard `etc` sub directories that has been created by the setup wizard. These directories are available in your onboarding directory, a directory that should be a git repository.

 - splunk/etc/apps
 - splunk/etc/deployment-apps
 - splunk/etc/shcluster/apps
 - splunk/etc/manager-apps with a symlink to
   splunk/etc/master-apps for backwards compatibility.


## Apps
The apps directory holds multiple sub directories, one for `versioned` and one for `selectable` apps.

### Apps - versioned
In the `apps/versioned` directory all Splunkbase apps should be stored. The apps are extracted using a supplied script `build_appFile_from_splunk_app.sh` located in this directory.
The script extracts the real directory name from the app and uses that together with the version information to store the app with a new name, `appFile-SPLUNKBASE_APP_vAPP_VERSION`. Now multiple versions of the same app can be stored side by side and makes it easy to upgrade and even downgrade an app using CCA.

### Apps - selectable
In the `apps/selectable` directory, apps that is purposely built for an target should be store here. This can for example be, db inputs, HEC inputs, etc. The name of the app in these environment directories doesn't need to be the same as the app has when deployed to a Splunk instance.

If we take a look at `splunk_httpinput` this is a default app in Splunk Enterprise, so Splunk requires that the app has the proper name in Splunk. Luckily CCA allows us to name it different on the CCA manager server, for example `cloud_metrics-splunk_httpinput` or `dmz-splunk_httpinput`.
The mapping of source and destination name of an app is handled when configuring app allocation in `environments/ENVIRONMENT_NAME/group_vars/ANSIBLE_GROUP`

While discussing the `splunk_httpinput` app, this is the exception in CCA as it cannot be deleted by setting `state: 'absent'`. If you do, the default app for `splunk_httpinput` will be removed and Splunk will throw a warning next time it restarts. To overcome this, empty the `splunk_httpinput` app in CCA and deploy the it.

When you look in the inventory directory and in the group_vars directory structure there is only one forwarders group. If you different group of forwarders, target option can be set for each app. This enables full control of which apps that should be deployed where.

## Deployment-apps
Deployment-apps are essential if your infrastructure has any Universal Forwarders that are managed via one or more Deployment Servers. All apps that should be distributed to the Deployment Servers should be stored in either the `deployment-apps/ENVIRONMENT_NAME` or for specific ones in `deployment-apps/selectable` directory.

Any controlling `serverclass.conf` file should be configured in a dedicated app stored in `apps/selectable` as they will be deployed into the Deployment Servers apps directory.

In the group_vars template structure there is only one deployment-servers group, if additional granularity is needed, use `target` option that can be set for each app and to limit the deployment to that host only.

### Deployment-apps - selectable
Apps in the selectable directory needs to be cherry picked by specifying them in `environments/ENVIRONMENT_NAME/group_vars/ANSIBLE_GROUP` for relevant deployment_servers group_vars file.

### Deployment-apps - environment
All apps in the environments directory will be deployed to the deployment servers.

## Search Head Cluster - shcluster apps
Shcluster apps needs a bit extra care, CCA supports up to 9 parallel search head clusters per environment. This require an extra level of sub directories per environment.
`splunk/etc/shcluster/ENVIRONMENT_NAME/SHCLUSTER_LABEL` SHCLUSTER_LABEL must match the `shcluster_label` name in the `hosts` inventory file.
The default names are shcluster_c1-c9 and can be changed to a name with alphanumeric english characters, hyphen (`-`) and underscore (`_`). If you choose to update the shcluster_label name in the inventory file, remember to rename the directory in the onboarding repo. The SHCLUSTER_LABEL value will also be used as the Search Head Cluster label on the Search heads and visible in the Monitoring Console.

### Search Head Cluster - shcluster - selectable
Store environment specific apps here and cherry pick those that should be deployed to respective search head cluster by specifying them in `splunk/etc/shcluster/selectable`

### Search Head Cluster - shcluster - environment - shcluster_label name
All apps stored in a `splunk/etc/shcluster/ENVIRONMENT_NAME/SHCLUSTER_LABEL` directory will be merged with cherry picked apps from the selectable directory and deployed to the search head deployer. This eases configuration management of apps where custom apps can just be added and deployed as a whole.

## Index Cluster - manager-apps
Manager apps needs a bit extra care, CCA supports up to 9 parallel index clusters per environment. This require an extra level of sub directories per environment.

`splunk/etc/manager-apps/ENVIRONMENT_NAME/CLUSTER_NAME` CLUSTER_NAME must match the `cluster_label` name in the `hosts` inventory file. The default names are cluster_c1-c9 and can be changed to a name with alphanumeric english characters, hyphen (`-`) and underscore (`_`). If you choose to update the cluster_label name in the inventory file, remember to rename the directory in the onboarding repo. The CLUSTER_NAME value will also be used as the Cluster label on the Index Cluster and be visible in the Monitoring Console.

### Index Cluster - manager-apps - selectable
Store environment specific apps here and cherry pick those that should be deployed to respective cluster manager cluster by specifying them in `environments/ENVIRONMENT_NAME/group_vars/cluster_manager_cluster_cX`

### Index Cluster - manager-apps - environment - cluster_label name
All apps stored in a `splunk/etc/manager-apps/ENVIRONMENT_NAME/CLUSTER_NAME` directory will be merged with any cherry picked apps and deployed to the cluster manager. This eases configuration management of apps where custom apps can just be added and deployed as a whole.

Playbooks
---------

There is 4 different playbooks that address a specific deployment scope and 1 that includes all of them and execute each of them in a predefined order.

* deploy_manager_apps.yml
* deploy_shcluster_apps.yml
* deploy_deployment_apps.yml
* deploy_apps.yml

* deploy_fullstack.yml

Requirements
------------

Splunk Apps must be stored in the dedicated onboarding repo and in the specified directory structure. Group vars files needs to be updated with apps that are to be deployed.

Role Variables
--------------

Description

### group_vars/hosts
* `cluster_label`
  * Name of cluster that is mapped to manager-apps directory. See above for supported names and dependencies to cluster apps directory.

* `shcluster_label`
  * Name of shcluster that is mapped to shcluster_label directory. See above for supported names and dependencies to shcluster apps directory.


### group_vars/all/onboarding
* `environment_name`
  * Extracted name from inventory path
* `absolute_file_store_path`
  * Base path to splunk directory where onboarding apps are stored
* `versioned_apps_sourcedir`
  * Maps directory to `apps/generic`.
* `selected_apps_sourcedir`
  * Maps environment specific directory for apps, uses `environment_name` variable to construct `apps/ENVIRONMENT_NAME`.
  * `selected_apps` can be added to any group vars file to install a Splunk app.
* `selected_apps_sourcedir`
  * Maps directory to `deployment-apps/ENVIRONMENT_NAME/selectable`
* `deployment_apps_sourcedir`
  * Maps specific directory for deployment-apps, uses `environment_name` variable to construct `deployment-apps/ENVIRONMENT_NAME`
* `shcluster_apps_sourcedir`
  * Maps directory for shcluster apps, uses `environment_name` variable to construct `shcluster/ENVIRONMENT_NAME/SHCLUSTER_LABEL/apps`
* `selected_shcluster_apps_sourcedir`
  * Maps specific directory for shcluster apps, uses `environment_name` variable to construct `shcluster/ENVIRONMENT_NAME/selectable`
* `manager_apps_sourcedir`
  * Maps directory for cluster apps, uses `environment_name` variable to construct `manager-apps/ENVIRONMENT_NAME/CLUSTER`
* `selected_manager_apps_sourcedir`
  * Maps specific directory for cluster apps, uses `environment_name` variable to construct `manager-apps/ENVIRONMENT_NAME/selectable`
* `deployment_apps_rsync_opts`
  * Default rsync options for deployment-apps
* `shcluster_apps_rsync_opts`
  * Default rsync options for shcluster apps
* `manager_apps_rsync_opts`
  * Default rsync options for manager-apps
* `selectable_apps_rsync_opts`
  * Default rsync options for apps
* `versioned_apps_rsync_opts`
  * Default rsync options for versioned splunkbase apps


### group_vars/all/defaults
* `cluster_manager_config_bundle_dir` is dynamically set based on Current Splunk Enterprise Version
### group_vars/INVENTORY_GROUP
* `forwarders`
  * Ansible inventory group name, apps and generic apps is in scope for standalone splunk servers. The destination for all apps are `SPLUNK_HOME/etc/apps`. The `INVENTORY_GROUP` can be towards any non-clustered splunk server.
  * Example config for an environment specific app, not versioned and specific to this organization.
    ```
    selected_apps:
      - name: 'innovationfleet-splunk_httpinput'
        source_app: 'innovationfleet-splunk_httpinput'
        state: 'present'
    ```

  * Example config for an environment specific app that should be deployed towards one target only.
    Multiple apps with the same name but different target hosts can be configured.
    ```
    selected_apps:
      - name: 'innovationfleet-splunk_httpinput'
        source_app: 'innovationfleet-splunk_httpinput'
        state: 'present'
        target: '<INVENTORY_HOST>'
    ```

  * Example config for an generic app, version 3.7.0 of Splunk DB Connect. Version info available as part of the app name. Supports easy upgrade/downgrade by updating the app version and re-deploy.
    ```
    versioned_apps:
      - name: 'splunk_app_db_connect'
        source_app: 'appFile-splunk_app_db_connect_v370'
        state: 'present'
        rsync_opts:
          - "--include=local/db_connection_types.conf"
          - "--include=local/dbx_settings.conf"
          - "--exclude=local/*"
          - "--exclude=secret/*"
          - "--exclude=certs/identity.dat"
          - "--exclude=keystore"
          - "--exclude=customized.java.path"
    ```

* `deployment_servers`
  * Target inventory group name, deployment-apps and apps are in scope for standalone splunk deployment servers.
  * Example config for an environment specific app that controls the serverclass.
    ```
    selected_apps:
      - name: 'innovationfleet_serverclass_conf'
        source_app: 'innovationfleet_serverclass_conf'
        state: 'present'
    ```
  * The destination directory for all deployment_apps are `SPLUNK_HOME/etc/deployment-apps`. All apps stored in `deployment_apps_sourcedir` on the manager will be deployed without need to specify individual apps.

  * Example config for an environment dependant deployment-server app, version 8.5.0 of Splunk TA for Windows. This one has custom inputs for production destination index names.
    ```
    selected_deployment_apps:
      - name: 'uf_output_conf'
        source_app: 'prod_innovationfleet_outputs_conf'
        state: 'present'
      - name: 'Splunk_TA_windows'
        source_app: 'prod-Splunk_TA_windows_v850'
        state: 'present'
    ```

* `searchhead_deployer_shcluster_c1`
  * Target inventory group name, generic apps, shcluster apps, selectable apps are in scope.
  * Example config for deploying a generic splunkbase app to a Search Head deployer. Here we need to specify the dest_dir as the app would end up in `etc/apps` instead of `shcluster/apps`
    ```
    versioned_apps:
      - name: 'Splunk_ML_Toolkit'
        source_app: 'appFile-Splunk_ML_Toolkit_v531'
        dest_dir: 'shcluster/apps'
        state: 'present'
    ```
  * Example config for an environment specific app, this one is used as landing page in Splunk UI.
    ```
    selected_shcluster_apps:
      - name: 'sh_landing_page'
        source_app: 'innovationfleet-sh_landing_page'
        state: 'present'
    ```
  * The destination directory for all shcluster apps are `SPLUNK_HOME/etc/shcluster/apps`. All apps stored in `shcluster_apps_sourcedir` on the manager will be deployed without need to specify individual apps.

* `cluster_manager_cluster_c1`
  * Target inventory group name with manager apps and selectable apps in scope.
  * Example config for an environment specific app, this one is used for parsing and event breaking events.
    ```
    selected_manager_apps:
      - name: 'innovationfleet_props_conf'
        source_app: 'innovationfleet_props_conf'
        state: 'present'
    ```
  * The destination directory for all manager apps are `SPLUNK_HOME/etc/manager-apps`. All apps stored in `manager_apps_sourcedir` on the manager will be deployed without need to specify individual apps.

### Configurable variables*
To protect apps from being deleted unintentionally from the directories related to Cluster Manager Configuration Bundles, Search Head Cluster Bundles or Deployment Server Apps, an `extra_vars` needs to be added to actively accept app removals.
If the `accept_*_removal` variable is left at default(false), the playbook will halt when an app removal is detected.
* `accept_manager_app_removal default(false)`
  * Apps for Cluster Peers are installed in `etc/manager-apps` on the Cluster Manager. If you would like to remove apps from the Cluster Manager Bundle directory, set
    ```
      {"accept_manager_app_removal":true}
    ```
    in the extra vars dialog in cca_ctrl UI.


* `accept_shc_app_removal default(false)`
  * Apps for Search Head Members are installed in `etc/shcluster` on the Search Head Deployer. If you would like to remove apps from the Search Head Bundle directory, set
    ```
      {"accept_shc_app_removal":true}
    ```
    in the extra vars dialog in cca_ctrl UI.

* `accept_deployment_app_removal default(false)`
  * Apps for Deployment Clients are installed in `etc/deployment-apps on the Deployment Server. If you would like to remove apps from the Deployment Server directory, set
    ```
      {"accept_deployment_app_removal":true}
    ```
    in the extra vars dialog in cca_ctrl UI.

* `force_bundle_push default(false)`
  * If you need to correct an error, the index clusters might be in a state where the pre-flight status is not ok. To skip the pre-flight check, set
    ```
      {"force_bundle_push":true}
    ```
    in the extra vars dialog in cca_ctrl UI.

Tasks
------------

### Main
* `stage_shcluster_apps.yml`
  * Collect all apps to a temporary directory on the manager server. Both generic, shcluster apps and environment specific shcluster apps.
* `deploy_shcluster_apps.yml`
  * Deploys all apps from the temporary directory towards the search head deployer using the Ansible synchronize module.
* `stage_manager_apps.yml`
  * Collect all apps to a temporary directory on the manager server. Both manager apps and environment specific manager apps.
* `deploy_manager_apps.yml`
  * Deploys all apps from the temporary directory towards the cluster manager using the Ansible synchronize module.
* `deploy_apps.yml`
  * Deploys apps, one by one towards destination servers.
* `stage_deployment_apps.yml`
  * Collect all apps to a temporary directory on the manager server. Both deployment apps and environment specific deployment apps.
* `deploy_deployment_apps.yml`
  * Deploys all apps from the temporary directory towards the deployment servers using the Ansible synchronize module.

### Standalone tasks
None

Files
------------

CCA onboarding repo holds a directory structure for all different apps. This pictures apps onboarded above in the `ONBOARDING_REPO/splunk/etc` directory.
```
.
├── apps
│   ├── selectable
│   │   ├── innovationfleet_serverclass_conf
│   │   └── innovationfleet-splunk_httpinput
│   └── versioned
│       ├── appFile-splunk_app_db_connect_v370
│       ├── appFile-Splunk_ML_Toolkit_v531
│       └── build_appFile_from_splunk_app.sh
├── deployment-apps
|   |── selectable
|   │   └── innovationfleet-prod_outputs_conf
│   └── ENVIRONMENT_NAME
│       ├── innovationfleet_sql_uf_inputs
│       └── innovationfleet_iis_uf_inputs
├── manager-apps
|   |── selectable
|   │   └── innovationfleet-props_conf
│   └── ENVIRONMENT_NAME
│       ├── cluster_c1 (must match cluster_label in inventory hosts file)
│       │   └── innovationfleet_indexes_conf
│       ├── cluster_c2
│       ├── cluster_c3
│       ├── cluster_c4
│       ├── cluster_c5
│       ├── cluster_c6
│       ├── cluster_c7
│       ├── cluster_c8
│       └── cluster_c9
└── shcluster
    |── selectable
    │   └── innovationfleet-sh_landing_page
    └── ENVIRONMENT_NAME
        ├── shcluster_c1 (must match shcluster_label in inventory hosts file)
        │   └── apps
        |       ├── sh_frontend_team
        │       └── sh_database_team
        ├── shcluster_c2
        ├── shcluster_c3
        ├── shcluster_c4
        ├── shcluster_c5
        ├── shcluster_c6
        ├── shcluster_c7
        ├── shcluster_c8
        └── shcluster_c9

```


Dependencies
------------

Dependent role name

License
-------

MIT