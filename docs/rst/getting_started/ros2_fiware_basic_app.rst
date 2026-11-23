.. _ros2_fiware_basic_app:

Communicating ROS 2 and the coraine Context Broker
==================================================

Introduction
------------
This tutorial shows how to connect ROS 2 and the ARISE context broker, `coraine <https://github.com/SEAMWARE/coraine>`__, using Docker containers.
coraine is an NGSI-LD context broker. Its DDS bridge maps ROS 2 topics, services and actions to NGSI-LD entity attributes.
By the end of this tutorial, you will be able to:

- Publish data from ROS 2 to the context broker.
- Inject data into the context broker and have it published in ROS 2.

We will use Docker Compose to run coraine, and a Vulcanexus Docker container to run the ROS 2 application.
You do not need to install anything beyond the tools listed below.

Prerequisites
-------------
- Docker installed on your system (Linux, ``amd64``).
- Docker Compose v2 installed on your system.
- ``curl`` and ``jq`` installed on your host.

.. note::

    The coraine image for ``arm64`` is built without the DDS bridge.
    To follow this tutorial, use an ``amd64`` machine.

Running the Context Broker
--------------------------

The context broker needs a simple configuration file. The file maps DDS Topics to NGSI-LD entity attributes.
Data published on a mapped DDS Topic is stored in the matching attribute of an NGSI-LD entity.
Data written to that attribute is published back on the DDS Topic.
The mapping therefore works in both directions.

Create a file named ``context_broker_config.json`` with the following content:

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
          "topics": {
            "rt/chatter": {
              "entityType": "Robot",
              "entityId": "urn:ngsi-ld:robot:1",
              "attribute": "chatter"
            }
          }
        }
      }
    }

Here's a breakdown of the configuration file structure:

- ``ddsmodule``: This section contains the configuration for the DDS transport layer.
  In this case, it sets the domain ID and the transport type (UDP).
  The domain must match the ``ROS_DOMAIN_ID`` of your ROS 2 nodes (``0`` by default).

- ``ngsild``: This section defines the mapping between DDS Topics and NGSI-LD entities.
  Each topic is associated with:

  - ``entityType``: The type of the NGSI-LD entity (e.g., ``Robot``).
  - ``entityId``: The unique identifier for the NGSI-LD entity (e.g., ``urn:ngsi-ld:robot:1``).
  - ``attribute``: The attribute of the NGSI-LD entity that corresponds to the DDS Topic (e.g., ``chatter``).

With this configuration, data published on the ``rt/chatter`` DDS Topic in ROS 2 is stored in the ``chatter`` attribute of the ``urn:ngsi-ld:robot:1`` entity.
Data written to that attribute is published back on the ``rt/chatter`` DDS Topic in ROS 2.

.. note::

    ROS 2 adds the prefix ``rt/`` to topic names on the DDS layer. The ROS 2 topic ``/chatter`` is the DDS Topic ``rt/chatter``.

Topics that are not in the configuration file are not lost.
coraine stores them in a default NGSI-LD entity, ``urn:ngsi-ld:dds:default``, with one attribute per topic.
This is a quick way to see what is on the ROS 2 network. For real deployments, map every topic you need in the configuration file.

To run the context broker, we will use Docker Compose.
Create a ``docker-compose.yml`` file with the following content:

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
          - ./context_broker_config.json:/opt/seamware/etc/dds.json:ro
          - coraine_data:/data
          - dds_types:/opt/seamware/etc/dds-types

    volumes:
      coraine_data:
      dds_types:

This configuration starts coraine with:

- ``--database corDB``: coraine's built-in database. No external database is needed. ``--dbDir /data`` keeps the data in the ``coraine_data`` volume.
- ``--bridges dds`` and ``--bridgeConfig``: loads the DDS bridge with your mapping file.
- ``--apiPlugins admin``: enables the admin API, used later to list what is on the ROS 2 network.
- ``network_mode: host`` and ``ipc: host``: let the broker discover ROS 2 nodes on the same machine. Fast DDS uses shared memory between processes on the same host, so ``ipc: host`` is required.

.. note::

    coraine images have no ``latest`` tag. This tutorial pins ``0.7.0``.
    You can pick a newer tag from `quay.io/seamware/coraine <https://quay.io/repository/seamware/coraine>`__.

To start the broker, run the following command:

.. code-block:: bash

    docker compose up -d

This command will download the image and start the container in detached mode.

Check that the broker is running:

.. code-block:: bash

    curl -s http://localhost:1026/version | jq

The mapped entity exists from startup. Its attribute has the value ``"uninitialized"`` until the first ROS 2 message arrives:

.. code-block:: bash

    curl -s "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?local=true" | jq

.. note::

    If something goes wrong during the setup, check the broker logs:

    .. code-block:: bash

      docker compose logs -f coraine

.. note::

    To stop and remove the running containers, use the following command:

    .. code-block:: bash

      docker compose down

Running the ROS 2 Publisher
---------------------------

To run the ROS 2 publication node, we will use a Vulcanexus Docker container.
Pull the Vulcanexus Jazzy image with the following command:

.. code-block:: bash

    docker pull eprosima/vulcanexus:jazzy-desktop

Create and run a new container with the following command:

.. code-block:: bash

    docker run -it --rm --net=host --ipc=host \
      eprosima/vulcanexus:jazzy-desktop

This command runs the Vulcanexus container on the host network and the host IPC namespace, so it can reach the context broker.

To start publishing data, you can use the ``talker`` node provided by the ``demo_nodes_cpp`` package.
It publishes messages such as "Hello World: [count]" to the ``/chatter`` topic (``rt/chatter`` on DDS), where ``[count]`` is an incrementing number.

To run the ``talker`` node inside the Vulcanexus container, execute the following command:

.. code-block:: bash

  ros2 run demo_nodes_cpp talker

You should see output indicating that messages are being published.
The context broker receives them because ``rt/chatter`` is mapped in ``context_broker_config.json``.

Querying the Context Broker via REST API
----------------------------------------

At this stage, the context broker is receiving the data published in ROS 2 and storing it in the ``chatter`` attribute of the ``urn:ngsi-ld:robot:1`` entity.
To check this data, query the context broker with ``curl``.

The following loop sends a ``GET`` request every second and prints the current value of the ``chatter`` attribute:

.. code-block:: bash

    while true; do
        curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?local=true" -s -S -H 'Accept: application/json' | jq -r '.chatter.value.data'
        sleep 1
    done

In this example:

- The ``curl`` command requests the ``urn:ngsi-ld:robot:1`` entity from the context broker.
- The ``-H 'Accept: application/json'`` header asks for a plain JSON response.
- The ``jq`` tool extracts the ``data`` field of the ``chatter`` value.
- The ``sleep 1`` command waits one second between queries, so you can see the value change.

You should see the "Hello World: [count]" messages published by the ``talker`` node.

Injecting Data into the Context Broker via REST API
---------------------------------------------------

In this section, we write data into the context broker with its REST API and receive it in a ROS 2 subscriber.

Before proceeding, stop the previous ``curl`` loop and the ``talker`` node.
You can do this by pressing ``Ctrl+C`` in each terminal.

To update the ``chatter`` attribute of the ``urn:ngsi-ld:robot:1`` entity, use the following command:

.. code-block:: bash

    payload='{"type":"Property","value":{"data":"Hello World from Context Broker"}}'

    curl http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1/attrs/chatter -X PATCH -d "$payload" -H 'Content-Type: application/json'

In this example:

- The ``-X PATCH`` option specifies that the request is a partial update.
- The URL points to the ``chatter`` attribute of the ``urn:ngsi-ld:robot:1`` entity.
- The ``-d`` option provides the JSON payload with the new value.
- The ``-H 'Content-Type: application/json'`` header indicates that the request body is in JSON format.

coraine publishes the value on the DDS Topic **before** it stores it.
If the value does not fit the topic's message type (``std_msgs/msg/String`` here), coraine answers ``400 Bad Request`` and stores nothing.
This way, the entity always matches what is actually on the ROS 2 network.

Receiving Data in a ROS 2 Subscription Node
-------------------------------------------

To receive the data written to the context broker, use the ``listener`` node provided by the ``demo_nodes_cpp`` package.
This node subscribes to the ``/chatter`` topic and prints the messages it receives.

Run the ``listener`` node in the same Vulcanexus container used earlier for the ``talker`` node:

.. code-block:: bash

  ros2 run demo_nodes_cpp listener

Now run the ``PATCH`` command from the previous section again.
The ``listener`` prints the message written to the context broker:

.. code-block:: text

    [INFO] [listener]: I heard: [Hello World from Context Broker]

Conclusion
----------

In this tutorial, we connected ROS 2 and the coraine context broker using Docker containers. You were able to:

- Publish data from ROS 2 to the context broker.
- Write data to the context broker and receive it in ROS 2.

This two-way link lets robots share their state with other systems, such as dashboards, digital twins or data spaces, through the standard NGSI-LD API.

You can extend this setup by adding more DDS Topics and NGSI-LD entities to the configuration file.
To go further, see :ref:`ros2_fiware_services_app`, :ref:`ros2_fiware_actions_app` and :ref:`coraine_how_to`.
