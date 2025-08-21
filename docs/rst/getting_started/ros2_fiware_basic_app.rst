.. _ros2_fiware_basic_app:

Communicating ROS 2 and FIWARE Context Broker
=============================================

Introduction
------------
This tutorial demonstrates how to establish communication between ROS 2 and the FIWARE Context Broker using Docker containers. By the end of this tutorial, you will be able to:

- Publish data from ROS 2 to the FIWARE Context Broker.
- Inject data into the FIWARE Context Broker and have it published in ROS 2.

We will use Docker Compose to run the FIWARE Context Broker, and a Vulcanexus Docker container to run the ROS 2 application.
This setup ensures that no prior installations are required beyond Docker and Docker Compose.

Prerequisites
-------------
- Docker installed on your system.
- Docker Compose installed on your system.

Running the FIWARE Context Broker
---------------------------------

The Context Broker requires a simple configuration file to establish a mapping between DDS Topics and NGSI-LD entity attributes.
This configuration file acts as a bridge, ensuring that data published on specific DDS Topics is correctly associated with the corresponding attributes of NGSI-LD entities.
By defining this mapping in the configuration file, the Context Broker facilitates bidirectional translation and synchronization of data between DDS (Data Distribution Service) and NGSI-LD.

Create a file named `context_broker_config.json` with the following content:

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

- ``dds``: This section contains the configuration for the DDS transport layer.
  In this case, it specifies the domain ID and transport type (e.g., UDP) used for communication.

- ``ngsild``: This section defines the mapping between DDS Topics and NGSI-LD entities.
  Each topic is associated with:

  - ``entityType``: The type of the NGSI-LD entity (e.g., ``Robot``).
  - ``entityId``: The unique identifier for the NGSI-LD entity (e.g., ``urn:ngsi-ld:robot:1``).
  - ``attribute``: The attribute of the NGSI-LD entity that corresponds to the DDS Topic (e.g., ``chatter``).

This configuration ensures that data published on the ``rt/chatter`` DDS Topic in ROS 2 is mapped to the ``chatter`` attribute of the ``Robot`` entity in the FIWARE Context Broker. Similarly, data injected into the ``chatter`` attribute of the ``Robot`` entity in the Context Broker is published back to the ``rt/chatter`` DDS Topic in ROS 2.

It is important to note that any data published to a DDS Topic that is not explicitly defined in the `context_broker_config.json` file will be automatically saved in a default NGSI-LD entity.
This entity is identified by the unique identifier ``urn:ngsi-ld:dds:default``.

For example:

- If a message is published to a topic named ``/unknown_topic`` that is not mapped in the configuration file, the data will be stored in the ``urn:ngsi-ld:dds:default`` entity.
- The attribute name for such data will match the DDS Topic name (e.g., ``unknown_topic``).

This behavior ensures that no data is lost, even if the topic is not explicitly defined in advance.
However, for better organization and clarity, it is recommended to define all expected topics in the configuration file whenever possible.

To run the FIWARE Context Broker, we will use Docker Compose.
Create a `docker-compose.yml` file with the following content:

.. code-block:: yaml

    services:

        mongodb:
            image: mongo:4.4
            privileged: true
            ipc: host
            network_mode: host
            command: --bind_ip_all
            volumes:
            - mongo_data:/data/db

    orion:
            image: fiware/orion-ld:1.10.0-PRE-1711
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
                - ./context_broker_config.json:/root/.orionld
            healthcheck:
                test: curl --fail -s http://localhost:1026/version || exit 1
                interval: 30s
                retries: 15

    volumes:
        mongo_data:


This configuration will set up FIWARE Context Broker and MongoDB, the database used by the Context Broker to save all data. To start the services, run the following command:

.. code-block:: bash

    docker compose up -d

.. note::

    This tutorial utilizes Docker Compose to manage containerized applications.
    It assumes that Docker Compose v2 (which is integrated with the Docker CLI) is installed on your system, so the `docker compose` command is used instead of the older `docker-compose` command.

    If the previous command did not work, try using the following command instead:

    .. code-block:: bash

        docker-compose up -d

This command will download the necessary images and start the containers in detached mode.

.. note::

    If something goes wrong during the setup, you can run the previous command without the detached mode to see the logs coming from the Context Broker:

    .. code-block:: bash

      docker compose up

    This will display the logs in real time, helping you identify any issues.

.. note::

    To stop and remove the running containers, you can use the following command:

    .. code-block:: bash

      docker compose down

    This will clean up the environment by stopping and removing all containers defined in the `docker-compose.yml` file.

Running the ROS 2 Publisher
---------------------------

To run the ROS 2 publication node, we will use a Vulcanexus Docker container.
Pull the latest Vulcanexus image with the following command:

.. code-block:: bash

    docker pull eprosima/vulcanexus:jazzy-desktop

Create and run a new container with the following command:

.. code-block:: bash

    docker run -it --rm --net=host --ipc=host --privileged \
      -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix \
      eprosima/vulcanexus:jazzy-desktop

This command will run the Vulcanexus container and connect it to the host network, allowing it to communicate with the FIWARE Context Broker.

To start publishing data, you can use the `talker` node provided by the `demo_nodes_cpp` package.
This node is a simple example that publishes string messages to a topic named ``rt/chatter``.

The `talker` node demonstrates the basic functionality of a ROS 2 publisher.
It continuously publishes messages such as "Hello World: [count]" to the ``rt/chatter`` topic, where ``[count]`` is an incrementing number.

To run the `talker` node inside the Vulcanexus container, execute the following command:

.. code-block:: bash

  ros2 run demo_nodes_cpp talker

This command will start the `talker` node, and you should see output indicating that messages are being published to the ``rt/chatter`` topic.
These messages can then be consumed by the FIWARE Context Broker if the appropriate mapping is configured in the `context_broker_config.json` file.

Querying the Context Broker via REST API
----------------------------------------

At this stage, the FIWARE Context Broker is actively receiving data published in the ROS 2 environment and storing it in the ``chatter`` attribute of the ``Robot`` entity.
To verify and access this data, you can query the Context Broker using the ``curl`` command.

The ``curl`` command sends a ``GET`` request to the Context Broker to retrieve the current value of the ``chatter`` attribute.
To continuously monitor the data, you can execute this query in a loop, fetching a new value every second.
Below is an example of how to achieve this:

.. code-block:: bash

    while true; do
        curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?prettyPrint=yes&local=true" -s -S -H 'Accept: application/json' | jq -r '.chatter.value.data'
        sleep 1
    done

In this example:

- The ``curl`` command sends a request to the Context Broker's endpoint, specifying the entity ID (``urn:ngsi-ld:robot:1``) and the attribute (``chatter``) to retrieve.
- The ``-H 'Accept: application/json'`` header ensures the response is returned in JSON format.
- The ``jq`` tool is used to format and display the JSON response in a readable manner.
- The ``sleep 1`` command introduces a one-second delay between each query, allowing you to observe the data updates in real time.

By running this loop, you can continuously monitor the data being published from the ROS 2 environment and stored in the Context Broker.
This provides a simple yet effective way to validate the integration and observe the flow of data between ROS 2 and FIWARE.

Injecting Data into the Context Broker via REST API
---------------------------------------------------

In this section, we will demonstrate how to inject data into the FIWARE Context Broker using its REST API and retrieve this data in a ROS 2 subscriber.
This process showcases the bidirectional communication between the FIWARE Context Broker and ROS 2, enabling seamless data exchange between the two systems.

Before proceeding, ensure that you stop the previous ``curl`` command and the ``talker`` node.
You can do this by pressing ``Ctrl+C`` in the respective terminal windows where they are running.

To inject data into the FIWARE Context Broker, you can use the ``curl`` command to send a ``POST`` or ``PATCH`` request. This allows you to create or update entities and their attributes in the Context Broker.

For example, to update the ``chatter`` attribute of the ``Robot`` entity, you can use the following command:

.. code-block:: bash

    payload='{"value":{"data":"Hello World from Context Broker"}}'

    curl http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1/attrs/chatter -X PATCH -d "$payload" -H 'Content-Type: application/json'

In this example:

- The ``-X PATCH`` option specifies that the request is a partial update.
- The URL points to the ``attrs`` endpoint of the ``Robot`` entity.
- The ``-d`` option provides the JSON payload, which updates the ``chatter`` attribute with a new value.
- The ``-H 'Content-Type: application/json'`` header indicates that the request body is in JSON format.

After running this command, the updated value will be available in the ROS 2 environment if the appropriate mapping is configured in the `context_broker_config.json` file.

Receiving Data in a ROS 2 Subscription Node
-------------------------------------------

To retrieve the data injected into the FIWARE Context Broker and published back into the ROS 2 environment, you can use the `listener` node provided by the `demo_nodes_cpp` package. This node subscribes to the ``rt/chatter`` topic and displays the messages it receives.

To run the `listener` node you can use the same Vulcanexus container used earlier for the `talker` node.
To start the `listener` node, execute the following command in the Vulcanexus container:

.. code-block:: bash

  ros2 run demo_nodes_cpp listener

The `listener` node will begin subscribing to the ``rt/chatter`` topic and display the messages it receives in real time.
These messages include the data injected into the FIWARE Context Broker and published back into the ROS 2 environment.

Now, run the command from previous section to observe the data being published from the FIWARE Context Broker.

Conclusion
----------

In this tutorial, we demonstrated how to establish seamless communication between ROS 2 and the FIWARE Context Broker using Docker containers. By following the steps outlined, you were able to:

- Publish data from ROS 2 to the FIWARE Context Broker.
- Inject data into the FIWARE Context Broker and observe it being published back into the ROS 2 environment.

This bidirectional communication enables powerful integrations between robotics systems and smart city platforms, allowing for real-time data exchange and enhanced interoperability. The use of Docker ensures a portable and reproducible setup, making it easier to deploy and scale the solution.

Feel free to expand on this setup by adding more DDS Topics and NGSI-LD entities to the configuration file, or by integrating additional ROS 2 nodes and FIWARE components to suit your specific use case.


