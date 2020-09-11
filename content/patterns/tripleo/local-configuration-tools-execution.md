+++
title = "Local Configuration Tools"
date = "2020-09-02"
author = "Danil Zhigalin"
tags = ["heat", "troubleshooting", "openstack"]
weight = 3
+++

## Where local tools get their data

So far we were talking about heat SoftwareDeployment resources without trying to find out what is happening on the host when those resources are created. While on the heat side everything may look shiny, local executions may encounter problems and we are usually presented some obscure error message that is given to us by heat. Without knowing what work certain heat resource is asking to be done on the host side it's difficult to figure out what caused an error. To find the cause we'll need to log into the server, where resource was applying the configuration and check the logs or even try to execute the script manually to inspect it more closely.

But first of all we need to know, how configuration tools are launched on the host. We've already mentioned in "Software configuration done by Heat" that in SoftwareConfig terminology, configuration tools executed locally on the hosts are called hooks. When SoftwareDeployment is created, it just publishes the script along with the hook that will need to execute via heat API. On the node there is a special service os-collect-config, that is polling some URL for any active Deployments that exist for the host, on which it is running. Depending on the metadata transport method set in OS::Nova::Server properties, os-collect-config is getting configuration scripts from one of the following places: 

* HEAT CFN endpoint 
* HEAT endpoint
* TMP URL, which is created to point to Swift container
* Zaquar queue

## How to see config data in Swift

Most of the time we will see that polling is happening with POLL_TEMP_URL, which utilizes Swift of the undercloud. We can make sure what is used as SoftwareConfigTransport by checking overcloud-resource-registry-puppet.yaml

To find out the URL that is used to get server metadata, we need to print the server resource:



```
# we already know that OS::Nova::Server resources are created within this ResourceGroup
openstack stack resource list overcloud | grep "ContrailDpdk "
| ContrailDpdk                                       | 8d40e39e-7e5b-4151-a91c-0033e6e36a3e                               | OS::Heat::ResourceGroup                          | UPDATE_COMPLETE | 2020-07-20T15:34:51Z |
```

Each resource inside resource grou has its sequence number as resource_name

{{< cmd >}}
openstack stack resource list 8d40e39e-7e5b-4151-a91c-0033e6e36a3e
{{< /cmd >}}

```
+---------------+--------------------------------------+---------------------------+-----------------+----------------------+
| resource_name | physical_resource_id                 | resource_type             | resource_status | updated_time         |
+---------------+--------------------------------------+---------------------------+-----------------+----------------------+
| 0             | a2ba92f7-e974-4426-9271-1680a8dce217 | OS::TripleO::ContrailDpdk | UPDATE_COMPLETE | 2020-07-20T15:34:58Z |
+---------------+--------------------------------------+---------------------------+-----------------+----------------------+
```

Metadata_url is one of the attributes of a server. It has different format based on the transport method chosed.
{{< cmd >}}
openstack stack resource show  8d40e39e-7e5b-4151-a91c-0033e6e36a3e 0 -c attributes | grep -o "metadata_url.*\},"
{{< /cmd >}}

```
metadata_url': u'http://172.18.2.1:8080/v1/AUTH_d5abf72d8fdd463d83376b98442077c2/ov-vxmx2tutr-0-gevtog3pjbb5-ContrailDpdk-nc42pbz5sqp4/b22306ee-73d0-4c09-a4d1-3059e89dfcfe?temp_url_sig=b6d73133471a6c848cfb654677e2595534ba01a9&temp_url_expires=2147483586'},
```

We can retrieve the information from temp URL with a simple curl command:

{{< cmd >}}
curl "http://172.18.2.1:8080/v1/AUTH_d5abf72d8fdd463d83376b98442077c2/ov-vxmx2tutr-0-gevtog3pjbb5-ContrailDpdk-nc42pbz5sqp4/b22306ee-73d0-4c09-a4d1-3059e89dfcfe?temp_url_sig=b6d73133471a6c848cfb654677e2595534ba01a9&temp_url_expires=2147483586"
{{< /cmd >}}

And here is the container, that stores all metadata for software deployments ov-vxmx2tutr-0-gevtog3pjbb5-ContrailDpdk-nc42pbz5sqp4. With this method of transport in use, similar containers are created in Swift:

{{< cmd >}}
openstack container list
{{< /cmd >}}

{{< code numbered="true" >}}
+-------------------------------------------------------+
| Name                                                  |
+-------------------------------------------------------+
| __cache__                                             |
| ov-2b6bcw7v-1-w4jiutykfifa-ContrailSriov-jm3eykjictdb |
| ov-5br-0-zvqudfi6joh6-ContrailController-6wlfakopxk5s |
| ov-5br-1-fouz4n2j4mg2-ContrailController-ddegfgdeexln |
| ov-5br-2-ek4ynmp3gtvo-ContrailController-eejkljgdzgap |
| ov-7jx2uxrp-1-gwzdwjwu32gf-ContrailDpdk1-yf467k72uimf |
| ov-czb3yu5vqmw-0-q5zj3g7sassj-Controller-yznioytzscth |
| ov-czb3yu5vqmw-1-5ci3euwxkdjr-Controller-erczumytp23y |
| ov-czb3yu5vqmw-2-xe3rxkuieanx-Controller-ynrqvfi3bv6v |
| ov-esio5zp2rg-0-2tpb2dw2dsix-CephStorage-lrkkrxrvzckh |
| ov-esio5zp2rg-1-4swvxjpr37uy-CephStorage-4uedtasootdd |
| ov-esio5zp2rg-2-ztka4url5lb7-CephStorage-w7fxf5kk7sa6 |
| ov-ptlmqcr4v4-0-b53urv2cfjje-NovaCompute-af3z53nwvqsb |
| [[[ov-vxmx2tutr-0-gevtog3pjbb5-ContrailDpdk-nc42pbz5sqp4]]] |
| ov-xavchfb-1-updocnngix7s-ContrailSriov1-2bd5lwl2qzzu |
| ov-y5o7lcof3h-0-h3zqwfcivan3-NovaCompute-otskkmy3zxgp |
| overcloud                                             |
| overcloud-messages                                    |
| overcloud-swift-rings                                 |
| overcloud_ceph_ansible_fetch_dir                      |
| overcloud_ceph_ansible_fetch_dir_segments             |
| plan-exports                                          |
| plan-exports_segments                                 |
+-------------------------------------------------------+
{{< /code >}}

1. here is the container from the previous example

No matter how we retrieve information - with curl to the temp URL or by downloading container from swift:

{{< cmd >}}
openstack container save ov-vxmx2tutr-0-gevtog3pjbb5-ContrailDpdk-nc42pbz5sqp4
{{< /cmd >}}

## Structure of configuration data

Inside it we will see data that all deployments will use locally on the nodes. Structure is:

```
{
  "deployments": [
    {
      "id": "id of the deployment, will be found in /var/lib/heat-config/deployed and /var/lib/heat-config/heat-config",
      "config": "configuration script contents, like ansible playbook or bash script; should be understood by config tool specified in the group"
      "options": {},
      "creation_time": "2020-06-22T09:49:51Z",
      "outputs": [
        {
          "description": "",
          "error_output": false,
          "name": "result",
          "type": "String"
        }
      ],
      "name": "resource name of this software deployment, as seein in templates like ContrailNodeInitDeployment",
      "group": "configuration tool used to execute the configuration",
      "inputs": [
      ... list of inputs into this software deployment ...
      ]
    },
    {
    ... next deployment ...
    }
  ]
}
```

So all deployments for each role are present in that Swift container. Knowing this, we can get them from the saved container all at once using jq:

```
cat b22306ee-73d0-4c09-a4d1-3059e89dfcfe | jq '.[] | .[]["name"]' -r
ContrailDpdkAllNodesDeployment
ContrailDpdkAllNodesValidationDeployment
ContrailDpdkArtifactsDeploy
ContrailDpdkDeployment
ContrailDpdkDeployment_Step1
ContrailDpdkDeployment_Step2
ContrailDpdkDeployment_Step3
ContrailDpdkDeployment_Step4
ContrailDpdkDeployment_Step5
ContrailDpdkHostPrepDeployment
ContrailDpdkHostsDeployment
ContrailDpdkSshKnownHostsDeployment
ContrailNodeInitDeployment
DpdkContainerParametersDeployment
DpdkHostParametersDeployment
DpdkKernelParametersDeployment
NetworkDeployment
RebootDeployment
RebootEnsureDeployment
SshHostPubKeyDeployment
UpdateDeployment
```

And now having their names we can find their resource and location of the configuration script that those deployments implement.

## What happens with configuration data on the node

All such scripts are copied to the node into respective hook directories of /var/lib/heat-config directory and executed locally by heat-config utility. For ansible, for example, it's running something like this:


{{< note >}}
That ansible-playbook is started directly on the host, to which configuration is being applied. This is different from the normal way of working with ansible, when hosts are controlled by ansible, but don't need to have ansible installed locally. Here f37357c4-53c3-4fd4-9c1a-09b840af3266 is the ID of software deploymeht, that can be found in swift contianer.
{{< /note >}}

{{< cmd >}}
ansible-playbook -i localhost, --module-path /usr/share/ansible-modules /var/lib/heat-config/heat-config-ansible/f37357c4-53c3-4fd4-9c1a-09b840af3266_playbook.yaml --extra-vars @/var/lib/heat-config/heat-config-ansible/f37357c4-53c3-4fd4-9c1a-09b840af3266_variables.json
{{< /cmd >}}

With this knowledge, when we see an error like this during deployment, we already know where to look further:

```
overcloud.AllNodesDeploySteps.ControllerDeployment_Step4.0:
  resource_type: OS::Heat::StructuredDeployment
  physical_resource_id: e21edc2b-a7db-4ee5-b40c-9cb1ecb32952
  status: CREATE_FAILED
  status_reason: |
    Error: resources[0]: Deployment to server failed: deploy_status_code : Deployment exited with non-zero status code: 2
  deploy_stdout: |
    ...
            "2fcde88233d7: Pull complete", 
            "Digest: sha256:ce467ee8692a2f6faae901233c3bee853773c56ff020ace7904d644bebf1d43b", 
            "Status: Downloaded newer image for 172.18.2.1:8787/rhosp13/openstack-swift-container:13.0-99.1582098609"
        ]
    }
     to retry, use: --limit @/var/lib/heat-config/heat-config-ansible/c76e80d8-1a78-4be1-966a-654d4542dfde_playbook.retry
    
    PLAY RECAP *****************************************************************
    localhost                  : ok=12   changed=8    unreachable=0    failed=1   
    
    (truncated, view all with --long)
  deploy_stderr: |
```

It's worth having a look at ansible playbook that is implementing the Step[1-5] deployments. For all those steps same playbook is used, which is in tripleo-heat-templates/common/deploy-steps-tasks.yaml. Some of the tasks are there due to legacy reasons. During the evolution of TripleO ways of configuring servers were changing constantly and some tasks are making sure that backward compatibility is maintained. 

Notable task is "Run puppet host configuration for step {{step}}". This is starting puppet from ansible. All the configuration that is still done by puppet is done here. Manifest that puppet is executing is located on each node in /var/lib/tripleo-config/puppet_step_config.pp. Even though many services were moved to containers, there are still tasks that need to be done on the system directly and traditionally they were handled by puppet. Among those tasks are: ntp configuration, iptables rules and so one. For each of those tasks we will see includes like `include ::tripleo::profile::base::time::ntp`, `include ::tripleo::firewall`. They are referring to manifests in `/usr/share/openstack-puppet/modules/tripleo/manifests`. 

* ::tripleo::profile::base::time::ntp - Higher level manifest is profile/base/time/ntp.pp. Implementation of functions of the higher level manifest is in profile/base/time/ntp/
* ::tripleo::firewall - Higher level manifest is firewall.pp. Implementation of functions of the higher level manifest is in firewall/

Understanding of puppet and ruby is needed to make full sence of those manifests, but you can get an idea of what they are doing by just reading the files.

## Starting containers with services

Another important task is "Start containers for step {{step}}". In this step paunch tool is used to start containers. This tool is somewhat analagous to docker-compose. It reads files  /var/lib/tripleo-config/hashed-docker-container-startup-config-step_{{step}}.json, which contain list of containers and all their parameters - environment variables, volumes, entrypoints, just everything you would see in a docker-compose.yaml. You can also use paunch to get the list of running containers that it controls:


```
paunch list
+---------------+------------------------------------+---------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+
| config        | container                          | image                                                                           | command                                                                                                                                                                                                                                                                                                                                                      | status  |
+---------------+------------------------------------+---------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+
| tripleo_step1 | mysql_bootstrap                    | 172.18.2.1:8787/rhosp13/openstack-mariadb:13.0-101.1582098557                   | bash -ec if [ -e /var/lib/mysql/mysql ]; then exit 0; fi                                                                                                                                                                                                                                                                                                     | exited  |
|               |                                    |                                                                                 | echo -e "\n[mysqld]\nwsrep_provider=none" >> /etc/my.cnf                                                                                                                                                                                                                                                                                                     |         |
|               |                                    |                                                                                 | kolla_set_configs                                                                                                                                                                                                                                                                                                                                            |         |
|               |                                    |                                                                                 | sudo -u mysql -E kolla_extend_start                                                                                                                                                                                                                                                                                                                          |         |
|               |                                    |                                                                                 | mysqld_safe --skip-networking --wsrep-on=OFF &                                                                                                                                                                                                                                                                                                               |         |
|               |                                    |                                                                                 | timeout ${DB_MAX_TIMEOUT} /bin/bash -c 'until mysqladmin -uroot -p"${DB_ROOT_PASSWORD}" ping 2>/dev/null; do sleep 1; done'                                                                                                                                                                                                                                  |         |
|               |                                    |                                                                                 | mysql -uroot -p"${DB_ROOT_PASSWORD}" -e "CREATE USER 'clustercheck'@'localhost' IDENTIFIED BY '${DB_CLUSTERCHECK_PASSWORD}';"                                                                                                                                                                                                                                |         |
|               |                                    |                                                                                 | mysql -uroot -p"${DB_ROOT_PASSWORD}" -e "GRANT PROCESS ON *.* TO 'clustercheck'@'localhost' WITH GRANT OPTION;"                                                                                                                                                                                                                                              |         |
|               |                                    |                                                                                 | timeout ${DB_MAX_TIMEOUT} mysqladmin -uroot -p"${DB_ROOT_PASSWORD}" shutdown                                                                                                                                                                                                                                                                                 |         |
...
| tripleo_step1 | rabbitmq_bootstrap                 | 172.18.2.1:8787/rhosp13/openstack-rabbitmq:13.0-102.1582098564                  | kolla_start                                                                                                                                                                                                                                                                                                                                                  | exited  |
| tripleo_step3 | horizon                            | 172.18.2.1:8787/rhosp13/openstack-horizon:13.0-100.1582098580                   | kolla_start                                                                                                                                                                                                                                                                                                                                                  | running |
| tripleo_step3 | keystone_cron                      | 172.18.2.1:8787/rhosp13/openstack-keystone:13.0-96.1582101268                   | /bin/bash -c /usr/local/bin/kolla_set_configs && /usr/sbin/crond -n                                                                                                                                                                                                                                                                                          | running |
...
| tripleo_step2 | haproxy_init_bundle                | 172.18.2.1:8787/rhosp13/openstack-haproxy:13.0-103.1582098553                   | /docker_puppet_apply.sh 2 file,file_line,concat,augeas,tripleo::firewall::rule,pacemaker::resource::bundle,pacemaker::property,pacemaker::resource::ip,pacemaker::resource::ocf,pacemaker::constraint::order,pacemaker::constraint::colocation include ::tripleo::profile::base::pacemaker; include ::tripleo::profile::pacemaker::haproxy_bundle            | exited  |
| tripleo_step5 | cinder_volume_init_bundle          | 172.18.2.1:8787/rhosp13/openstack-cinder-volume:13.0-103.1582098606             | /docker_puppet_apply.sh 5 file,file_line,concat,augeas,pacemaker::resource::bundle,pacemaker::property,pacemaker::constraint::location include ::tripleo::profile::base::pacemaker;include ::tripleo::profile::pacemaker::cinder::volume_bundle                                                                                                              | exited  |
| tripleo_step5 | ceilometer_gnocchi_upgrade         | 172.18.2.1:8787/rhosp13/openstack-ceilometer-central:13.0-95.1582098612         | /usr/bin/bootstrap_host_exec ceilometer_agent_central su ceilometer -s /bin/bash -c 'for n in {1..10}; do /usr/bin/ceilometer-upgrade && exit 0 || sleep 30; done; exit 1'                                                                                                                                                                                   | exited  |
...
| tripleo_step4 | heat_engine                        | 172.18.2.1:8787/rhosp13/openstack-heat-engine:13.0-95.1582098592                | kolla_start                                                                                                                                                                                                                                                                                                                                                  | running |
...
+---------------+------------------------------------+---------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+
```

Here we see all containers that are managed by paunch on the local node, at which step they were started, what image and entrypoint they are using.

{{< note >}}
Due to some long-standing bug paunch is crashing wile displaying the list of containers. To work around that problem I'm using this method. Replace the line in /usr/lib/python2.7/site-packages/paunch/cmd.py as is shown below
{{< /note >}}

```
diff --git a/paunch/cmd.py b/paunch/cmd.py
index d069230..4f7e363 100644
--- a/paunch/cmd.py
+++ b/paunch/cmd.py
@@ -288,7 +288,7 @@ class List(lister.Lister):
         for k, v in configs.items():
             for i in v:
                 name = i.get('Name', '/')[1:]  # strip the leading slash
-                cmd = ' '.join(i.get('Config', {}).get('Cmd', []))
+                cmd =  ' '.join(i.get('Config', {}).get('Cmd', [])) if i.get('Config', {}).get('Cmd', []) is not None else ' '
                 image = i.get('Config', {}).get('Image')
                 status = i.get('State', {}).get('Status')
                 data.append((k, name, image, cmd, status))
```

But paunch is not the only tool managing containers in TripleO. Some containers are managed as pacemaker resources. To find out how exactly those resources are created we need to inspect puppet manifests of pacemaker, same way as it was shown in the example above. We can see that containers are running on Controller node by just running docker ps command.

```
docker ps | grep pcmklatest
aa110c3c2785        172.18.2.1:8787/rhosp13/openstack-rabbitmq:pcmklatest                             "dumb-init --singl..."   7 weeks ago         Up 7 weeks                                 rabbitmq-bundle-docker-2
9a756ed7f15d        172.18.2.1:8787/rhosp13/openstack-haproxy:pcmklatest                              "dumb-init --singl..."   8 weeks ago         Up 8 weeks                                 haproxy-bundle-docker-2
0fede0687214        172.18.2.1:8787/rhosp13/openstack-redis:pcmklatest                                "dumb-init --singl..."   8 weeks ago         Up 8 weeks                                 redis-bundle-docker-2
1020a909028b        172.18.2.1:8787/rhosp13/openstack-mariadb:pcmklatest                              "dumb-init -- /bin..."   8 weeks ago         Up 8 weeks                                 galera-bundle-docker-2
```

Configuration scripts also start containers in one-off fashion to do some required preparations. This is often done by contrail init containers. Those containers are usually just started by ansible playbooks that are launched by contrail software deployment resources. An example of such deployment is ContrailNodeInitDeployment.
