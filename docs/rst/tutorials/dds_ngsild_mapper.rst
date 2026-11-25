.. _dds_ngsild_mapper_tutorial:

Generating the DDS Mapping with the DDS NGSI-LD Mapper
======================================================

Introduction
------------

The previous tutorials wrote the context broker's mapping file by hand.
That works for one topic or one service, but a real robot exposes dozens of topics, services and actions.
Writing the mapping for all of them by hand is slow and easy to get wrong.

The `DDS NGSI-LD Mapper <https://github.com/Engineering-Research-and-Development/dds-ngsild-mapper>`__ automates this work.
It reads the list of topics, services and actions found on a DDS / ROS 2 network, lets you decide what to expose and how to model it in NGSI-LD, and writes two files:

- ``dds-config.json``: the DDS bridge configuration, with the DDS settings (``ddsmodule``) and the mapping from each DDS endpoint to an NGSI-LD ``entityId``, ``entityType`` and ``attribute`` (``ngsild``).
- ``dds-context.jsonld``: a JSON-LD ``@context`` that expands the short entity type and attribute names used in the mapping into full IRIs.

.. mermaid::

    flowchart LR
        D[Discovery inventory<br/>file, HTTP or WebSocket] --> M[DDS NGSI-LD Mapper<br/>CLI, wizard or web UI]
        SDM[(Smart Data Models)] -.-> M
        M --> C1[dds-config.json]
        M --> C2[dds-context.jsonld]
        C1 --> B[Context broker<br/>DDS bridge]
        C2 --> B

By the end of this tutorial, you will be able to:

- Generate a mapping file automatically from a discovery inventory.
- Model the mapping in the web UI, grouping several ROS 2 topics into one NGSI-LD entity.
- Align the mapping with a published Smart Data Model.
- Use the generated files with the context broker.
- Update an existing mapping when the robot changes.

.. note::

    This tutorial builds on :ref:`ros2_fiware_basic_app`. We recommend that you complete it first, so that you know what the mapping file does.

Prerequisites
-------------

- `Node.js <https://nodejs.org/>`__ 18 or later, **or** Docker.
- ``git`` to download the mapper.
- Optionally, a running context broker and ROS 2 system, as in :ref:`ros2_fiware_basic_app`, to try the generated files.

Installing the Mapper
---------------------

Clone the repository and install its dependencies:

.. code-block:: bash

    git clone https://github.com/Engineering-Research-and-Development/dds-ngsild-mapper.git
    cd dds-ngsild-mapper
    npm install

The mapper has only two runtime dependencies and needs no build step.

.. note::

    To use Docker instead of Node.js, build the image once:

    .. code-block:: bash

        docker build -t dds-ngsi-mapper .

    The Docker commands are shown in :ref:`dds_ngsild_mapper_docker`.

The Discovery Inventory
-----------------------

The mapper starts from a **discovery inventory**: the list of topics, services and actions found on the DDS domain.
It is a JSON object with three arrays:

.. code-block:: json

    {
      "topics": [
        { "name": "rt/cmd_vel",       "typeName": "geometry_msgs/msg/Twist" },
        { "name": "rt/battery_state", "typeName": "sensor_msgs/msg/BatteryState" }
      ],
      "services": [
        { "name": "set_bool",
          "requestType": "std_srvs/srv/SetBool_Request",
          "replyType":   "std_srvs/srv/SetBool_Response" }
      ],
      "actions": [
        { "name": "navigate_to_pose",
          "goalType": "nav2_msgs/action/NavigateToPose_SendGoal_Request" }
      ]
    }

Only ``name`` is required for each entry. The type names are used to suggest meaningful NGSI-LD names.

The mapper can read the inventory from three sources:

- **A local file**, with ``--input <file>``.
- **An HTTP endpoint**, with ``--discovery-url http://...``. A single ``GET`` must return the whole object.
- **A WebSocket stream**, with ``--discovery-url ws://...``. The backend can send the whole object in one frame, or one entry per frame.

The repository ships an example inventory in ``examples/discovery.json``, with five topics, two services and two actions of a mobile robot.
We use it in this tutorial, so you can follow along without a robot.

.. tip::

    On a live system, the context broker can tell you what it sees on the ROS 2 network (see :ref:`coraine_how_to`).
    Save that list as a file and use it as the mapper's input, or point ``--discovery-url`` at a DDS Enabler discovery endpoint.

Generating a Mapping Automatically
----------------------------------

Run the mapper on the example inventory, without any question:

.. code-block:: bash

    node src/index.js --input examples/discovery.json --auto

The mapper prints a summary:

.. code-block:: text

    Discovery source: file  → examples/discovery.json
    Discovered: 5 topics, 2 services, 2 actions
    Auto-mapped: 8 entries

    Done — 8 entries mapped.
      Config  → out/dds-config.json
      Context → out/dds-context.jsonld

Eight entries are mapped, not nine: the ``rt/debug/log`` topic carries ROS 2 log messages (``rcl_interfaces/msg/Log``), so the mapper puts it in the ``blocklist`` instead.
Log topics such as ``/rosout`` produce a constant stream of messages that you almost never want in the context broker.
Use ``--keep-logs`` to map them anyway.

Open ``out/dds-config.json``. The ``ddsmodule`` section holds the DDS settings, and the ``ngsild`` section holds the mapping:

.. code-block:: json

    {
      "dds": {
        "ddsmodule": {
          "dds": {
            "domain": 0,
            "allowlist": [ { "name": "*" } ],
            "blocklist": [ { "name": "rt/debug/log" } ]
          }
        },
        "ngsild": {
          "topics": {
            "rt/cmd_vel": {
              "entityId": "urn:ngsi-ld:dds:default",
              "entityType": "DDS",
              "attribute": "cmd_vel"
            },
            "rt/battery_state": {
              "entityId": "urn:ngsi-ld:dds:default",
              "entityType": "DDS",
              "attribute": "battery_state"
            }
          }
        }
      }
    }

The example above is shortened. In ``--auto`` mode, every endpoint goes to the same entity, ``urn:ngsi-ld:dds:default``, with one attribute per endpoint.
The attribute name is the DDS name without the ``rt/`` prefix.
This keeps batch runs predictable, and it is a good starting point, but it is not a real data model.
The next section shows how to model the data properly.

The settings can also come from a ``.env`` file in the mapper folder, and every setting can be overridden on the command line.
The most useful options are:

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - Option
     - Meaning
   * - ``--input <file>``
     - Read the inventory from a local file.
   * - ``--discovery-url <url>``
     - Read the inventory from an ``http``, ``https``, ``ws`` or ``wss`` endpoint.
   * - ``--domain <n>``
     - The DDS domain. It must match ``ROS_DOMAIN_ID``.
   * - ``--iri-base <url>``
     - The base used to build the IRIs in the ``@context``.
   * - ``--out-config``, ``--out-context``
     - Where to write the two files (default ``out/``).
   * - ``--config``, ``--context``
     - Load existing files to update them (see :ref:`dds_ngsild_mapper_roundtrip`).
   * - ``--auto``
     - Map everything with the defaults, without questions.
   * - ``--keep-logs``
     - Do not blocklist ROS 2 log topics.

Run ``node src/index.js --help`` for the full list.

Modeling the Robot in the Web UI
--------------------------------

In NGSI-LD, a robot is usually **one entity** with one attribute per piece of information: its battery, its position, its velocity, and so on.
The web UI lets you build that model and see the result while you edit.

Start the web UI:

.. code-block:: bash

    npm run web

Open http://localhost:3000 in your browser. The page is organized in five steps.

1. Discovery source
^^^^^^^^^^^^^^^^^^^

Choose where the inventory comes from: a backend URL, a file upload, or pasted JSON.
Upload ``examples/discovery.json`` and load it.
Leave **Auto-blocklist log topics** enabled.

2. Global settings
^^^^^^^^^^^^^^^^^^

These settings come from ``.env``. Check at least:

- **DDS domain**: the same value as ``ROS_DOMAIN_ID`` on your robots.
- **IRI base**: the base for the IRIs of your own terms, for example ``https://example.org/dds/``.

3. Mapping
^^^^^^^^^^

Each discovered endpoint is a row. For each row, choose an action:

- ``map``: include it in the mapping, with the ``entityType``, ``entityId`` and ``attribute`` shown in the row.
- ``skip``: leave it out of the mapping. The bridge still sees it.
- ``blocklist``: exclude it from the bridge entirely.

Unlike ``--auto``, the web UI suggests a name for every row, based on its ROS 2 type.
For example, ``rt/cmd_vel`` (``geometry_msgs/msg/Twist``) gets the type ``Twist`` and the identifier ``urn:ngsi-ld:Twist:cmd_vel``.
The ↺ button restores the suggestion after an edit.

To model the robot as one entity, map the three topics below to the same ``entityId`` and ``entityType``, each with its own attribute:

.. list-table::
   :header-rows: 1
   :widths: 25 35 15 25

   * - DDS name
     - ``entityId``
     - ``entityType``
     - ``attribute``
   * - ``rt/battery_state``
     - ``urn:ngsi-ld:Robot:amr-01``
     - ``Robot``
     - ``battery``
   * - ``rt/pose``
     - ``urn:ngsi-ld:Robot:amr-01``
     - ``Robot``
     - ``pose``
   * - ``rt/cmd_vel``
     - ``urn:ngsi-ld:Robot:amr-01``
     - ``Robot``
     - ``velocityCommand``

Skip the endpoints you do not need, for example ``rt/scan`` if you do not want laser scans in the context broker.

4. Smart Data Models
^^^^^^^^^^^^^^^^^^^^

This step is optional. See :ref:`dds_ngsild_mapper_sdm` below.

5. Generate output
^^^^^^^^^^^^^^^^^^

The page shows a live preview of ``dds-config.json`` and ``dds-context.jsonld``.
You can copy them, download them, or save them to the ``out/`` folder.

The ``topics`` section of the mapping now describes one robot:

.. code-block:: json

    "topics": {
      "rt/battery_state": { "entityId": "urn:ngsi-ld:Robot:amr-01", "entityType": "Robot", "attribute": "battery" },
      "rt/pose":          { "entityId": "urn:ngsi-ld:Robot:amr-01", "entityType": "Robot", "attribute": "pose" },
      "rt/cmd_vel":       { "entityId": "urn:ngsi-ld:Robot:amr-01", "entityType": "Robot", "attribute": "velocityCommand" }
    }

And the ``@context`` gives each short name a full IRI:

.. code-block:: json

    {
      "@context": {
        "Robot":           "https://example.org/dds/Robot",
        "battery":         "https://example.org/dds/battery",
        "pose":            "https://example.org/dds/pose",
        "velocityCommand": "https://example.org/dds/velocityCommand"
      }
    }

Before writing anything, the mapper checks the mapping.
Generation fails, with a message that names the row, if:

- an ``entityId`` is not a valid URI, such as ``urn:ngsi-ld:Robot:amr-01``;
- an ``attribute`` or ``entityType`` contains characters that are not safe in an IRI (only letters, digits and ``. _ ~ -`` are allowed);
- two endpoints of the same kind write the same attribute of the same entity.

.. _dds_ngsild_mapper_sdm:

Aligning with Smart Data Models
-------------------------------

The names above work, but they are private to your project: another system cannot know that ``battery`` means the battery level of a device.
`Smart Data Models <https://smartdatamodels.org/>`__ is a FIWARE initiative that publishes common data models, with standard attribute names and IRIs.
Step 4 of the web UI lets you reuse one of them instead of inventing your own.

#. **Search the catalog.** Type a keyword, for example ``Device``, and pick a model.
   The page lists its attributes, with their type, unit, and whether they are required.
#. **Choose a target entity**, for example ``urn:ngsi-ld:Device:amr-01``.
#. **Review the suggestions.** For each endpoint, the mapper suggests a matching attribute of the model.
   It understands common ROS 2 abbreviations: ``battery_state`` is matched with ``batteryLevel``, and ``pose`` with ``location``.
   Every suggestion is a drop-down list that you can change.
#. **Apply to mapping.** The chosen rows of step 3 take the model's type and attribute names, and are tagged ``SDM``.

The ``@context`` then uses the model's own IRIs for these rows.
Rows that you did not align keep an IRI built from your IRI base, so you can adopt Smart Data Models one attribute at a time:

.. code-block:: json

    {
      "@context": {
        "Device":       "https://smartdatamodels.org/dataModel.Device/Device",
        "batteryLevel": "https://smartdatamodels.org/dataModel.Device/batteryLevel",
        "location":     "https://uri.etsi.org/ngsi-ld/location",
        "cmd_vel":      "https://example.org/dds/cmd_vel"
      }
    }

.. note::

    The mapper changes only the **names** used in NGSI-LD.
    The bridge still stores the ROS 2 message as it is published, so the value of ``batteryLevel`` is the whole ``BatteryState`` message, not a single number.
    The compatibility report in step 4 points out where the message and the model's attribute types differ.

The catalog and the schemas are downloaded from GitHub on first use and cached on disk, so the page keeps working offline.
On a machine without Internet access, set ``SDM_ENABLED=false`` in ``.env`` to hide the step.

Using the Terminal Wizard
-------------------------

If you prefer the terminal, run the mapper without ``--auto``:

.. code-block:: bash

    node src/index.js --input examples/discovery.json

It asks what to do with each endpoint:

.. code-block:: text

    [2/5]  rt/battery_state
      type : sensor_msgs/msg/BatteryState
      state: unmapped   (defaults: entityId=urn:ngsi-ld:dds:default  type=DDS  attr=battery_state)
      [m]ap/edit  [d]efaults  [s]kip  [b]locklist  [Enter]=keep  [q]uit >

Press ``m`` to enter the ``entityId``, ``entityType`` and ``attribute``, ``d`` to accept the defaults, ``s`` to skip, ``b`` to blocklist, or ``q`` to save what you have decided so far.
The wizard suggests the entity identifiers you have already used, so grouping several topics into one entity is quick.

Using the Generated Files with the Context Broker
-------------------------------------------------

``dds-config.json`` is the mapping file of the context broker's DDS bridge.
Use it in place of the ``context_broker_config.json`` file of :ref:`ros2_fiware_basic_app`: mount it in the container and pass it to the bridge, as shown in :ref:`coraine_how_to`.

.. code-block:: yaml

    volumes:
      - ./out/dds-config.json:/opt/seamware/etc/dds.json:ro

``dds-context.jsonld`` is optional. Give it to the broker as its default user ``@context`` if you want clients to use the short names (``battery``, ``Robot``) and get the full IRIs.
Otherwise, clients use the full IRIs, or send the ``@context`` with each request.

.. note::

    At the end of a run, the command-line tool prints a launch hint in the Orion-LD syntax (``orionld -wip dds --config ... -duc ...``).
    The file works the same way in coraine; see the "Coming from Orion-LD" part of :ref:`coraine_how_to` to translate the options.

.. important::

    For **topics**, the generated mapping is complete.
    For **services** and **actions**, the broker also needs the DDS type names (``requestType`` and ``replyType`` for a service, ``type`` for an action), which the mapper does not write yet.
    Add them by hand, as shown in :ref:`ros2_fiware_services_app` and :ref:`ros2_fiware_actions_app`.

Start the broker and your ROS 2 nodes, then read the entity:

.. code-block:: bash

    curl -s "http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Robot:amr-01?local=true" | jq

Each mapped topic appears as an attribute of the entity, with the last message published on it.

.. _dds_ngsild_mapper_roundtrip:

Updating an Existing Mapping
----------------------------

Robots change: a node is added, a topic is renamed.
You do not have to start over. Load the files you generated before, together with a fresh inventory:

.. code-block:: bash

    node src/index.js \
      --input examples/discovery.json \
      --config out/dds-config.json \
      --context out/dds-context.jsonld

The mapper keeps every decision you made before, and the DDS domain, types directory and IRI base saved in the files.
Only the new endpoints need a decision.
Endpoints that are in the old mapping but no longer in the inventory are kept, and marked ``(not in current discovery)``, so you can decide whether to remove them.

.. _dds_ngsild_mapper_docker:

Running the Mapper with Docker
------------------------------

The image runs the command-line tool. Mount a folder to get the generated files:

.. code-block:: bash

    docker run --rm \
      -v "$(pwd)/examples:/app/examples" \
      -v "$(pwd)/out:/app/out" \
      dds-ngsi-mapper --input examples/discovery.json --auto

To read the inventory from a live endpoint, pass the settings as environment variables:

.. code-block:: bash

    docker run --rm \
      -e DDS_DISCOVERY_URL=http://dds-backend:8080/api/discovery \
      -e NGSI_IRI_BASE=https://example.org/dds/ \
      -e MAPPER_MODE=auto \
      -v "$(pwd)/out:/app/out" \
      dds-ngsi-mapper

To run the web UI, override the entry point and publish its port:

.. code-block:: bash

    docker run --rm -p 3000:3000 \
      -v "$(pwd)/out:/app/out" \
      --entrypoint node dds-ngsi-mapper src/server.js

.. note::

    The terminal wizard needs an interactive terminal. In Docker, use ``--auto`` or the web UI.

Troubleshooting
---------------

- **No endpoint is discovered.** Check the discovery URL. With a WebSocket stream, the mapper stops reading once the stream is quiet for one second; raise ``--ws-quiet`` for slow backends.
- **Generation fails.** Read the message: it names the row and the rule. Usually an ``entityId`` is not a URI, or two endpoints write the same attribute of the same entity.
- **A topic is missing from the broker.** Check that it is mapped and not in the ``blocklist``, and that the ``domain`` in ``dds-config.json`` matches ``ROS_DOMAIN_ID``.
- **The Smart Data Models step is empty.** The catalog could not be downloaded and nothing is cached yet. Check the Internet connection, or paste the model's JSON schema by hand.

Conclusion
----------

In this tutorial, we generated the context broker's DDS mapping with the DDS NGSI-LD Mapper instead of writing it by hand.

By following these steps, you were able to:

- Generate a complete mapping from a discovery inventory in one command.
- Model a robot as one NGSI-LD entity in the web UI.
- Reuse standard names and IRIs from Smart Data Models.
- Use the generated files with the context broker.
- Update the mapping when the robot changes, without losing earlier decisions.

The `mapper's README <https://github.com/Engineering-Research-and-Development/dds-ngsild-mapper#readme>`__ documents every option, the input and output formats, and the web UI's API.
