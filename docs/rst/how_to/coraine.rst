.. _coraine_how_to:

coraine: the ARISE Context Broker
=================================

`coraine <https://github.com/SEAMWARE/coraine>`__ is the NGSI-LD context broker of the ARISE framework.
It is developed by SEAMWARE and released under the Apache-2.0 license.

This guide shows how to run coraine, connect it to ROS 2, and choose a setup for your deployment.
For step-by-step examples, see the tutorials:

- :ref:`ros2_fiware_basic_app`: ROS 2 topics.
- :ref:`ros2_fiware_services_app`: ROS 2 services.
- :ref:`ros2_fiware_actions_app`: ROS 2 actions and their history.
- :ref:`dds_ngsild_mapper_tutorial`: generate the mapping file instead of writing it by hand.

Why coraine in ARISE
--------------------

- **Standard API.** coraine implements the ETSI NGSI-LD API. Any NGSI-LD client, dashboard or FIWARE component can use it.
- **ROS 2 built in.** Its DDS bridge, based on eProsima's DDS Enabler, maps ROS 2 topics, services and actions to NGSI-LD attributes, in both directions.
- **Plugin-driven.** Databases, history, APIs and bridges (DDS, MQTT, OPC UA, Modbus and more) are plugins. You load only what you need.
- **No external database needed.** Its built-in database, corDB, keeps the current state and the history in the broker itself. MongoDB and TimescaleDB remain available for larger deployments.
- **The entity mirrors the wire.** A value is sent to ROS 2 first and stored only if ROS 2 accepted it. A slow robot never makes the broker slow.

Run coraine in one command
--------------------------

Start a broker with its built-in database:

.. code-block:: bash

    docker run --rm -p 1026:1026 quay.io/seamware/coraine:0.7.0 --database corDB

Create an entity and read it back:

.. code-block:: bash

    curl -X POST localhost:1026/ngsi-ld/v1/entities \
         -H 'Content-Type: application/json' \
         -d '{
               "id":   "urn:ngsi-ld:Robot:1",
               "type": "Robot",
               "batteryLevel": { "type": "Property", "value": 78 }
             }'

    curl localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Robot:1

.. note::

    coraine images have no ``latest`` tag. Each tag is immutable, so a deployment always runs a version you can name.
    Pick a tag from `quay.io/seamware/coraine <https://quay.io/repository/seamware/coraine>`__ and pin it.
    The images are available for ``amd64`` and ``arm64``, but the ``arm64`` image does not include the DDS bridge.

coraine is also available as a Debian package for Ubuntu 24.04, Ubuntu 26.04 and Debian 13.
See `Install with apt <https://github.com/SEAMWARE/coraine#install-with-apt>`__.
The apt package does not include the DDS bridge either. For ROS 2, use the ``amd64`` container image.

Connect coraine to ROS 2
------------------------

The DDS bridge is part of the ``amd64`` image. It is loaded only when you ask for it, with two options:

- ``--bridges dds`` loads the bridge.
- ``--bridgeConfig <file>`` gives it your mapping file.

A minimal ``docker-compose.yml``:

.. code-block:: yaml

    services:
      coraine:
        image: quay.io/seamware/coraine:0.7.0
        network_mode: host
        ipc: host
        restart: unless-stopped
        command: [ "--database", "corDB", "--dbDir", "/data",
                   "--troe", "corDB",
                   "--apiPlugins", "admin",
                   "--bridges", "dds", "--bridgeConfig", "/opt/seamware/etc/dds.json" ]
        volumes:
          - ./dds.json:/opt/seamware/etc/dds.json:ro
          - coraine_data:/data
          - dds_types:/opt/seamware/etc/dds-types

    volumes:
      coraine_data:
      dds_types:

The ``dds_types`` volume keeps the message types the broker learns from the ROS 2 network.
After a restart, the broker does not have to wait for every node to announce its types again.

The mapping file
^^^^^^^^^^^^^^^^

One file holds the DDS settings (``ddsmodule``) and the mapping to NGSI-LD (``ngsild``):

.. code-block:: json

    {
      "dds": {
        "ddsmodule": { "dds": { "domain": 0 } },
        "ngsild": {
          "topics": {
            "rt/chatter": { "entityType": "Robot", "entityId": "urn:ngsi-ld:robot:1", "attribute": "chatter" }
          },
          "services": {
            "add_two_ints": {
              "entityType": "Robot", "entityId": "urn:ngsi-ld:robot:1", "attribute": "add_two_ints",
              "requestType": "example_interfaces::srv::dds_::AddTwoInts_Request_",
              "replyType":   "example_interfaces::srv::dds_::AddTwoInts_Response_"
            }
          },
          "actions": {
            "fibonacci": {
              "entityType": "Robot", "entityId": "urn:ngsi-ld:robot:1", "attribute": "fibonacci",
              "type": "example_interfaces::action::dds_::Fibonacci"
            }
          }
        }
      }
    }

What each mapping does:

.. list-table::
   :header-rows: 1
   :widths: 15 40 45

   * - Kind
     - Reading the attribute
     - Writing the attribute
   * - Topic
     - The last sample published on the topic.
     - Publishes a sample on the topic.
   * - Service
     - The last request (``value``) and its ``reply`` sub-property.
     - Calls the service. Add ``?ddsSync=true`` to wait briefly for the reply.
   * - Action
     - One instance per running goal (``datasetId`` ``urn:goal:<id>``), with its feedback, status and result.
     - Sends a goal. ``DELETE`` the goal's instance to cancel it.

The main settings of ``ddsmodule.dds``:

.. list-table::
   :header-rows: 1
   :widths: 30 45 25

   * - Setting
     - Meaning
     - Default
   * - ``domain``
     - The DDS domain. It must match ``ROS_DOMAIN_ID``.
     - ``0``
   * - ``transport``
     - ``builtin`` (shared memory and UDP), ``udp`` or ``shm``.
     - ``builtin``
   * - ``whitelist-interfaces``
     - IPv4 addresses of the interfaces UDP may use.
     - all interfaces
   * - ``allowlist``, ``blocklist``
     - Topics to include or exclude, with ``*`` and ``?`` wildcards.
     - everything

Every mapped attribute exists from startup, with the value ``"uninitialized"`` until the first sample arrives.
Changes to ``topics``, ``allowlist`` and ``blocklist`` are reloaded while the broker runs.
The other settings are read once, at start.

Find out what is on the ROS 2 network
-------------------------------------

You do not need to know the topics in advance. There are two ways to see what the broker has found.

**The default entity.** Topics that no mapping claims are stored on ``urn:ngsi-ld:dds:default``, one attribute per topic:

.. code-block:: bash

    curl -s localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:dds:default | jq

**The inventory.** With ``--apiPlugins admin``, the broker lists every topic, service and action on the network.
It also gives each one's type, with a sample message:

.. code-block:: bash

    curl -s localhost:1026/admin/bridges/dds/inventory | jq '{topics: [.topics[].name], services: [.services[].name], actions: [.actions[].name]}'

Use the inventory to write your mapping file: it gives you the exact DDS type names that ``requestType``, ``replyType`` and ``type`` need.
To generate the mapping for many topics at once, use the DDS NGSI-LD Mapper (see :ref:`dds_ngsild_mapper_tutorial`).

Choose where data is stored
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - Need
     - Current state (``--database``)
     - History (``--troe``)
   * - Tutorials, demos, a single robot cell
     - ``corDB`` (with ``--dbDir`` to keep it on disk)
     - ``corDB``
   * - No history needed
     - ``corDB`` or ``mongoc``
     - ``none`` (the default)
   * - Long-term history, SQL analytics, Grafana
     - ``corDB`` or ``mongoc``
     - ``timescale`` (PostgreSQL/TimescaleDB)

With ``--troe``, coraine serves the NGSI-LD temporal API (``/ngsi-ld/v1/temporal/...``) itself. No separate temporal service is needed.

Coming from Orion-LD
--------------------

coraine reads the same DDS configuration file as Orion-LD, including ``syncTimeoutMs``.
To move a deployment:

#. Mount the file and pass it with ``--bridgeConfig`` instead of mounting it on ``/root/.orionld``.
#. Replace ``-wip dds`` with ``--bridges dds``.
#. Replace ``-troe`` and the ``ORIONLD_TROE_*`` variables with ``--troe corDB`` or ``--troe timescale``.
#. Keep ``-duc`` (``--defaultUserContext``) if you use a default ``@context``.
#. Check your clients against the differences below.

.. list-table::
   :header-rows: 1
   :widths: 34 33 33

   * - Situation
     - Orion-LD
     - coraine
   * - A value that does not fit the topic's type
     - Stored, ``204``. DDS never sees it.
     - Refused, ``400``. Nothing is stored.
   * - Writing to DDS
     - Stored first, then published.
     - Published first, stored only if DDS took it.
   * - A slow service
     - The write waits 5 s, then ``504``. Nothing is stored.
     - ``202`` at once (or a short wait with ``?ddsSync=true``). The reply is written when it arrives.
   * - Discovered services
     - Visible.
     - Visible and callable.
   * - After an action's last goal
     - The whole attribute is removed.
     - The attribute stays, without the goals.
   * - ROS names with a ``/``
     - —
     - ``robot1/navigate`` becomes the attribute ``robot1_navigate``.
   * - One write, several attributes, one refused by DDS
     - The whole write fails.
     - The others are written. ``207`` names the refused one.

The ``reply``, ``ddsActionFeedback``, ``ddsActionStatus`` and ``ddsActionResult`` sub-properties keep Orion-LD's names, so existing clients keep working.
This representation is provisional: it will follow ETSI's upcoming NGSI-LD Service Execution when that is published.

Before going to production
--------------------------

- **Security.** coraine has no authentication until you configure it. Do not expose port ``1026`` to an untrusted network.
  See `Authorization <https://github.com/SEAMWARE/coraine/blob/main/doc/authorization.md>`__.
- **Pin the image tag** you tested.
- **Use** ``amd64`` **for ROS 2.** The ``arm64`` image and the apt package do not include the DDS bridge.
- **Use** ``ipc: host``. Fast DDS uses shared memory between processes on the same machine. Without it, the broker finds no ROS 2 node, and logs no error.
- **Run ROS 2 nodes and coraine as the same user** when they share memory. Otherwise the nodes are discovered but their samples never arrive. The coraine image and the Vulcanexus image both run as ``root``.
- **Stay on one network segment.** DDS discovery uses UDP multicast. It does not cross a router, a VPN or a NAT. A firewall that drops multicast blocks nodes on other machines.
- **Same domain.** A broker and a node on different DDS domains do not fail; they never see each other.

Resources
---------

- coraine on GitHub: https://github.com/SEAMWARE/coraine
- DDS and ROS 2 in coraine: https://github.com/SEAMWARE/coraine/blob/main/doc/dds.md
- DDS bridge configuration: https://github.com/SEAMWARE-cor/corDdsBridge
- Container images: https://quay.io/repository/seamware/coraine
- c³, a visual console for coraine: https://github.com/SEAMWARE/c3
- NGSI-LD: https://ngsi-ld.org/
