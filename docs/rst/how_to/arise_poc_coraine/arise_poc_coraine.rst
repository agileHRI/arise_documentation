.. _arise_poc_coraine:

Run the ARISE PoC with coraine
==============================

This guide runs the ARISE Proof of Concept with :ref:`coraine <coraine_how_to>`, the ARISE context broker.
Three ROS 2 turtles move on screen. coraine turns each turtle into an NGSI-LD entity and keeps its history.
An OPC UA machine, a press, sends its values through the FIWARE IoT Agent.
Grafana shows live and historical data, and the coraine Control Center (c³) shows the entities as a live graph, with alarms.

.. note::

    This guide covers the ROS 2 part and the OPC UA part of the original :ref:`ARISE PoC Guide <how_to_arise_poc>`.
    For now, OPC UA goes through the FIWARE IoT Agent; it will move to coraine's own OPC UA bridge.

What you will run
-----------------

.. code-block:: text

    turtlesim (ROS 2) ──DDS──────────────►  coraine  ──►  TimescaleDB  ──SQL──────►  Grafana
    OPC UA server ──► FIWARE IoT Agent ──►     │                                     ▲
                                               ├──── NGSI-LD temporal API ───────────┘
                                               │
                                               └──── NGSI-LD API + live notifications ──►  c³

- **turtlesim**: eProsima's keyed turtlesim demo, on Vulcanexus (ROS 2 Jazzy). Three turtles share the ``/pose`` and ``/cmd_vel`` topics; each message carries a ``turtle_id``.
- **coraine**: makes one ``Turtle`` entity per ``turtle_id`` and keeps every change in its temporal history (TRoE), stored in TimescaleDB.
- **TimescaleDB**: the PostgreSQL database for the history.
- **Grafana**: two dashboards. One reads the history with SQL; the other reads it through coraine's NGSI-LD temporal API.
- **OPC UA server**: Microsoft's OPC PLC simulator, whose values change on their own.
- **FIWARE IoT Agent for OPC UA**: reads the OPC UA values and writes them to coraine as the ``Machine`` entity ``urn:ngsi-ld:Machine:press-1``.
- **c³**: the coraine Control Center. It shows the entities as a graph that updates live, with the history of each value and alarms.

The entities form a small model of the robot cell:

.. code-block:: text

    RobotCell  urn:ngsi-ld:RobotCell:arise-poc
        ▲ inCell
    Arena      urn:ngsi-ld:Arena:turtlesim
        ▲ inArena
    Turtle     urn:ngsi-ld:Turtle:1   urn:ngsi-ld:Turtle:2   urn:ngsi-ld:Turtle:3
               x, y, theta, linearVelocity, angularVelocity, linearCommand, angularCommand

Differences with the original PoC
---------------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - Topic
     - Original PoC
     - This guide
   * - Context broker
     - Orion-LD with MongoDB
     - coraine, with its built-in database (corDB)
   * - Entities
     - One ``Robot`` entity; the three turtles share one attribute
     - One ``Turtle`` entity per turtle, linked to an ``Arena`` and a ``RobotCell``
   * - Values
     - One JSON object per pose
     - ``x``, ``y``, ``theta``… as separate numbers
   * - History API
     - Mintaka (a separate service)
     - coraine's own NGSI-LD temporal API
   * - SQL dashboards
     - Orion-LD's tables (``attributes.compound``)
     - coraine's tables (``troe_attrs``)
   * - Time range of the dashboards
     - Fixed dates in the queries
     - Grafana's time picker
   * - Panels per turtle
     - Nine copies of each panel
     - One set of panels, with a turtle selector
   * - Live view of the entities
     - —
     - c³, as a graph, with alarms

Prerequisites
-------------

- Linux on ``amd64``, with Docker and Docker Compose v2.
- ``git`` and ``curl``.
- About 10 GB of free disk space for the images.
- Optional: an X11 desktop, to see the turtles on screen.
- Optional: c³, for the live graph. It is included in the ``coraine`` Debian package; see `Install with apt <https://github.com/SEAMWARE/coraine#install-with-apt>`__.

Step 1 - Get the files
----------------------

The files of this guide are in their own repository, `agileHRI/arise_poc_coraine <https://github.com/agileHRI/arise_poc_coraine>`__:

.. code-block:: bash

    git clone --branch v0.2.0 https://github.com/agileHRI/arise_poc_coraine.git
    cd arise_poc_coraine

This guide is written for version ``v0.2.0`` of the repository. Its `CHANGELOG <https://github.com/agileHRI/arise_poc_coraine/blob/v0.1.0/CHANGELOG.md>`__ lists what each version contains.

It contains:

.. code-block:: text

    arise_poc_coraine/
    ├── docker-compose.yaml          # timescale, coraine, seed, opcplc, iot-agent, iot-provision,
    │                                # c3-descriptor, ros2, grafana
    ├── dds.json                     # ROS 2 topics → Turtle entities
    ├── seed/arena.json              # the robot cell and the arena, created once
    ├── seed/cell-opcua.json         # the robot cell again, in the tenant of the OPC UA part
    ├── seed/press-device.json       # the IoT Agent provisioning: OPC UA nodes → press attributes
    ├── c3/c3.json                   # the c³ demo descriptor (turtles)
    ├── c3/c3-opcua.json             # the c³ demo descriptor (press)
    ├── ros2/Dockerfile              # Vulcanexus + keyed turtlesim
    └── grafana/
        ├── provisioning/            # data sources, dashboard provider, alert rules
        └── dashboards/              # the two dashboards

The repository is licensed under the Apache License 2.0, as coraine.

Step 2 - Understand the configuration
-------------------------------------

The mapping file
^^^^^^^^^^^^^^^^

``dds.json`` tells coraine how to turn the ROS 2 messages into entities:

.. code-block:: json

   {
     "dds": {
       "ddsmodule": {
         "dds": {
           "domain": 0
         }
       },
       "ngsild": {
         "defaultEntity": false,
         "dataChannels": {
           "rt/pose": {
             "entities": [
               {
                 "type": "Turtle",
                 "id": "urn:ngsi-ld:Turtle:{turtle_id}",
                 "attributes": {
                   "x": {
                     "column": "x",
                     "as": "number"
                   },
                   "y": {
                     "column": "y",
                     "as": "number"
                   },
                   "theta": {
                     "column": "theta",
                     "as": "number"
                   },
                   "linearVelocity": {
                     "column": "linear_velocity",
                     "as": "number"
                   },
                   "angularVelocity": {
                     "column": "angular_velocity",
                     "as": "number"
                   },
                   "inArena": {
                     "relationship": "urn:ngsi-ld:Arena:turtlesim"
                   }
                 }
               }
             ]
           },
           "rt/cmd_vel": {
             "entities": [
               {
                 "type": "Turtle",
                 "id": "urn:ngsi-ld:Turtle:{turtle_id}",
                 "attributes": {
                   "linearCommand": {
                     "json": "linear"
                   },
                   "angularCommand": {
                     "json": "angular"
                   }
                 }
               }
             ]
           }
         }
       }
     }
   }

It uses **DataChannels**: each message on a topic is a record, and the record becomes an entity.

- ``"id": "urn:ngsi-ld:Turtle:{turtle_id}"`` builds the entity id from the ``turtle_id`` field of the message. The three turtles share one topic, but each one gets its own entity.
- ``{ "column": "x", "as": "number" }`` copies the ``x`` field into an ``x`` Property, as a number. So do ``y``, ``theta`` and the velocities.
- ``{ "relationship": "urn:ngsi-ld:Arena:turtlesim" }`` adds the ``inArena`` Relationship to every turtle.
- On ``rt/cmd_vel``, ``{ "json": "linear" }`` keeps the ``linear`` part of the command, ``{"x": …, "y": …, "z": …}``, as a ``JsonProperty``.
- ``"defaultEntity": false`` tells coraine to ignore every other topic. Without it, ROS 2 housekeeping topics such as ``rosout`` would fill the history.

The robot cell and the arena do not change, so they are not in the mapping: the ``seed`` service creates them once, from ``seed/arena.json``.

.. note::

    A value that is the same in every message (such as the arena) belongs in the seed, not in the mapping.
    In the mapping, it would be written again with every message, and each write would add a line to the history.

The services
^^^^^^^^^^^^

``docker-compose.yaml`` starts these services:

- ``timescale``: TimescaleDB, with the user ``coraine``.
- ``coraine``: the broker, with:

  - ``--database corDB``: the current state, in coraine's built-in database.
  - ``--troe timescale`` and ``--troeHost``, ``--troeUser``, ``--troeName arise``: the history, in the ``arise`` database of TimescaleDB. coraine creates the database and its tables on first use. The password is passed in the ``CORAINE_TROEPWD`` variable, so it does not appear on the command line.
  - ``--bridges dds`` and ``--bridgeConfig``: the DDS bridge with ``dds.json``.
  - ``--transports ws``: the WebSocket transport, which c³ uses for live updates.
  - ``ipc: host``: required, so that coraine can discover the ROS 2 nodes through shared memory.

- ``seed``: creates the robot cell and the arena when coraine is ready, then stops.
- ``c3-descriptor``: serves ``c3/c3.json`` on port ``7996``, for c³.
- ``ros2``: Vulcanexus with the keyed turtlesim demo, built from ``ros2/Dockerfile``. It also uses ``ipc: host``.
- ``grafana``: Grafana 13 with the Infinity plugin, the two data sources and the two dashboards.

.. warning::

    This setup is for a local demo. The passwords are the defaults (``coraine`` for the database, ``admin`` / ``admin`` for Grafana), and coraine has no authentication.
    Do not expose these ports to an untrusted network.

Step 3 - Start the services
---------------------------

.. code-block:: bash

    docker compose up -d --build

The first run builds the turtlesim image and downloads the other images. It can take several minutes.

Check that coraine is running, and that the robot cell and the arena exist:

.. code-block:: bash

    curl -s localhost:1026/version
    curl -s "localhost:1026/ngsi-ld/v1/entities?type=Arena,RobotCell"

The ``Turtle`` entities appear when the turtles start to move (Step 5).

Step 4 - Start the turtles
--------------------------

**With a desktop**, allow the container to open a window, then start the turtles:

.. code-block:: bash

    xhost +local:docker
    docker compose exec ros2 bash -ic "ros2 run docs_turtlesim turtlesim_node_keys"

A window opens with three turtles.

.. note::

    ``xhost +local:docker`` lets local containers use your display. Run ``xhost -local:docker`` when you finish.

**Without a desktop** (for example on a server), run the turtles without a window:

.. code-block:: bash

    docker compose exec ros2 bash -ic "QT_QPA_PLATFORM=offscreen ros2 run docs_turtlesim turtlesim_node_keys"

Leave this terminal open.

Step 5 - Move the turtles
-------------------------

In a second terminal, start the keyboard controller:

.. code-block:: bash

    docker compose exec ros2 bash -ic "ros2 run docs_turtlesim turtlesim_multi_control"

- Press ``1``, ``2`` or ``3`` to choose a turtle.
- Use the arrow keys to move it: up and down to go forward and back, left and right to turn.
- Press ``q`` to quit.

A turtle publishes its pose only while it moves.

Step 6 - Check the data in coraine
----------------------------------

Each turtle that has moved is an entity:

.. code-block:: bash

    curl -s "localhost:1026/ngsi-ld/v1/entities?type=Turtle"

The history of each value is available through the NGSI-LD temporal API. For example, the last five ``x`` positions of turtle 1:

.. code-block:: bash

    curl -s -G localhost:1026/ngsi-ld/v1/temporal/entities/urn:ngsi-ld:Turtle:1 \
      --data-urlencode 'attrs=x' \
      --data-urlencode 'lastN=5'

To see how many messages each DataChannel has turned into entities:

.. code-block:: bash

    curl -s localhost:1026/ngsi-ld/v1/dataChannels

Step 7 - Open the dashboards
----------------------------

Open http://localhost:3000 and log in with ``admin`` / ``admin``. Grafana asks you to set a new password.
The dashboards are in the **ARISE PoC** folder. Both have a **Turtle** selector at the top.

ARISE PoC - Turtles (live, SQL)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It refreshes every 5 seconds and shows the last 15 minutes:

- the position (``x``, ``y``) and the heading (``theta``) of the selected turtle;
- the linear and angular velocity commands sent to it;
- a table with the current position of every turtle, read from coraine's NGSI-LD API.

ARISE PoC - Turtles (history, NGSI-LD temporal API)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It shows the same data, read through coraine's temporal API instead of SQL.
Use Grafana's time picker to choose the period.

The two dashboards should show the same values. They read the same history in two different ways.

Step 8 - See the PoC in c³
--------------------------

c³ can open this PoC as a **named demo**: you type ``arise`` instead of a broker address, and c³ reads the descriptor ``c3/c3.json``:

.. code-block:: json

   {
     "name": "ARISE PoC",
     "broker": "http://127.0.0.1:1026",
     "tenant": null,
     "view": { "mode": "graph", "relationships": "all" },
     "watch": ["Turtle", "Arena", "RobotCell"],
     "alerts": [
       { "id": "turtle-east-wall",  "title": "Turtle near the east wall",  "severity": "warning", "type": "Turtle", "q": "x>10" },
       { "id": "turtle-west-wall",  "title": "Turtle near the west wall",  "severity": "warning", "type": "Turtle", "q": "x<1" },
       { "id": "turtle-north-wall", "title": "Turtle near the north wall", "severity": "warning", "type": "Turtle", "q": "y>10" },
       { "id": "turtle-south-wall", "title": "Turtle near the south wall", "severity": "warning", "type": "Turtle", "q": "y<1" }
     ]
   }

The descriptor gives the broker, the graph view, the types to load and four alarm rules: one for each wall of the arena.

#. Add the demo to c³'s configuration file, ``~/.config/c3/c3.json``, in its ``demos`` section (create the section if it is not there):

   .. code-block:: json

       "demos": {
         "arise": { "url": "http://127.0.0.1:7996/c3.json", "hint": "start it with docker compose up -d" }
       }

   Edit the file while c³ is not running: c³ writes it as you work.

#. Start c³:

   .. code-block:: bash

       c3

#. In **Explore**, type ``arise`` in the broker field and press **Connect**.

The graph shows the robot cell, the arena and the three turtles, joined by ``inCell`` and ``inArena``.
**Live · WebSocket** next to the counts means that the graph follows the broker as it changes.

- **Click a turtle** to see its attributes. They change as the turtle moves. Its **History** shows each number (``x``, ``y``, ``theta``…) over the last 10 minutes.
- **Drive a turtle against a wall** (Step 5): a *Turtle near the … wall* alarm appears, and **Alarms** counts it. The alarm clears when the turtle moves away.
- **Rules…** in **Alarms** shows the four rules. You can add your own, for example ``linearVelocity>1``.

Step 9 - Alerts
---------------

The PoC raises the same situation, a turtle less than 1 m from a wall of the arena, in two places:

- **c³** - the four alarm rules of the descriptor (Step 8), for whoever is watching the graph.
- **Grafana** - the alert rule *Turtle near a wall*, which can notify people outside c³, for example in Telegram.

The Grafana rule
^^^^^^^^^^^^^^^^

The rule is in ``grafana/provisioning/alerting/arise-rules.yaml`` and is loaded when Grafana starts.
Every 10 seconds it reads the last position of each turtle from coraine's history and computes its distance to the nearest wall:

.. code-block:: text

    WITH last AS (
      SELECT DISTINCT ON (a.entity_id, a.attr_name) a.entity_id, a.attr_name, a.v_number
      FROM troe_attrs a
      WHERE a.entity_id LIKE 'urn:ngsi-ld:Turtle:%'
        AND a.attr_name IN ('https://uri.etsi.org/ngsi-ld/default-context/x',
                            'https://uri.etsi.org/ngsi-ld/default-context/y')
      ORDER BY a.entity_id, a.attr_name, a.modified_at DESC
    )
    SELECT
      replace(entity_id, 'urn:ngsi-ld:', '') AS turtle,
      LEAST(
        MAX(v_number) FILTER (WHERE attr_name LIKE '%/x'),
        11.09 - MAX(v_number) FILTER (WHERE attr_name LIKE '%/x'),
        MAX(v_number) FILTER (WHERE attr_name LIKE '%/y'),
        11.09 - MAX(v_number) FILTER (WHERE attr_name LIKE '%/y')
      ) AS wall_distance
    FROM last
    GROUP BY entity_id

A distance below ``1`` fires the alert. There is one alert per turtle, labelled ``turtle`` (for example ``Turtle:1``).

To see it, drive a turtle against a wall (Step 5), then open **Alerting → Alert rules** in Grafana: *Turtle near a wall* is **Firing** for that turtle.
Its description says how far the turtle is from the wall, for example *Turtle:1 is 0.00 m from the nearest wall of the arena*.

Notifications in Telegram (optional)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Without a contact point, Grafana shows the alerts but sends them nowhere. To receive them in a Telegram chat:

#. Create a bot with `@BotFather <https://t.me/BotFather>`__ and keep its token. Get the id of the chat that will receive the alerts (group ids start with ``-100``).
#. Copy the example and write the two values in it, keeping the quotes:

   .. code-block:: bash

       cp grafana/provisioning/alerting/telegram.yaml.example grafana/provisioning/alerting/telegram.yaml

   .. code-block:: yaml

       bottoken: "<your bot token>"
       chatid: "<your chat id>"

#. Restart Grafana:

   .. code-block:: bash

       docker compose restart grafana

``telegram.yaml`` holds a secret: it is listed in ``.gitignore``, so it is not committed by mistake.

.. warning::

    Grafana does not start if a file in ``grafana/provisioning/`` is not valid.
    If http://localhost:3000 does not answer after a change, check ``docker compose logs grafana`` for ``Failed to provision``.

Step 10 - The OPC UA part
-------------------------

The press is a machine with an OPC UA server. Its values reach coraine through the **FIWARE IoT Agent for OPC UA**.

.. note::

    The IoT Agent is used **for now**. It will be replaced by coraine's own OPC UA bridge, which is not yet in the coraine image.

The services
^^^^^^^^^^^^

- ``opcplc``: Microsoft's `OPC PLC <https://github.com/Azure-Samples/iot-edge-opc-plc>`__ simulator, an OPC UA server whose values change on their own.
- ``iot-agent``: the FIWARE IoT Agent for OPC UA. It subscribes to the OPC UA values and writes each change to coraine with the NGSI-LD API. Its device registry is kept in memory, so no MongoDB is needed.
- ``iot-provision``: tells the IoT Agent, once, which OPC UA node becomes which attribute (``seed/press-device.json``).

The mapping
^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 30 25 45

   * - OPC UA node
     - Attribute
     - Behaviour in the simulator
   * - ``ns=3;s=StepUp``
     - ``cycleCount``
     - A counter that goes up every second.
   * - ``ns=3;s=AlternatingBoolean``
     - ``running``
     - Switches between ``true`` and ``false``.
   * - ``ns=3;s=PositiveTrendData``
     - ``temperature``
     - Starts at 100 and rises about 1.2 per minute.
   * - ``ns=3;s=SpikeData``
     - ``vibration``
     - A wave between -100 and 100, with frequent spikes to 1000.

The provisioning also adds ``name`` ("Press 1") and the Relationship ``inCell`` to the robot cell.

The tenant
^^^^^^^^^^

The IoT Agent always writes to a **named tenant**: its ``fiware-service``, here ``arise``.
The DDS bridge writes the turtles to the default tenant.
So the press lives in the tenant ``arise``, with its own copy of the robot cell (``seed/cell-opcua.json``).
To read it, send the tenant in the ``NGSILD-Tenant`` header:

.. code-block:: bash

    curl -s -H 'NGSILD-Tenant: arise' localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Machine:press-1

coraine keeps the history of each tenant in its own database: ``arise_arise`` for this one.

In Grafana and c³
^^^^^^^^^^^^^^^^^

- **Grafana**: the dashboard **ARISE PoC - Press (OPC UA)** shows the temperature, the vibration, the cycle count and whether the press is running, plus its current state.
  It uses two data sources for the tenant ``arise``: ``coraine-history-sql-arise`` (database ``arise_arise``) and ``coraine-api-arise`` (sends ``NGSILD-Tenant: arise``).
- **Alert**: the rule **Press temperature high** fires when the temperature goes above 103, about 3 minutes after the start. To reset the temperature, restart the simulator:

  .. code-block:: bash

      docker compose restart opcplc

  The temperature goes back to 100, the alert resolves, and the IoT Agent reconnects on its own.

- **c³**: add the second demo to ``~/.config/c3/c3.json``, next to ``arise``:

  .. code-block:: json

      "arise-opcua": { "url": "http://127.0.0.1:7996/c3-opcua.json", "hint": "start it with docker compose up -d" }

  Type ``arise-opcua`` in **Explore** and press **Connect**. c³ opens the tenant ``arise``: the press, linked to the robot cell, with its values live and the alarm *Press temperature high*.

Write your own panels
---------------------

With SQL
^^^^^^^^

coraine keeps one row per change of an attribute in the ``troe_attrs`` table. The main columns are:

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Column
     - Content
   * - ``entity_id``, ``entity_type``
     - The entity, for example ``urn:ngsi-ld:Turtle:1`` and ``Turtle``.
   * - ``attr_name``
     - The attribute, as a full name (see below).
   * - ``modified_at``, ``observed_at``
     - When the change was stored, and when the sample was observed.
   * - ``v_number``, ``v_text``, ``v_bool``
     - The value, when it is a number, a text or a boolean.
   * - ``v_compound``
     - The value, when it is a JSON object (``jsonb``), such as ``linearCommand``.
   * - ``dataset_id``
     - The instance, for example the goal of a ROS 2 action. Empty for ordinary values.

coraine stores attribute names in their full form: ``x`` is ``https://uri.etsi.org/ngsi-ld/default-context/x``.
This is the standard NGSI-LD expansion of names that have no ``@context`` of their own.

For example, the position of turtle 1, one series per attribute, with Grafana's time filter:

.. code-block:: text

    SELECT
      a.modified_at AS time,
      replace(a.attr_name, 'https://uri.etsi.org/ngsi-ld/default-context/', '') AS metric,
      a.v_number AS value
    FROM troe_attrs a
    WHERE a.entity_id = 'urn:ngsi-ld:Turtle:1'
      AND a.attr_name IN ('https://uri.etsi.org/ngsi-ld/default-context/x',
                          'https://uri.etsi.org/ngsi-ld/default-context/y')
      AND $__timeFilter(a.modified_at)
    ORDER BY 1

And the linear speed sent to it, from the ``linearCommand`` JSON object:

.. code-block:: text

    SELECT
      a.modified_at AS time,
      (a.v_compound->>'x')::real AS "linear x"
    FROM troe_attrs a
    WHERE a.entity_id = 'urn:ngsi-ld:Turtle:1'
      AND a.attr_name = 'https://uri.etsi.org/ngsi-ld/default-context/linearCommand'
      AND $__timeFilter(a.modified_at)
    ORDER BY 1

.. note::

    Queries written for the original PoC do not work with coraine: the tables, the columns and the entities are different.

With the NGSI-LD temporal API
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The ``coraine-api`` data source uses the Infinity plugin. A query needs:

- **URL**: the temporal API for one attribute, with Grafana's time range:

  .. code-block:: text

      http://coraine:1026/ngsi-ld/v1/temporal/entities/urn:ngsi-ld:Turtle:1?attrs=x&timerel=between&timeAt=${__from:date:iso}&endTimeAt=${__to:date:iso}

- **Parser**: ``Backend``.
- **Rows / root selector**: the attribute, for example ``x``.
- **Columns**: ``observedAt`` as the time, and ``value`` (``json.x`` for a ``JsonProperty`` such as ``linearCommand``).

Add one query per attribute to show several in one panel.

Stop the PoC
------------

.. code-block:: bash

    docker compose down

To also delete the stored data (current state, history and Grafana settings):

.. code-block:: bash

    docker compose down -v

Troubleshooting
---------------

- **No** ``Turtle`` **entity**: a turtle publishes only while it moves. Move one with the keyboard controller.
- **coraine sees no topic**: check that both ``coraine`` and ``ros2`` have ``ipc: host``. To list what coraine sees on the ROS 2 network:

  .. code-block:: bash

      curl -s localhost:1026/admin/bridges/dds/inventory

- **No turtle window**: run ``xhost +local:docker`` before starting the turtles, and check that ``DISPLAY`` is set in your terminal. You can also run them without a window (see Step 4).
- **Empty panels**: check the time range in Grafana, and the **Turtle** selector.
- **c³ says the demo is not running**: check that ``http://127.0.0.1:7996/c3.json`` answers.
- **Grafana does not start after a change**: a provisioning file is not valid. ``docker compose logs grafana`` names it (``Failed to provision``).
- **c³ shows Not live**: coraine needs ``--transports ws``. c³ runs outside Docker, so coraine cannot send it notifications over HTTP.

What comes next
---------------

This guide will grow until it replaces the original PoC guide:

- **OPC UA through coraine's own bridge**: when the coraine image includes it, it replaces the IoT Agent, and the press joins the turtles in the default tenant.
- **Commands from NGSI-LD**: driving the turtles by writing in coraine. This direction does not work yet with the keyed turtlesim demo.
- **Velocity commands as numbers**: DataChannels cannot read a nested field such as ``linear.x`` yet, so the commands are kept as JSON objects.
