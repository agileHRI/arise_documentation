.. _fiware_components:

FIWARE Components
=================

FIWARE provides the context-management layer of the ARISE Framework. In ARISE, FIWARE is used to make robotics, industrial systems, enterprise applications, and engineering tools interoperate through shared context information.

The main idea is straightforward: ROS 2 modules can continue to communicate through ROS 2 and DDS, while external applications can access selected robot, process, and environment information through standard NGSI-LD APIs. This makes robotics capabilities easier to integrate with IT and engineering systems without forcing every application to become a ROS 2 application.

FIWARE-based platforms help connect three technology worlds:

* **OT — Operational Technology**: robots, machines, sensors, actuators, PLCs, workcells, and shop-floor systems.
* **IT — Information Technology**: MES, ERP, dashboards, databases, cloud services, identity management, and business applications.
* **ET — Engineering Technology**: simulation, digital twins, AI/ML services, optimization tools, validation environments, and system engineering platforms.

ARISE uses FIWARE to support convergence between these worlds.


NGSI-LD Context Broker
----------------------

The NGSI-LD Context Broker is the central FIWARE component used in ARISE. It manages context information as NGSI-LD entities.

An entity represents something relevant in the system, for example:

* a robot,
* a workcell,
* a human operator,
* a robot task,
* a safety zone,
* a machine,
* a production operation,
* an observation from a perception module.

Each entity has attributes. These attributes can describe the current state of the entity, such as position, battery level, operational state, assigned task, detected person, quality result, or safety condition.

Applications can use the NGSI-LD API to:

* create entities,
* update context information,
* query the current state,
* subscribe to changes,
* receive notifications when something relevant happens.

This gives IT and engineering applications a standard access point to robotics and industrial context.


Connecting FIWARE and ROS 2
---------------------------

ROS 2 uses DDS as its communication middleware. DDS is well suited for distributed robotics because it provides publish-subscribe communication, discovery, message exchange, and configurable Quality of Service policies.

FIWARE and ROS 2 can be connected by mapping DDS data to NGSI-LD entities.

A simplified view is:

.. code-block:: text

   ROS 2 / DDS world                  FIWARE / NGSI-LD world

   ROS 2 topic sample  ────────────►  NGSI-LD entity attribute
   ROS 2 robot state   ────────────►  Robot entity
   ROS 2 perception    ────────────►  Observation or HumanDetection entity
   ROS 2 task status   ────────────►  RobotTask entity
   NGSI-LD task update ────────────►  ROS 2 topic, service, or action

This allows ROS 2 modules to remain ROS 2-compliant while becoming visible to systems that interact through HTTP APIs and semantic data models.

For example, a robot can publish its state through ROS 2. The FIWARE Context Broker can expose that state as an NGSI-LD ``Robot`` entity. A dashboard, AI service, MES, or digital twin can then query or subscribe to that entity without joining the ROS 2 graph directly.


Data Models
-----------

Data models define the shared meaning of the information exchanged through NGSI-LD.

They describe:

* entity types,
* attributes,
* relationships,
* value formats,
* expected semantics.

This is important because integration is not only about moving data from one protocol to another. The receiving system also needs to understand what the data means.

For example, a ROS 2 perception module may publish detected humans. Through an appropriate NGSI-LD data model, this information can be represented as ``HumanDetection`` entities with attributes such as position, confidence, timestamp, posture, and relationship to a workcell or robot.

The data model becomes the contract between the robotics module and the external applications consuming its context.


Other FIWARE Components
-----------------------

Although the Context Broker is the core FIWARE component in ARISE, it can be combined with other FIWARE building blocks depending on the needs of the solution.

These components can be used to:

* ease integration with other technologies,
* connect external devices and platforms,
* persist and analyze historical data,
* secure access to APIs,
* create dashboards and user-facing applications,
* support data sharing and data-space scenarios,
* build new reusable solution components Powered by FIWARE.

Typical examples include:

* **IoT Agents and protocol adapters**, used to connect devices, sensors, machines, or non-NGSI systems.
* **Data persistence and historical context components**, used to store context changes over time and support analytics.
* **Security components**, used to provide identity management, authentication, authorization, and policy enforcement.
* **Visualization and application-enablement components**, used to create dashboards or operational user interfaces.
* **Data-space and marketplace components**, used when context information must be shared across organizations.

This means FIWARE can be used in different ways. A developer may only expose a ROS 2 module through the Context Broker, or may combine several FIWARE components to create a more complete industrial platform or a new reusable component Powered by FIWARE.


What This Means for Solution Developers
---------------------------------------

For solution developers, the FIWARE layer provides a clear integration path.

A developer can:

#. build a robotics capability as a ROS 2 module;
#. decide which ROS 2 topics, services, or actions should be exposed outside the robotics layer;
#. define the corresponding NGSI-LD entities and data models;
#. configure the DDS-to-NGSI-LD mapping;
#. expose the module through the NGSI-LD API;
#. allow IT and engineering applications to query, subscribe to, or update context information.

The result is a robotics module that remains native to ROS 2, but is also accessible to enterprise applications, digital twins, AI services, dashboards, and industrial workflow systems.


Example
-------

A robot publishes its operational state in ROS 2:

.. code-block:: text

   Topic: /robot/state
   Data: mode, battery level, pose, active task, error code

The Context Broker exposes the same information as an NGSI-LD entity:

.. code-block:: json

   {
     "id": "urn:ngsi-ld:Robot:robot-01",
     "type": "Robot",
     "operationalState": {
       "type": "Property",
       "value": "executing"
     },
     "batteryLevel": {
       "type": "Property",
       "value": 78
     },
     "activeTask": {
       "type": "Relationship",
       "object": "urn:ngsi-ld:RobotTask:task-0001"
     }
   }

An external application can then query:

.. code-block:: text

   GET /ngsi-ld/v1/entities/urn:ngsi-ld:Robot:robot-01

Or subscribe to changes such as:

.. code-block:: text

   Notify me when operationalState changes.
   Notify me when batteryLevel is below a threshold.
   Notify me when activeTask changes.

This is the value of the FIWARE layer in ARISE: robotics capabilities become easier to integrate, monitor, orchestrate, and reuse across industrial systems.


Benefits
--------

The FIWARE component layer brings several benefits to ARISE deployments:

* **Interoperability** between robotics, industrial, enterprise, and engineering systems.
* **Loose coupling** between ROS 2 modules and external applications.
* **Semantic clarity** through NGSI-LD entities and data models.
* **Bidirectional integration** between ROS 2 / DDS and NGSI-LD when required.
* **Scalable adoption**, from simple monitoring to full platform integration.
* **Reuse of solution components** across different industrial deployments.
* **A foundation for new components Powered by FIWARE**.


Resources
---------

* `FIWARE Catalogue <https://www.fiware.org/catalogue/>`_
* `NGSI-LD <https://ngsi-ld.org/>`_
* `FIWARE Orion-LD Context Broker <https://github.com/FIWARE/context.Orion-LD>`_
* `ROS 2 on DDS <https://design.ros2.org/articles/ros_on_dds.html>`_
* `Vulcanexus Documentation <https://docs.vulcanexus.org/>`_
* `ROS 2 and FIWARE basic app <https://github.com/agileHRI/arise_documentation/blob/main/docs/rst/getting_started/ros2_fiware_basic_app.rst>`_
