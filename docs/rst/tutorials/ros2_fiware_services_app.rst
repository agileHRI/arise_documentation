.. _ros2_fiware_services_app:

Calling ROS 2 Services from the coraine Context Broker
======================================================

Introduction
------------

This tutorial shows how to call a ROS 2 service from the ARISE context broker, coraine, using Docker containers.
In this example, coraine acts as a client of a ROS 2 service, and a ROS 2 service server provides the
``add_two_ints`` service.

By the end of this tutorial, you will be able to:

- Launch the context broker with a DDS-to-NGSI-LD service mapping.
- Start a ROS 2 service server that adds two integers.
- Call the ROS 2 service from the context broker through the NGSI-LD REST API.
- Read the service response from the context broker.

We will use Docker Compose to run coraine, and a Vulcanexus Docker container to run the ROS 2
service server.

.. note::

    This tutorial follows the same approach as :ref:`ros2_fiware_basic_app`. We recommend that you complete that tutorial first.

Prerequisites
-------------

- Docker installed on your system (Linux, ``amd64``).
- Docker Compose v2 installed on your system.
- ``curl`` and ``jq`` installed on your host.

Running the Context Broker
--------------------------

The context broker needs a configuration file that maps ROS 2 services to NGSI-LD entity attributes.

In this tutorial, the ``add_two_ints`` ROS 2 service is mapped to the ``add_two_ints`` attribute of the
``urn:ngsi-ld:robot:1`` NGSI-LD entity. coraine uses this mapping to act as a client of the ROS 2
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
              "attribute": "add_two_ints",
              "requestType": "example_interfaces::srv::dds_::AddTwoInts_Request_",
              "replyType": "example_interfaces::srv::dds_::AddTwoInts_Response_"
            }
          }
        }
      }
    }

Here's a breakdown of the configuration file structure:

- ``ddsmodule``: This section contains the configuration for the DDS transport layer.

  - ``domain``: The DDS domain ID used by both coraine and the ROS 2 service server.
  - ``transport``: The DDS transport used for communication. In this tutorial, UDP is used.

- ``ngsild``: This section defines the mapping between ROS 2 services and NGSI-LD entities.

  - ``services``: This section contains the ROS 2 services that coraine can call.
  - ``add_two_ints``: The name of the ROS 2 service.
  - ``entityType``: The type of the NGSI-LD entity associated with the service.
  - ``entityId``: The unique identifier of the NGSI-LD entity associated with the service.
  - ``attribute``: The NGSI-LD attribute used to represent the service request and response.
  - ``requestType`` and ``replyType``: The DDS type names of the service request and reply.
    ``ros2 service type /add_two_ints`` shows the ROS 2 type, ``example_interfaces/srv/AddTwoInts``. On DDS it becomes the names above.

With this configuration, a request written to the ``add_two_ints`` attribute of the
``urn:ngsi-ld:robot:1`` entity becomes a call to the ``/add_two_ints`` ROS 2 service.

To run the context broker, create a ``docker-compose-srv.yml`` file with the following content:

.. code-block:: yaml

    services:

      coraine:
        image: quay.io/seamware/coraine:0.7.0
        network_mode: host
        ipc: host
        restart: unless-stopped
        command:
          - --database
          - corDB
          - --dbDir
          - /data
          - --apiPlugins
          - admin
          - --bridges
          - dds
          - --bridgeConfig
          - /opt/seamware/etc/dds.json
        volumes:
          - ./context_broker_srv_config.json:/opt/seamware/etc/dds.json:ro
          - coraine_data:/data
          - dds_types:/opt/seamware/etc/dds-types

    volumes:
      coraine_data:
      dds_types:

This configuration starts coraine with its built-in database (``corDB``) and the DDS bridge.
No external database is needed.

Start the broker with the following command:

.. code-block:: bash

    docker compose -f docker-compose-srv.yml up -d

.. note::

    If something goes wrong during the setup, check the broker logs:

    .. code-block:: bash

        docker compose -f docker-compose-srv.yml logs -f coraine

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

    docker run -it --rm --net=host --ipc=host \
      eprosima/vulcanexus:jazzy-desktop

This command runs the Vulcanexus container on the host network and the host IPC namespace, so it can reach the context broker.

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

.. code-block:: text

    /add_two_ints

You can also inspect the service type with:

.. code-block:: bash

    ros2 service type /add_two_ints

The expected type is:

.. code-block:: text

    example_interfaces/srv/AddTwoInts

Calling the ROS 2 Service from the Context Broker
-------------------------------------------------

At this stage, coraine is running with a service mapping and the ROS 2 service server is waiting for
requests.

To call the service, update the ``add_two_ints`` attribute of the ``urn:ngsi-ld:robot:1`` entity
with the two integers to add:

.. code-block:: bash

    payload='{"type":"Property","value":{"a":10,"b":32}}'

    curl -s "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1/attrs/add_two_ints?ddsSync=true" \
      -X PATCH \
      -d "$payload" \
      -H 'Content-Type: application/json' \
      -w '%{http_code}\n'

In this example:

- ``a`` is set to ``10``.
- ``b`` is set to ``32``.
- coraine uses the configured ``services`` mapping to call the ``/add_two_ints`` ROS 2 service.
- The ROS 2 service server receives the request and computes the sum.
- ``?ddsSync=true`` makes the request wait briefly for the reply (200 ms by default). If the reply arrives in time, the request returns ``204`` and the reply is already in the entity.

The service server terminal should show that it received a request similar to:

.. code-block:: text

    Incoming request
    a: 10 b: 32

.. note::

    Without ``?ddsSync=true``, or when the service is slower than the wait, coraine answers ``202 Accepted`` right away.
    The reply is written to the entity as soon as it arrives. A slow robot never blocks the broker.

    If no server provides the service, coraine answers ``503 Service Unavailable`` and stores nothing.

Querying the Service Response
-----------------------------

Query the NGSI-LD entity to inspect the ``add_two_ints`` attribute:

.. code-block:: bash

    curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?local=true" \
      -s -S \
      -H 'Accept: application/json' | jq

The response contains the ``add_two_ints`` attribute. Its ``value`` is the request you sent.
Two sub-properties sit next to it, ``request`` and ``reply``:

.. code-block:: json

    {
      "id": "urn:ngsi-ld:robot:1",
      "type": "Robot",
      "add_two_ints": {
        "type": "Property",
        "value": { "a": 10, "b": 32 },
        "request": {
          "type": "Property",
          "value": { "a": 10, "b": 32 },
          "ddsDataType": {
            "type": "Property",
            "value": "example_interfaces::srv::dds_::AddTwoInts_Request_"
          },
          "requestId": { "type": "Property", "value": 1 },
          "publishedAt": { "type": "Property", "value": 1763900000 },
          "observedAt": "2026-11-23T10:00:00.000Z"
        },
        "reply": {
          "type": "Property",
          "value": { "sum": 42 },
          "ddsDataType": {
            "type": "Property",
            "value": "example_interfaces::srv::dds_::AddTwoInts_Response_"
          },
          "requestId": { "type": "Property", "value": 1 },
          "participantId": { "type": "Property", "value": "..." },
          "instanceHandleId": { "type": "Property", "value": "..." },
          "publishedAt": { "type": "Property", "value": 1763900000 },
          "observedAt": "2026-11-23T10:00:00.010Z"
        }
      }
    }

The ``value`` of ``reply`` is the service response, with the computed sum. To print only the sum:

.. code-block:: bash

    curl -s "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?local=true" \
      -H 'Accept: application/json' | jq '.add_two_ints.reply.value'

.. code-block:: json

    { "sum": 42 }

The ``requestId`` of the reply matches the ``requestId`` of the request, so you can tell which request a reply answers.

Troubleshooting
---------------

If the service call does not reach the ROS 2 service server, check the following points:

- Make sure the ``add_two_ints`` service server is running.
- Make sure coraine and the ROS 2 container are using the same DDS domain (``domain`` in the configuration file and ``ROS_DOMAIN_ID`` in ROS 2).
- Make sure both containers use the host network and the host IPC namespace (``--net=host --ipc=host``, or ``network_mode: host`` and ``ipc: host``).
- Make sure the service name in the configuration file is ``add_two_ints``.
- Check that ROS 2 can see the service with:

  .. code-block:: bash

      ros2 service list
      ros2 service type /add_two_ints

- Check what the broker sees on the ROS 2 network:

  .. code-block:: bash

      curl -s http://localhost:1026/admin/bridges/dds/inventory | jq '.services[].name'

Conclusion
----------

In this tutorial, we called a ROS 2 service from the coraine context broker.
coraine was configured as a ROS 2 service client through a DDS-to-NGSI-LD service mapping, and a ROS 2
``add_two_ints`` service server was launched in a Vulcanexus container.

By following these steps, you were able to:

- Configure coraine with an NGSI-LD mapping for a ROS 2 service.
- Launch a ROS 2 service server.
- Send service requests through the NGSI-LD REST API.
- Read the service result from the context broker.

You can extend this example by adding more ROS 2 services to the ``services`` section of the
``context_broker_srv_config.json`` file and mapping each one to a different NGSI-LD entity attribute.
The next tutorial, :ref:`ros2_fiware_actions_app`, covers long-running tasks.
