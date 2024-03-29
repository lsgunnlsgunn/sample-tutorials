---
layout: layout.pug
navigationTitle: Basic configuration settings
title: Basic configuration settings
menuWeight: 20
excerpt: Provides examples for the most common Edge-LB configuration settings
enterprise: true
---
This section provides examples that illustrate how to set some of the most basic Edge-LB pool configuration options.

# Before you begin
Before you create Edge-LB pools and pool configuration files, you should have DC/OS Enterprise cluster nodes installed and ready to use and have previously downloaded and installed the latest Edge-LB packages.

* You must have Edge-LB installed as described in the Edge-LB [installation instructions](/services/edge-lb/getting-started/installing).
* You must have the core DC/OS command-line interface (CLI) installed and configured to communicate with the DC/OS cluster.
* You must have the `edgelb` command-line interface (CLI) installed.
* You must have an active and properly-configured DC/OS Enterprise cluster.
* The DC/OS Enterprise cluster must have at least one DC/OS **private agent** node to run the load-balanced service and at least one DC/OS **public agent** node for exposing the load-balanced service.

For information about installing Edge-LB packages, see the [installation](/services/edge-lb/getting-started/installing/) instructions.

# Using Edge-LB for a sample Marathon application
DC/OS services typically run as applications on the Marathon framework. To create a pool configuration file for a Marathon application, you need to know the Mesos `task` name and `port` name. With this information, you can then configure an Edge-LB pool to handle load balancing for that Marathon appllication.

## Identify the task and port for an app
The following code snippet is an excerpt from a sample Marathon app definition. For this sample app definition:
* the `task` name is `my-app`
* the `port` name is `web`

```json
{
  "id": "/my-app",
  ...
  "portDefinitions": [
    {
      "name": "web",
      "protocol": "tcp",
      "port": 0
    }
  ]
}
```

## Configure load balancing for the sample app
After you identify a Marathon application task name and port, you can create a pool configuration for load balancing that app definition. 

1. Create an Edge-LB pool configuration file and name it `app-lb.json`.

    The following code provides a simple example of how to configure an Edge-LB pool named `app-lb` with a pool configuration file named `app-lb.json` to do load balancing for the sample Marathon application:

    ```json
    {
      "apiVersion": "V2",
      "name": "app-lb",
      "count": 1,
      "haproxy": {
        "frontends": [{
          "bindPort": 80,
          "protocol": "HTTP",
          "linkBackend": {
            "defaultBackend": "app-backend"
          }
        }],
        "backends": [{
          "name": "app-backend",
          "protocol": "HTTP",
          "services": [{
            "marathon": {
              "serviceID": "/my-app"
            },
            "endpoint": {
              "portName": "web"
            }
          }]
        }]
      }
    }
    ```

1. Upload the pool configuration to Edge-LB by running the following command:

    ```bash
    dcos edgelb create app-lb.json
    ```

1. Create a Marathon application definition containing the service and name it `my-app.json`. For example:

    ```json
    {
        "id": "/my-app",
        "cmd": "/start $PORT0",
        "instances": 1,
        "cpus": 0.1,
        "mem": 32,
        "container": {
            "type": "DOCKER",
            "docker": {
                "image": "mesosphere/httpd"
            }
        },
        "portDefinitions": [
            {
                "name": "web",
                "protocol": "tcp",
                "port": 0
            }
        ],
        "healthChecks": [
            {
                "portIndex": 0,
                "path": "/",
                "protocol": "HTTP"
            }
        ]
    }
    ```

1. Deploy the service by running the following command:

    ```bash
    dcos marathon app add my-app.json
    ```

# Using static DNS and virtual IP addresses

Internal addresses, such as those generated by Mesos-DNS, Spartan, or the DC/OS layer-4 load-balancer (`dcos-l4lb`) can be exposed outside of the cluster with Edge-LB by using `pool.haproxy.backend.service.endpoint.type: "ADDRESS"`.

Keep in mind that exposing secured internal services to the outside world using an insecure endpoint can make your cluster vulnerable to attack. To mitigate the risk involved when using this feature, you should ensure access to internal endopoints is strictly regulated and monitored.

The following code sample illustrates the Edge-LB pool configuration for a static IP address resolved by Mesos-DNS.

```json
{
  "apiVersion": "V2",
  "name": "dns-lb",
  "count": 1,
  "haproxy": {
    "frontends": [{
      "bindPort": 80,
      "protocol": "HTTP",
      "linkBackend": {
        "defaultBackend": "app-backend"
      }
    }],
    "backends": [{
      "name": "app-backend",
      "protocol": "HTTP",
      "services": [{
        "endpoint": {
          "type": "ADDRESS",
          "address": "myapp.marathon.l4lb.thisdcos.directory",
          "port": 555
        }
      }]
    }]
  }
}
```

# Using Edge-LB with other frameworks and data services

You can use Edge-LB load balancing for frameworks and data services that run tasks not managed by Marathon. For example, you might have tasks managed by Kafka brokers or Cassandra. For tasks that run under other frameworks and data services, you can use the `pool.haproxy.backend.service.mesos` object to filter and select tasks for load balancing.

```json
{
  "apiVersion": "V2",
  "name": "services-lb",
  "count": 1,
  "haproxy": {
    "frontends": [{
      "bindPort": 1025,
      "protocol": "TCP",
      "linkBackend": {
        "defaultBackend": "kafka-backend"
      }
    }],
    "backends": [{
      "name": "kafka-backend",
      "protocol": "TCP",
      "services": [{
        "mesos": {
          "frameworkName": "beta-confluent-kafka",
          "taskNamePattern": "^broker-*$"
        },
        "endpoint": {
          "port": 1025
        }
      }]
    }]
  }
}
```

Other useful fields for selecting frameworks and tasks in `pool.haproxy.backend.service.mesos`:

* `frameworkName`: Exact match
* `frameworkNamePattern`: Regular expression
* `frameworkID`: Exact match
* `frameworkIDPattern`: Regular expression
* `taskName`: Exact match
* `taskNamePattern`: Regular expression
* `taskID`: Exact match
* `taskIDPattern`: Regular expression

# Using virtual networks

The following example creates a pool that will be launched on the virtual network provided by DC/OS overlay called "dcos". In general, you can launch a pool on any CNI network, by setting `pool.virtualNetworks[].name` to the CNI network name.

```json
{
  "apiVersion": "V2",
  "name": "vnet-lb",
  "count": 1,
  "virtualNetworks": [
    {
      "name": "dcos",
      "labels": {
        "key0": "value0",
        "key1": "value1"
      }
    }
  ],
  "haproxy": {
    "frontends": [{
      "bindPort": 80,
      "protocol": "HTTP",
      "linkBackend": {
        "defaultBackend": "vnet-be"
      }
    }],
    "backends": [{
      "name": "vnet-be",
      "protocol": "HTTP",
      "services": [{
        "marathon": {
          "serviceID": "/my-vnet-app"
        },
        "endpoint": {
          "portName": "my-vnet-port"
        }
      }]
    }]
  }
}
```
