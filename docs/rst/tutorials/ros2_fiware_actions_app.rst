.. _ros2_fiware_actions_app:

Calling ROS 2 Actions from FIWARE Context Broker
================================================

Introduction
------------

This tutorial demonstrates how to call a ROS 2 action from the FIWARE Context Broker using Docker containers.
In this example, Orion-LD acts as a client for a ROS 2 action, while a ROS 2 action server provides the ``fibonacci`` action.

ROS 2 actions work differently from ROS 2 services.
A service is a single request/response exchange, while an action is a long-running task that provides **feedback** during execution and a final **result** when it completes.
Because of this, the integration behaves differently:

- When you write a goal to the action attribute, Orion-LD sends a goal request to the ROS 2 action server.
- While the goal is being processed, Orion-LD updates the action attribute of the entity with the goal's **status** (``accepted``, ``executing``, ``succeeded`` …) and the latest **feedback**. Each goal is tracked under its own ``datasetId``, which corresponds to the ROS 2 ``goalId``.
- Once the goal finishes, Orion-LD **removes the goal-specific data from the current state** of the entity. Only the value you wrote (the goal request) remains in the live entity.
- The complete history of the goal (every feedback update and every status transition) is preserved in the temporal database (TRoE/PostgreSQL) and can be retrieved at any time using the goal's ``datasetId`` (the ``goalId``).

Optionally, you can attach an HTTP ``endpoint`` to the goal so that Orion-LD pushes the live feedback and status updates to you as NGSI-LD notifications, which is convenient for following a goal in real time.

By the end of this tutorial, you will be able to:

- Launch the FIWARE Context Broker with a DDS-to-NGSI-LD action mapping and temporal storage enabled.
- Start a ROS 2 action server that computes a Fibonacci sequence.
- Trigger a ROS 2 action goal from the Context Broker through the NGSI-LD REST API.
- Follow the goal's feedback and status live through an HTTP notification endpoint.
- Retrieve the full historical data of the goal from the temporal database using its ``goalId``.

We will use Docker Compose to run Orion-LD, MongoDB and TimescaleDB, and a Vulcanexus Docker container to run the ROS 2 action server.

.. note::

    This tutorial replicates the approach of the :ref:`ros2_fiware_services_app` tutorial, it is recommended to review that tutorial first.

Prerequisites
-------------

- Docker installed on your system.
- Docker Compose installed on your system.

Running the FIWARE Context Broker
---------------------------------

The Context Broker requires a configuration file to establish a mapping between ROS 2 actions and NGSI-LD entity attributes.

In this tutorial, the ``fibonacci`` ROS 2 action is mapped to the ``fibonacci`` attribute of the ``urn:ngsi-ld:robot:1`` NGSI-LD entity.
Orion-LD will use this mapping to act as a client of the ROS 2 action.

Create a file named ``context_broker_action_config.json`` with the following content:

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
          "actions": {
            "fibonacci": {
              "entityType": "Robot",
              "entityId": "urn:ngsi-ld:robot:1",
              "attribute": "fibonacci"
            }
          }
        }
      }
    }

Here's a breakdown of the configuration file structure:

- ``dds``: This section contains the configuration for the DDS transport layer.

  - ``domain``: The DDS domain ID used by both Orion-LD and the ROS 2 action server.
  - ``transport``: The DDS transport used for communication. In this tutorial, UDP is used.

- ``ngsild``: This section defines the mapping between ROS 2 actions and NGSI-LD entities.

  - ``actions``: This section contains the ROS 2 actions exposed through Orion-LD.
  - ``fibonacci``: The name of the ROS 2 action that Orion-LD will call.
  - ``entityType``: The type of the NGSI-LD entity associated with the action.
  - ``entityId``: The unique identifier of the NGSI-LD entity associated with the action.
  - ``attribute``: The NGSI-LD attribute used to represent the action goal, feedback and status.

This configuration ensures that goals written to the ``fibonacci`` attribute of the ``urn:ngsi-ld:robot:1`` entity are translated into ROS 2 action goals sent to the ``/fibonacci`` action server.

To run the FIWARE Context Broker, create a ``docker-compose-action.yml`` file with the following content:

.. code-block:: yaml

    services:

      timescale-db:
        image: timescale/timescaledb-postgis:1.7.5-pg12
        hostname: timescale-db
        container_name: db-timescale
        healthcheck:
          test: [ "CMD-SHELL", "pg_isready -U orion" ]
          interval: 15s
          timeout: 15s
          retries: 5
          start_period: 60s
        environment:
          - POSTGRES_USER=orion
          - POSTGRES_PASSWORD=orion
          - POSTGRES_HOST_AUTH_METHOD=trust
        command: ["postgres", "-c", "log_statement=none"]
        ports:
          - "5432:5432"
        volumes:
          - timescale-db:/var/lib/postgresql/data

      mongodb:
        image: mongo:7.0
        privileged: true
        ipc: host
        network_mode: host
        command: --bind_ip_all
        volumes:
          - mongo_data:/data/db

      orion:
        image: fiware/orion-ld:1.13.0-PRE-1852
        privileged: true
        ipc: host
        network_mode: host
        depends_on:
          mongodb:
            condition: service_started
          timescale-db:
            condition: service_healthy
        restart: always
        command: -dbhost localhost -wip dds -mongocOnly -troe
        environment:
          - ORIONLD_MONGO_HOST=localhost
          - ORIONLD_TROE_HOST=localhost
          - ORIONLD_TROE_PORT=5432
          - ORIONLD_TROE_USER=orion
          - ORIONLD_TROE_PWD=orion
        volumes:
          - ./context_broker_action_config.json:/root/.orionld
        healthcheck:
          test: curl --fail -s http://localhost:1026/version || exit 1
          interval: 30s
          retries: 15

    volumes:
      mongo_data:
      timescale-db:

This configuration starts MongoDB, TimescaleDB and the FIWARE Context Broker.
MongoDB stores the current state of the Context Broker, while TimescaleDB stores the temporal (historical) representation of the entities.

The ``-troe`` flag enables the **Temporal Representation of Entities** (TRoE), which is what allows us to query the full history of an action goal later on.

.. note::

    The ``ORIONLD_TROE_USER`` and ``ORIONLD_TROE_PWD`` environment variables must match the credentials of the TimescaleDB container (``orion`` / ``orion`` in this example).
    Orion-LD's defaults are ``postgres`` / ``password``, which do not exist in the TimescaleDB container, so without these variables the broker would fail to connect to PostgreSQL.

Start the services with the following command:

.. code-block:: bash

    docker compose -f docker-compose-action.yml up -d

.. note::

    This tutorial uses Docker Compose v2, which is integrated with the Docker CLI and uses the ``docker compose`` command.

    If that command does not work in your environment, try the older command instead:

    .. code-block:: bash

        docker-compose -f docker-compose-action.yml up -d

This command downloads the required images and starts the containers in detached mode.

.. note::

    If something goes wrong during the setup, run the previous command without detached mode to inspect the logs:

    .. code-block:: bash

        docker compose -f docker-compose-action.yml up

.. note::

    To stop and remove the running containers, use the following command:

    .. code-block:: bash

        docker compose -f docker-compose-action.yml down

.. note::

    The previous command keeps the data volumes, so MongoDB and TimescaleDB
    retain their data across restarts. To also delete the database volumes and
    start completely fresh (wiping both the current state and the temporal
    history), add the ``-v`` flag:

    .. code-block:: bash

        docker compose -f docker-compose-action.yml down -v

    This is irreversible: it removes the ``mongo_data`` and ``timescale-db``
    volumes and all the data they contain.

    If the containers were already stopped with a plain ``down`` (without
    ``-v``), you can remove the volumes afterwards with ``docker volume rm``.

    .. code-block:: bash

        docker volume ls
        docker volume rm <project>_mongo_data <project>_timescale-db

Running the ROS 2 Action Server
-------------------------------

To run the ROS 2 action server, use a Vulcanexus Docker container.

Pull the Vulcanexus image with the following command:

.. code-block:: bash

    docker pull eprosima/vulcanexus:jazzy-desktop

Create and run a new container with the following command:

.. code-block:: bash

    docker run -it --rm --net=host --ipc=host --privileged \
      -e DISPLAY=$DISPLAY -v /tmp/.X11-unix:/tmp/.X11-unix \
      eprosima/vulcanexus:jazzy-desktop

This command will run the Vulcanexus container and connect it to the host network, allowing it to communicate with the FIWARE Context Broker.

Inside the Vulcanexus container, run the ``fibonacci`` action server provided by the ``examples_rclcpp_minimal_action_server`` package:

.. code-block:: bash

    ros2 run examples_rclcpp_minimal_action_server action_server_member_functions

The action server will wait for goals on the ``/fibonacci`` action.
Each goal contains an ``order`` field, which is the number of terms of the Fibonacci sequence to compute.
While the goal is executing, the server publishes feedback (the partial sequence) and, when it finishes, it returns the full sequence as the result.

You can verify that the action is available from another terminal inside the same container with:

.. code-block:: bash

    ros2 action list

You should see the following action in the output:

.. code-block:: bash

    /fibonacci

You can also inspect the action type with:

.. code-block:: bash

    ros2 action type /fibonacci

The expected type is:

.. code-block:: bash

    example_interfaces/action/Fibonacci

Following the Action Live with a Notification Endpoint
------------------------------------------------------

Because an action is a long-running task, it is useful to follow its progress in real time.
Orion-LD can push the goal's feedback and status updates to an HTTP endpoint as NGSI-LD notifications.

Create a small HTTP server that prints every notification it receives, pretty-printed as JSON.
Save the following file as ``notify_listener.py``:

.. code-block:: python

    from http.server import BaseHTTPRequestHandler, HTTPServer
    import json
    import datetime


    class NotificationHandler(BaseHTTPRequestHandler):
        def do_POST(self):
            length = int(self.headers.get("Content-Length", 0) or 0)
            body = self.rfile.read(length) if length else b""

            # Acknowledge the notification.
            self.send_response(200)
            self.send_header("Content-Length", "0")
            self.end_headers()

            timestamp = datetime.datetime.now().strftime("%H:%M:%S")
            print(f"\n=== notification @ {timestamp} ===")
            try:
                print(json.dumps(json.loads(body), indent=2))
            except json.JSONDecodeError:
                print(body.decode(errors="replace"))

        def log_message(self, *args):
            pass  # silence the default request logging


    if __name__ == "__main__":
        print("Listening for notifications on http://0.0.0.0:7000 ...")
        HTTPServer(("0.0.0.0", 7000), NotificationHandler).serve_forever()

Run the listener in a dedicated terminal on the host:

.. code-block:: bash

    python3 notify_listener.py

It will wait for notifications on port ``7000``.

.. note::

    This is a more convenient alternative to a raw one-liner such as ``while true; do printf 'HTTP/1.1 200 OK\r\nContent-Length: 0\r\n\r\n' | nc -l -p 7000; done``.
    The Python listener pretty-prints the JSON body of each notification, making the feedback and status updates much easier to read.

Sending an Action Goal from the Context Broker
----------------------------------------------

At this stage, Orion-LD is running with an action mapping, the ROS 2 action server is waiting for goals, and the notification listener is ready.

To send a goal, write the ``fibonacci`` attribute of the ``urn:ngsi-ld:robot:1`` entity through the NGSI-LD REST API.
Include the ``endpoint`` sub-property so that Orion-LD pushes the live updates to your listener:

.. code-block:: bash

    curl -X PATCH http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1 \
      -H "Content-Type: application/json" \
      -d '{
            "fibonacci": {
              "value": { "order": 5 },
              "endpoint": { "value": "http://localhost:7000/notify" }
            }
          }'

In this example:

- ``order`` is set to ``5``, requesting the first 5 terms of the Fibonacci sequence.
- ``endpoint`` tells Orion-LD where to push the live notifications. It is optional; omit it if you only want to query the state or the history afterwards.
- Orion-LD uses the configured ``actions`` mapping to send a goal to the ``/fibonacci`` ROS 2 action server.
- The ROS 2 action server accepts the goal, executes it, and publishes feedback and the final result.

Observing the Live Feedback and Status
---------------------------------------

While the goal is executing, the notification listener will print a sequence of notifications.
Each one is an NGSI-LD ``Notification`` whose ``data`` array contains the entity with the ``fibonacci`` attribute.
The goal is identified by its ``datasetId`` (the ``goalId``), and carries three relevant sub-properties:

- ``ddsActionStatus``: the current status of the goal (``accepted``, ``executing``, ``succeeded`` …).
- ``ddsActionFeedback``: the latest feedback published by the action server (the partial sequence).
- ``ddsActionResult``: the final result returned by the action server when the goal completes (the full sequence).

A feedback notification looks like this:

.. code-block:: text

    === notification @ 14:13:37 ===
    {
      "id": "urn:ngsi-ld:Notification:6a021328-...",
      "type": "Notification",
      "subscriptionId": "urn:ngsi-ld:subscription:6a00b0aa-...",
      "notifiedAt": "2026-06-03T14:13:37.553Z",
      "data": [
        {
          "id": "urn:ngsi-ld:robot:1",
          "type": "Robot",
          "fibonacci": {
            "type": "Property",
            "datasetId": "urn:goal:9a387aad-8264-7baa-aeb8-dad745cb1c45",
            "value": { "order": 5 },
            "ddsActionFeedback": {
              "type": "Property",
              "value": { "sequence": [ 0, 1, 1 ] }
            }
          }
        }
      ]
    }

And a status notification looks like this:

.. code-block:: text

    "ddsActionStatus": {
      "type": "Property",
      "value": {
        "code": "accepted",
        "message": "Action goal accepted"
      }
    }

When the goal completes, a final notification carries the ``ddsActionResult`` with the full sequence:

.. code-block:: text

    "ddsActionResult": {
      "type": "Property",
      "value": {
        "result": { "sequence": [ 0, 1, 1, 2, 3, 5 ] },
        "status": 4
      }
    }

Over the life of the goal you will see the status transition through ``accepted`` → ``executing`` → ``succeeded``, the feedback grow from ``[0, 1]`` up to the complete sequence ``[0, 1, 1, 2, 3, 5]``, and a final ``ddsActionResult`` carrying that same sequence.

.. note::

    Take note of the ``datasetId`` value (for example ``urn:goal:9a387aad-8264-7baa-aeb8-dad745cb1c45``).
    This is the ``goalId`` and you will use it to retrieve the goal's history.

Querying the Current State
---------------------------

While the goal is executing, you can also query the live entity to see the goal under its ``datasetId``:

.. code-block:: bash

    curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?prettyPrint=yes&attrs=fibonacci" \
      -s -S \
      -H 'Accept: application/json' | jq

During execution, the ``fibonacci`` attribute is returned as an array of datasets: the goal request you wrote and the active goal with its ``ddsActionStatus`` and ``ddsActionFeedback``.

Once the goal **finishes**, Orion-LD removes the goal-specific data from the current state.
Querying the entity again returns only the value you wrote:

.. code-block:: text

    {
      "type": "Property",
      "value": { "order": 5 },
      "endpoint": {
        "type": "Property",
        "value": "http://localhost:7000/notify"
      }
    }

This is expected: the live entity always reflects the *current* state, and a completed goal is no longer active.
The full record of the goal lives in the temporal database.

Querying the Historical Data of a Goal
--------------------------------------

The complete history of the goal (every feedback update, every status transition and the final result) is stored in the temporal database (TRoE).
Retrieve it through the NGSI-LD temporal API, filtering by the ``fibonacci`` attribute and the ``datasetId`` (the ``goalId`` you noted earlier).

We show two ways of retrieving it: the full JSON representation, and a compact summary built with ``jq``.

Option 1: full JSON representation
""""""""""""""""""""""""""""""""""

Adapt the following request to your specific example by setting the ``datasetId`` to return the complete temporal representation, pretty-printed as JSON:

.. code-block:: bash

    curl -G "http://localhost:1026/ngsi-ld/v1/temporal/entities/urn:ngsi-ld:robot:1" \
      -H 'Accept: application/json' \
      --data-urlencode 'attrs=fibonacci' \
      --data-urlencode 'datasetId=<INTRODUCE_YOUR_DATA_SET_ID>' \
      --data-urlencode 'prettyPrint=yes'

The response returns the ``fibonacci`` attribute as an **array of timed instances**.
Each instance carries one of the action sub-properties (``ddsActionStatus`` (a status transition), ``ddsActionFeedback`` (a growing sequence) or ``ddsActionResult`` (the final sequence)) reconstructing the complete timeline of the goal:

.. code-block:: json

    [
      {
        "id": "urn:ngsi-ld:robot:1",
        "type": "Robot",
        "fibonacci": [
          {
            "type": "Property",
            "value": { "order": 5 },
            "datasetId": "urn:goal:9a387aad-8264-7baa-aeb8-dad745cb1c45",
            "ddsActionStatus": {
              "type": "Property",
              "value": { "code": "accepted", "message": "Action goal accepted" }
            }
          },
          {
            "type": "Property",
            "value": { "order": 5 },
            "datasetId": "urn:goal:9a387aad-8264-7baa-aeb8-dad745cb1c45",
            "ddsActionFeedback": {
              "type": "Property",
              "value": { "sequence": [ 0, 1, 1 ] }
            }
          },
          {
            "type": "Property",
            "value": { "order": 5 },
            "datasetId": "urn:goal:9a387aad-8264-7baa-aeb8-dad745cb1c45",
            "ddsActionResult": {
              "type": "Property",
              "value": { "result": { "sequence": [ 0, 1, 1, 2, 3, 5 ] }, "status": 4 }
            }
          }
        ]
      }
    ]

Option 2: compact summary with jq
"""""""""""""""""""""""""""""""""

If you only care about the values, pipe the response through ``jq`` to print one concise line per instance.
Drop the ``prettyPrint=yes`` parameter and format the output instead. Remember to adapt the command by setting the ``datasetId`` of the action you want to inspect.

.. code-block:: bash

    curl -s -G "http://localhost:1026/ngsi-ld/v1/temporal/entities/urn:ngsi-ld:robot:1" \
      -H 'Accept: application/json' \
      --data-urlencode 'attrs=fibonacci' \
      --data-urlencode 'datasetId=<INTRODUCE_YOUR_DATA_SET_ID>' \
      | jq -r '.fibonacci | reverse[] |
          if   .ddsActionFeedback then "ddsActionFeedback:  [" + ([.ddsActionFeedback.value.sequence[]|tostring]|join(", ")) + "]"
          elif .ddsActionResult   then "ddsActionResult:  ["   + ([.ddsActionResult.value.result.sequence[]|tostring]|join(", ")) + "]"
          elif .ddsActionStatus   then "ddsActionStatus: "     + .ddsActionStatus.value.code
          else empty end'

The ``jq`` filter inspects each instance, detects which action sub-property it carries
(``ddsActionFeedback``, ``ddsActionResult`` or ``ddsActionStatus``) and prints a single line for it.
The result is the goal's timeline in a compact, readable form:

.. code-block:: text

    ddsActionStatus: accepted
    ddsActionFeedback:  [0, 1, 1]
    ddsActionFeedback:  [0, 1, 1, 2]
    ddsActionFeedback:  [0, 1, 1, 2, 3]
    ddsActionStatus: executing
    ddsActionFeedback:  [0, 1, 1, 2, 3, 5]
    ddsActionStatus: succeeded
    ddsActionResult:  [0, 1, 1, 2, 3, 5]

.. note::

    Because the ``goalId`` is part of the URI, use ``-G`` together with ``--data-urlencode`` so that curl encodes the parameters correctly.

    You can also limit the number of instances returned per attribute with ``--data-urlencode 'lastN=20'``.

.. note::

    Retrieving the temporal representation of a single entity by its id (as above) does not require the ``timerel`` parameter.
    If instead you query by type (``/ngsi-ld/v1/temporal/entities?type=Robot``), you must add a temporal filter, for example ``--data-urlencode 'timerel=after' --data-urlencode 'timeAt=1970-01-01T00:00:00Z'`` to retrieve everything.

Inspecting PostgreSQL Directly (Optional)
-----------------------------------------

If you want to look at the raw temporal data, you can connect to the TimescaleDB container.
The list of ``goalId``s recorded for the ``fibonacci`` attribute can be obtained with:

.. code-block:: bash

    docker exec db-timescale psql -U orion -d orion -c \
      "SELECT DISTINCT datasetid FROM attributes WHERE id LIKE '%fibonacci' AND datasetid IS NOT NULL;"

And the recorded instances of a goal with:

.. code-block:: bash

    docker exec db-timescale psql -U orion -d orion -c \
      "SELECT datasetid, compound, ts FROM attributes WHERE id LIKE '%fibonacci' ORDER BY ts DESC LIMIT 20;"

Troubleshooting
---------------

If the action goal does not reach the ROS 2 action server, check the following points:

- Make sure the ``fibonacci`` action server is running.
- Make sure Orion-LD and the ROS 2 container are using the same DDS domain.
- Make sure both containers are using ``--net=host`` or ``network_mode: host``.
- Make sure the action name in the configuration file is ``fibonacci``.

- Check that ROS 2 can see the action, and that it has one server, with:

  .. code-block:: bash

      ros2 action list
      ros2 action info /fibonacci

If you do not receive notifications on your endpoint:

- Make sure the ``notify_listener.py`` server is running and listening on port ``7000``.
- Make sure the ``endpoint`` value in the goal points to a host and port reachable from the Orion-LD container (``http://localhost:7000/notify`` works when Orion-LD runs with ``network_mode: host``).

If the historical query returns ``Entity Not Found``:

- Make sure the broker was started with the ``-troe`` flag and that the TimescaleDB container is healthy.
- Remember that only entities **created while TRoE was enabled** appear in temporal queries.

Conclusion
----------

In this tutorial, we demonstrated how to call a ROS 2 action from the FIWARE Context Broker.
Orion-LD was configured as a ROS 2 action client through a DDS-to-NGSI-LD action mapping, a ROS 2 ``fibonacci`` action server was launched in a Vulcanexus container, and temporal storage was enabled to keep the history of each goal.

By following these steps, you were able to:

- Configure Orion-LD with an NGSI-LD mapping for a ROS 2 action and enable temporal storage.
- Launch a ROS 2 action server.
- Send an action goal through the NGSI-LD REST API.
- Follow the goal's feedback and status live through an HTTP notification endpoint.
- Observe how the goal data is removed from the current state once the goal completes.
- Retrieve the full history of the goal from the temporal database using its ``goalId``.

You can extend this example by adding more ROS 2 actions to the ``actions`` section of the ``context_broker_action_config.json`` file and mapping each one to a different NGSI-LD entity attribute.
