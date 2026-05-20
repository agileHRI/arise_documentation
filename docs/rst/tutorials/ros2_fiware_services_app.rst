.. _ros2_fiware_services_app:

Calling ROS 2 Services from FIWARE Context Broker
=================================================

Introduction
------------

This tutorial demonstrates how to call a ROS 2 service from the FIWARE Context Broker using Docker containers.
In this example, Orion-LD acts as a client for a ROS 2 service, while a ROS 2 service server provides the
``add_two_ints`` service.

By the end of this tutorial, you will be able to:

- Launch the FIWARE Context Broker with a DDS-to-NGSI-LD service mapping.
- Start a ROS 2 service server that adds two integers.
- Trigger a ROS 2 service call from the Context Broker through the NGSI-LD REST API.
- Retrieve the service response from the Context Broker.

We will use Docker Compose to run Orion-LD and MongoDB, and a Vulcanexus Docker container to run the ROS 2
service server.

.. note::

    This tutorial replicates the approach of the :ref:`ros2_fiware_basic_app` tutorial, it is recommended to
    review that tutorial first.

Prerequisites
-------------

- Docker installed on your system.
- Docker Compose installed on your system.

Running the FIWARE Context Broker
---------------------------------

The Context Broker requires a configuration file to establish a mapping between ROS 2 services and NGSI-LD
entity attributes.

In this tutorial, the ``add_two_ints`` ROS 2 service is mapped to the ``add_two_ints`` attribute of the
``urn:ngsi-ld:robot:1`` NGSI-LD entity. Orion-LD will use this mapping to act as a client of the ROS 2
service.

Create a file named ``context_broker_srv_config.json`` with the following content:

.. code-block:: json

    {
      "dds": {
        "ddsmodule": {
          "dds": {
            "domain": 0,
            "transport": "udp"
          }
        },
        "ngsild": {
          "services": {
            "add_two_ints": {
              "entityType": "Robot",
              "entityId": "urn:ngsi-ld:robot:1",
              "attribute": "add_two_ints"
            }
          }
        }
      }
    }

Here's a breakdown of the configuration file structure:

- ``dds``: This section contains the configuration for the DDS transport layer.

  - ``domain``: The DDS domain ID used by both Orion-LD and the ROS 2 service server.
  - ``transport``: The DDS transport used for communication. In this tutorial, UDP is used.

- ``ngsild``: This section defines the mapping between ROS 2 services and NGSI-LD entities.

  - ``services``: This section contains the ROS 2 services exposed through Orion-LD.
  - ``add_two_ints``: The name of the ROS 2 service that Orion-LD will call.
  - ``entityType``: The type of the NGSI-LD entity associated with the service.
  - ``entityId``: The unique identifier of the NGSI-LD entity associated with the service.
  - ``attribute``: The NGSI-LD attribute used to represent the service request and response.

This configuration ensures that requests written to the ``add_two_ints`` attribute of the
``urn:ngsi-ld:robot:1`` entity are translated into ROS 2 service calls to ``/add_two_ints``.

To run the FIWARE Context Broker, create a ``docker-compose-srv.yml`` file with the following content:

.. code-block:: yaml

    services:

      mongodb:
        image: mongo:7.0
        privileged: true
        ipc: host
        network_mode: host
        command: --bind_ip_all
        volumes:
          - mongo_data:/data/db

      orion:
        image: fiware/orion-ld:1.13.0-PRE-1835
        privileged: true
        ipc: host
        network_mode: host
        depends_on:
          - mongodb
        restart: always
        command: -dbhost localhost -wip dds -mongocOnly
        environment:
          - ORIONLD_MONGO_HOST=localhost
        volumes:
          - ./context_broker_srv_config.json:/root/.orionld
        healthcheck:
          test: curl --fail -s http://localhost:1026/version || exit 1
          interval: 30s
          retries: 15

    volumes:
      mongo_data:

This configuration starts MongoDB and FIWARE Context Broker. MongoDB is used by Orion-LD to store the current state of
the Context Broker.

Start the services with the following command:

.. code-block:: bash

    docker compose -f docker-compose-srv.yml up -d

.. note::

    This tutorial uses Docker Compose v2, which is integrated with the Docker CLI and uses the
    ``docker compose`` command.

    If that command does not work in your environment, try the older command instead:

    .. code-block:: bash

        docker-compose -f docker-compose-srv.yml up -d

This command downloads the required images and starts the containers in detached mode.

.. note::

    If something goes wrong during the setup, run the previous command without detached mode to inspect the logs:

    .. code-block:: bash

        docker compose -f docker-compose-srv.yml up

.. note::

    To stop and remove the running containers, use the following command:

    .. code-block:: bash

        docker compose -f docker-compose-srv.yml down

Running the ROS 2 Service Server
--------------------------------

To run the ROS 2 service server, use a Vulcanexus Docker container.

Pull the Vulcanexus image with the following command:

.. code-block:: bash

    docker pull eprosima/vulcanexus:jazzy-desktop

Create and run a new container with the following command:

.. code-block:: bash

    docker run -it --rm --net=host --ipc=host --privileged \
      -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix \
      eprosima/vulcanexus:jazzy-desktop

This command will run the Vulcanexus container and connect it to the host network, allowing it to communicate with the FIWARE Context Broker.

Inside the Vulcanexus container, run the ``add_two_ints`` service server provided by the
``demo_nodes_cpp`` package:

.. code-block:: bash

    ros2 run demo_nodes_cpp add_two_ints_server

The service server will wait for requests on the ``/add_two_ints`` service. Each request contains two
integers, ``a`` and ``b``. The response contains their sum.

You can verify that the service is available from another terminal inside the same container with:

.. code-block:: bash

    ros2 service list

You should see the following service in the output:

.. code-block:: bash

    /add_two_ints

You can also inspect the service type with:

.. code-block:: bash

    ros2 service type /add_two_ints

The expected type is:

.. code-block:: bash

    example_interfaces/srv/AddTwoInts

Calling the ROS 2 Service from the Context Broker
-------------------------------------------------

At this stage, Orion-LD is running with a service mapping and the ROS 2 service server is waiting for
requests.

To call the service, update the ``add_two_ints`` attribute of the ``urn:ngsi-ld:robot:1`` entity through
the NGSI-LD REST API.

To do so, send a request to the ROS 2 service by patching the ``add_two_ints`` attribute with the two integers to
add:

.. code-block:: bash

    payload='{"value":{"a":10,"b":32}}'

    curl http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1/attrs/add_two_ints \
      -X PATCH \
      -d "$payload" \
      -H 'Content-Type: application/json'

In this example:

- ``a`` is set to ``10``.
- ``b`` is set to ``32``.
- Orion-LD uses the configured ``services`` mapping to call the ``/add_two_ints`` ROS 2 service.
- The ROS 2 service server receives the request and computes the sum.

The service server terminal should show that it received a request similar to:

.. code-block:: text

    Incoming request
    a: 10 b: 32

Querying the Service Response
-----------------------------

After the service call has been processed, query the NGSI-LD entity to inspect the current value of the
``add_two_ints`` attribute:

.. code-block:: bash

    curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?prettyPrint=yes&local=true" \
      -s -S \
      -H 'Accept: application/json' | jq

The response should contain the ``add_two_ints`` attribute.
The attribute contains several fields:

.. code-block:: text

    {
      "id": "urn:ngsi-ld:robot:1",
      "type": "Robot",

      "add_two_ints": {
        "type": "Property",
        "value": {
          "a": 10,
          "b": 32
        },
        "request": {...},
        "reply": {...}
      },
      "ddsType": {
        "type": "Property",
        "value": "fastdds"
      }
    }

The ``value`` field contains the service request with the two integers to add.
The ``request`` and ``reply`` sub-attributes contain several metadata fields related to the DDS entities.
We are interested in the value field of the ``reply`` attribute, which contains the service response with the computed sum.

For this example, the expected service result is:

.. code-block:: text

    "reply": {
      "type": "Property",
      "value": {
        "sum": 42
      },


You can continuously monitor the entity while sending service requests with:

.. code-block:: bash

    while true; do
        curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?prettyPrint=yes&local=true" \
          -s -S \
          -H 'Accept: application/json' | jq
        sleep 1
    done

Troubleshooting
---------------

If the service call does not reach the ROS 2 service server, check the following points:

- Make sure the ``add_two_ints`` service server is running.
- Make sure Orion-LD and the ROS 2 container are using the same DDS domain.
- Make sure both containers are using ``--net=host`` or ``network_mode: host``.
- Make sure the service name in the configuration file is ``add_two_ints``.

- Check that ROS 2 can see the service with:

  .. code-block:: bash

      ros2 service list
      ros2 service type /add_two_ints

Conclusion
----------

In this tutorial, we demonstrated how to call a ROS 2 service from the FIWARE Context Broker.
Orion-LD was configured as a ROS 2 service client through a DDS-to-NGSI-LD service mapping, and a ROS 2
``add_two_ints`` service server was launched in a Vulcanexus container.

By following these steps, you were able to:

- Configure Orion-LD with an NGSI-LD mapping for a ROS 2 service.
- Launch a ROS 2 service server.
- Send service requests through the NGSI-LD REST API.
- Observe the service result through the Context Broker.

You can extend this example by adding more ROS 2 services to the ``services`` section of the
``context_broker_srv_config.json`` file and mapping each one to a different NGSI-LD entity attribute.
