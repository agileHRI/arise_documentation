.. _ros2_fiware_actions_app:

Calling ROS 2 Actions from the coraine Context Broker
=====================================================

Introduction
------------

This tutorial shows how to call a ROS 2 action from the ARISE context broker, coraine, using Docker containers.
In this example, coraine acts as a client for a ROS 2 action, while a ROS 2 action server provides the ``fibonacci`` action.

ROS 2 actions work differently from ROS 2 services.
A service is a single request/response exchange, while an action is a long-running task that provides **feedback** during execution and a final **result** when it completes.
Because of this, the integration behaves differently:

- When you write a goal to the action attribute, coraine sends a goal request to the ROS 2 action server and answers ``202 Accepted``.
- While the goal is being processed, coraine updates the action attribute of the entity with the goal's **status** (``accepted``, ``executing``, ``succeeded`` …) and the latest **feedback**. Each goal is tracked under its own ``datasetId``, which corresponds to the ROS 2 ``goalId``.
- Once the goal finishes, coraine **removes the goal's instance** from the current state of the entity. The attribute itself stays, with the last goal you sent as its value.
- The complete history of the goal (every feedback update and every status transition) is kept in the temporal history (TRoE) and can be retrieved at any time using the goal's ``datasetId`` (the ``goalId``).

Optionally, you can attach an HTTP ``endpoint`` to the goal so that coraine pushes the live feedback and status updates to you as NGSI-LD notifications, which is convenient for following a goal in real time.

By the end of this tutorial, you will be able to:

- Launch the context broker with a DDS-to-NGSI-LD action mapping and temporal storage enabled.
- Start a ROS 2 action server that computes a Fibonacci sequence.
- Trigger a ROS 2 action goal from the Context Broker through the NGSI-LD REST API.
- Follow the goal's feedback and status live through an HTTP notification endpoint.
- Retrieve the full history of the goal using its ``goalId``.

We will use Docker Compose to run coraine, and a Vulcanexus Docker container to run the ROS 2 action server.
coraine keeps both the current state and the temporal history in its built-in database, so no external database is needed.

.. note::

    This tutorial follows the same approach as :ref:`ros2_fiware_services_app`. We recommend that you complete that tutorial first.

Prerequisites
-------------

- Docker installed on your system (Linux, ``amd64``).
- Docker Compose v2 installed on your system.
- ``curl``, ``jq`` and ``python3`` installed on your host.

Running the Context Broker
--------------------------

The context broker requires a configuration file to establish a mapping between ROS 2 actions and NGSI-LD entity attributes.

In this tutorial, the ``fibonacci`` ROS 2 action is mapped to the ``fibonacci`` attribute of the ``urn:ngsi-ld:robot:1`` NGSI-LD entity.
coraine will use this mapping to act as a client of the ROS 2 action.

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
              "attribute": "fibonacci",
              "type": "example_interfaces::action::dds_::Fibonacci"
            }
          }
        }
      }
    }

Here's a breakdown of the configuration file structure:

- ``ddsmodule``: This section contains the configuration for the DDS transport layer.

  - ``domain``: The DDS domain ID used by both coraine and the ROS 2 action server.
  - ``transport``: The DDS transport used for communication. In this tutorial, UDP is used.

- ``ngsild``: This section defines the mapping between ROS 2 actions and NGSI-LD entities.

  - ``actions``: This section contains the ROS 2 actions that coraine can call.
  - ``fibonacci``: The name of the ROS 2 action.
  - ``entityType``: The type of the NGSI-LD entity associated with the action.
  - ``entityId``: The unique identifier of the NGSI-LD entity associated with the action.
  - ``attribute``: The NGSI-LD attribute used to represent the action goal, feedback and status.
  - ``type``: The DDS type of the action. ``ros2 action type /fibonacci`` shows the ROS 2 type, ``example_interfaces/action/Fibonacci``. On DDS it becomes ``example_interfaces::action::dds_::Fibonacci``.

This configuration ensures that goals written to the ``fibonacci`` attribute of the ``urn:ngsi-ld:robot:1`` entity are translated into ROS 2 action goals sent to the ``/fibonacci`` action server.

To run the context broker, create a ``docker-compose-action.yml`` file with the following content:

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
          - --troe
          - corDB
          - --apiPlugins
          - admin
          - --bridges
          - dds
          - --bridgeConfig
          - /opt/seamware/etc/dds.json
        volumes:
          - ./context_broker_action_config.json:/opt/seamware/etc/dds.json:ro
          - coraine_data:/data
          - dds_types:/opt/seamware/etc/dds-types

    volumes:
      coraine_data:
      dds_types:

This configuration starts coraine with its built-in database (``corDB``) and the DDS bridge.

The ``--troe corDB`` option enables the **Temporal Representation of Entities** (TRoE) in the same built-in database.
TRoE is what allows us to query the full history of an action goal later on.

.. note::

    For large or long-lived deployments, coraine can keep the temporal history in PostgreSQL/TimescaleDB instead, with ``--troe timescale``.
    See :ref:`coraine_how_to`.

Start the broker with the following command:

.. code-block:: bash

    docker compose -f docker-compose-action.yml up -d

.. note::

    If something goes wrong during the setup, check the broker logs:

    .. code-block:: bash

        docker compose -f docker-compose-action.yml logs -f coraine

.. note::

    To stop and remove the running containers, use the following command:

    .. code-block:: bash

        docker compose -f docker-compose-action.yml down

.. note::

    The previous command keeps the data volume, so coraine keeps its data
    across restarts. To also delete the volumes and start completely fresh
    (wiping both the current state and the temporal history), add the ``-v``
    flag:

    .. code-block:: bash

        docker compose -f docker-compose-action.yml down -v

    This is irreversible: it removes the ``coraine_data`` and ``dds_types``
    volumes and all the data they contain.

    If the containers were already stopped with a plain ``down`` (without
    ``-v``), you can remove the volumes afterwards with ``docker volume rm``.

    .. code-block:: bash

        docker volume ls
        docker volume rm <project>_coraine_data <project>_dds_types

Running the ROS 2 Action Server
-------------------------------

To run the ROS 2 action server, use a Vulcanexus Docker container.

Pull the Vulcanexus image with the following command:

.. code-block:: bash

    docker pull eprosima/vulcanexus:jazzy-desktop

Create and run a new container with the following command:

.. code-block:: bash

    docker run -it --rm --net=host --ipc=host \
      eprosima/vulcanexus:jazzy-desktop

This command runs the Vulcanexus container on the host network and the host IPC namespace, so it can reach the context broker.

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

.. code-block:: text

    /fibonacci

You can also inspect the action type with:

.. code-block:: bash

    ros2 action type /fibonacci

The expected type is:

.. code-block:: text

    example_interfaces/action/Fibonacci

Following the Action Live with a Notification Endpoint
------------------------------------------------------

Because an action is a long-running task, it is useful to follow its progress in real time.
coraine can push the goal's feedback and status updates to an HTTP endpoint as NGSI-LD notifications.

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

At this stage, coraine is running with an action mapping, the ROS 2 action server is waiting for goals, and the notification listener is ready.

To send a goal, write the ``fibonacci`` attribute of the ``urn:ngsi-ld:robot:1`` entity through the NGSI-LD REST API.
Include the ``endpoint`` sub-property so that coraine pushes the live updates to your listener:

.. code-block:: bash

    curl -X PATCH http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1 \
      -H "Content-Type: application/json" \
      -w '%{http_code}\n' \
      -d '{
            "fibonacci": {
              "type": "Property",
              "value": { "order": 5 },
              "endpoint": { "value": "http://localhost:7000/notify" }
            }
          }'

In this example:

- ``order`` is set to ``5``, requesting the first 5 terms of the Fibonacci sequence.
- ``endpoint`` tells coraine where to push the live notifications. It is optional; omit it if you only want to query the state or the history afterwards.
- coraine uses the configured ``actions`` mapping to send a goal to the ``/fibonacci`` ROS 2 action server.
- The ROS 2 action server accepts the goal, executes it, and publishes feedback and the final result.
- coraine answers ``202 Accepted``: the goal has been sent, and its progress arrives later.

.. note::

    If no server provides the action, coraine answers ``503 Service Unavailable`` and stores nothing.
    If the server rejects the goal, coraine answers ``422`` and stores nothing.

Observing the Live Feedback and Status
---------------------------------------

While the goal is executing, the notification listener will print a sequence of notifications (about eight for a goal of order 5).
Each one is an NGSI-LD ``Notification`` whose ``data`` array contains the entity with the ``fibonacci`` attribute.
The goal is identified by its ``datasetId`` (the ``goalId``), and carries up to three sub-properties:

- ``ddsActionStatus``: the current status of the goal (``accepted``, ``executing``, ``succeeded`` …).
- ``ddsActionFeedback``: the latest feedback published by the action server (the partial sequence).
- ``ddsActionResult``: the final result returned by the action server when the goal completes (the full sequence).

Each notification carries the latest value of every sub-property received so far. A feedback notification looks like this:

.. code-block:: text

    === notification @ 14:13:37 ===
    {
      "id": "urn:ngsi-ld:Notification:6a021328-...",
      "type": "Notification",
      "subscriptionId": "urn:ngsi-ld:Subscription:6a00b0aa-...",
      "notifiedAt": "2026-11-23T14:13:37.553Z",
      "data": [
        {
          "id": "urn:ngsi-ld:robot:1",
          "type": "Robot",
          "fibonacci": {
            "type": "Property",
            "datasetId": "urn:goal:42dd44ac-a3fd-ddbb-4361-b32c36fd1478",
            "value": { "order": 5 },
            "ddsActionStatus": {
              "type": "Property",
              "value": { "code": "accepted", "message": "Action goal accepted" }
            },
            "ddsActionFeedback": {
              "type": "Property",
              "value": { "sequence": [ 0, 1, 1 ] }
            }
          }
        }
      ]
    }

When the goal completes, the last notification also carries the ``ddsActionResult`` with the full sequence:

.. code-block:: text

    "ddsActionResult": {
      "type": "Property",
      "value": {
        "result": { "sequence": [ 0, 1, 1, 2, 3, 5 ] },
        "status": 4
      }
    }

Over the life of the goal you will see the status go through ``accepted``, ``executing`` and ``succeeded``, the feedback grow from ``[0, 1, 1]`` up to the complete sequence ``[0, 1, 1, 2, 3, 5]``, and a final ``ddsActionResult`` carrying that same sequence.
The ``status`` value ``4`` is the ROS 2 code for a goal that succeeded.

.. note::

    Take note of the ``datasetId`` value (for example ``urn:goal:42dd44ac-a3fd-ddbb-4361-b32c36fd1478``).
    This is the ``goalId`` and you will use it to retrieve the goal's history.

Querying the Current State
---------------------------

While the goal is executing, you can also query the live entity to see the goal under its ``datasetId``:

.. code-block:: bash

    curl "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1?attrs=fibonacci" \
      -s -S \
      -H 'Accept: application/json' | jq

During execution, the ``fibonacci`` attribute is returned as an array of instances: the default instance, holding the goal request you wrote, and the active goal (under its ``datasetId``) with its ``ddsActionStatus`` and ``ddsActionFeedback``.

Once the goal **finishes**, coraine removes the goal's instance. The ``fibonacci`` attribute stays, with the request you wrote as its value:

.. code-block:: json

    {
      "id": "urn:ngsi-ld:robot:1",
      "type": "Robot",
      "fibonacci": {
        "type": "Property",
        "value": { "order": 5 },
        "endpoint": { "type": "Property", "value": "http://localhost:7000/notify" }
      }
    }

This is expected: the live entity only reflects the *current* state, and a completed goal is no longer active.
The full record of the goal lives in the temporal history.

.. note::

    To cancel a running goal, delete its instance:

    .. code-block:: bash

        curl -X DELETE "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:robot:1/attrs/fibonacci?datasetId=<INTRODUCE_YOUR_DATA_SET_ID>"

    coraine answers ``202 Accepted``, sends the cancel to the action server, and the goal ends with the status ``canceled``.

Querying the Historical Data of a Goal
--------------------------------------

The complete history of the goal (every feedback update, every status transition and the final result) is stored in the temporal history (TRoE).
Retrieve it through the NGSI-LD temporal API, filtering by the ``fibonacci`` attribute and the ``datasetId`` (the ``goalId`` you noted earlier).

We show two ways of retrieving it: the full JSON representation, and a compact summary built with ``jq``.

Option 1: full JSON representation
""""""""""""""""""""""""""""""""""

Adapt the following request to your specific example by setting the ``datasetId`` to return the complete temporal representation, pretty-printed as JSON:

.. code-block:: bash

    curl -s -G "http://localhost:1026/ngsi-ld/v1/temporal/entities/urn:ngsi-ld:robot:1" \
      -H 'Accept: application/json' \
      --data-urlencode 'attrs=fibonacci' \
      --data-urlencode 'datasetId=<INTRODUCE_YOUR_DATA_SET_ID>' | jq

The response returns the ``fibonacci`` attribute as an **array of instances**, oldest first.
Each instance is one step of the goal, and carries the latest ``ddsActionStatus``, ``ddsActionFeedback`` and ``ddsActionResult`` known at that step.
Together they rebuild the complete timeline of the goal. Shortened, it looks like this:

.. code-block:: json

    {
      "id": "urn:ngsi-ld:robot:1",
      "type": "Robot",
      "fibonacci": [
        {
          "type": "Property",
          "value": { "order": 5 },
          "datasetId": "urn:goal:42dd44ac-a3fd-ddbb-4361-b32c36fd1478",
          "ddsActionStatus": {
            "type": "Property",
            "value": { "code": "accepted", "message": "Action goal accepted" }
          }
        },
        {
          "type": "Property",
          "value": { "order": 5 },
          "datasetId": "urn:goal:42dd44ac-a3fd-ddbb-4361-b32c36fd1478",
          "ddsActionStatus": {
            "type": "Property",
            "value": { "code": "accepted", "message": "Action goal accepted" }
          },
          "ddsActionFeedback": {
            "type": "Property",
            "value": { "sequence": [ 0, 1, 1 ] }
          }
        },
        {
          "type": "Property",
          "value": { "order": 5 },
          "datasetId": "urn:goal:42dd44ac-a3fd-ddbb-4361-b32c36fd1478",
          "ddsActionStatus": {
            "type": "Property",
            "value": { "code": "succeeded", "message": "The goal was achieved successfully by the action server" }
          },
          "ddsActionFeedback": {
            "type": "Property",
            "value": { "sequence": [ 0, 1, 1, 2, 3, 5 ] }
          },
          "ddsActionResult": {
            "type": "Property",
            "value": { "result": { "sequence": [ 0, 1, 1, 2, 3, 5 ] }, "status": 4 }
          }
        }
      ]
    }

Option 2: compact summary with jq
"""""""""""""""""""""""""""""""""

If you only care about the values, pipe the response through ``jq`` to print one concise line per instance.
Remember to adapt the command by setting the ``datasetId`` of the action you want to inspect.

.. code-block:: bash

    curl -s -G "http://localhost:1026/ngsi-ld/v1/temporal/entities/urn:ngsi-ld:robot:1" \
      -H 'Accept: application/json' \
      --data-urlencode 'attrs=fibonacci' \
      --data-urlencode 'datasetId=<INTRODUCE_YOUR_DATA_SET_ID>' \
      | jq -r '.fibonacci[] | select(.ddsActionStatus) |
          [ "status: " + .ddsActionStatus.value.code,
            (if .ddsActionFeedback then "feedback: " + (.ddsActionFeedback.value.sequence|tostring) else empty end),
            (if .ddsActionResult   then "result: "   + (.ddsActionResult.value.result.sequence|tostring) else empty end)
          ] | join("  |  ")'

The ``jq`` filter prints one line per instance, with the status and, when present, the feedback and the result.
The output is the goal's timeline in a compact, readable form:

.. code-block:: text

    status: executing
    status: accepted
    status: accepted  |  feedback: [0,1,1]
    status: accepted  |  feedback: [0,1,1,2]
    status: accepted  |  feedback: [0,1,1,2,3]
    status: accepted  |  feedback: [0,1,1,2,3,5]
    status: succeeded  |  feedback: [0,1,1,2,3,5]
    status: succeeded  |  feedback: [0,1,1,2,3,5]  |  result: [0,1,1,2,3,5]

.. note::

    Because the ``goalId`` is part of the URI, use ``-G`` together with ``--data-urlencode`` so that curl encodes the parameters correctly.

    You can also limit the number of instances returned per attribute with ``--data-urlencode 'lastN=20'``.

.. note::

    Retrieving the temporal representation of a single entity by its id (as above) does not require the ``timerel`` parameter.
    If instead you query by type (``/ngsi-ld/v1/temporal/entities?type=Robot``), you must add a temporal filter, for example ``--data-urlencode 'timerel=after' --data-urlencode 'timeAt=1970-01-01T00:00:00Z'`` to retrieve everything.

Troubleshooting
---------------

If the action goal does not reach the ROS 2 action server, check the following points:

- Make sure the ``fibonacci`` action server is running.
- Make sure coraine and the ROS 2 container are using the same DDS domain (``domain`` in the configuration file and ``ROS_DOMAIN_ID`` in ROS 2).
- Make sure both containers use the host network and the host IPC namespace (``--net=host --ipc=host``, or ``network_mode: host`` and ``ipc: host``).
- Make sure the action name in the configuration file is ``fibonacci``.

- Check that ROS 2 can see the action, and that it has one server, with:

  .. code-block:: bash

      ros2 action list
      ros2 action info /fibonacci

- Check what the broker sees on the ROS 2 network:

  .. code-block:: bash

      curl -s http://localhost:1026/admin/bridges/dds/inventory | jq '.actions[].name'

If you do not receive notifications on your endpoint:

- Make sure the ``notify_listener.py`` server is running and listening on port ``7000``.
- Make sure the ``endpoint`` value in the goal points to a host and port reachable from the coraine container (``http://localhost:7000/notify`` works when coraine runs with ``network_mode: host``).

If the historical query returns ``Entity Not Found``:

- Make sure the broker was started with the ``--troe corDB`` option.
- Remember that only changes made **while TRoE was enabled** appear in temporal queries.

Conclusion
----------

In this tutorial, we called a ROS 2 action from the coraine context broker.
coraine was configured as a ROS 2 action client through a DDS-to-NGSI-LD action mapping, a ROS 2 ``fibonacci`` action server was launched in a Vulcanexus container, and temporal storage was enabled to keep the history of each goal.

By following these steps, you were able to:

- Configure coraine with an NGSI-LD mapping for a ROS 2 action and enable temporal storage.
- Launch a ROS 2 action server.
- Send an action goal through the NGSI-LD REST API.
- Follow the goal's feedback and status live through an HTTP notification endpoint.
- Observe how the goal's instance is removed from the current state once the goal completes.
- Retrieve the full history of the goal using its ``goalId``.

You can extend this example by adding more ROS 2 actions to the ``actions`` section of the ``context_broker_action_config.json`` file and mapping each one to a different NGSI-LD entity attribute.
