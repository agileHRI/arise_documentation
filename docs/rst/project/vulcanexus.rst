.. _vulcanexus:

.. raw:: html

    </br></br>

.. image:: /rst/_static/images/vulcanexus_banner.png
    :align: center
    :alt: Vulcanexus

.. raw:: html

   <div style="visibility: hidden;">

Vulcanexus
==========

.. raw:: html

   </div>

Vulcanexus is an all-in-one toolset built around the ROS 2 (Robot Operating System 2) ecosystem.
It is developed and maintained by eProsima, the same team behind Fast DDS, and plays a key role in the ARISE framework by providing a solid, extensible, and production-grade base for robotics middleware. It is fully compatible with existing robotics applications developed using ROS 2, allowing seamless migration and integration without requiring changes to current ROS 2-based projects.

Vulcanexus is designed to streamline the development of robotic applications by offering high-performance communication, advanced tooling, and system-wide integration capabilities.

Unlike the standard ROS 2 distribution, Vulcanexus introduces a tailored suite of enhancements, packages, and configuration profiles that simplify ROS 2 usage in real-world deployments.

For more detailed information, guidance, and the latest updates, please refer to the official Vulcanexus documentation at https://docs.vulcanexus.org/en/latest/.

What Vulcanexus Offers
-----------------------

Vulcanexus provides significant improvements and additions over base ROS 2:

**1. Advanced DDS Support**

- Integrates Fast DDS v3+ (vs v2.14 in ROS 2 Jazzy) with better support for:
  - Zero-copy transfers
  - Customizable discovery servers
  - Data filtering and optimized serialization
- DDS Keys support for instance-based data tracking
- OMG DDS-XTypes v1.3 support for dynamic data modeling

**2. Better Discovery and Performance**

- Uses Discovery Server v3 to optimize discovery and reduce peer-to-peer overhead
- Minimizes network traffic and improves performance in lossy/cloud scenarios

**3. Developer Tooling**

- ``ROS 2 Monitor``: GUI tool for DDS/ROS 2 runtime introspection
- ``ROS 2 Spy``: CLI utility for debugging message flow and topic visibility
- ``ROS 2 Record & Replay``: Tools to log/playback DDS traffic
- ``ROS 2 Router``: DDS bridging for remote/cloud/multi-domain communication

**4. Modular Meta-Packages**

Vulcanexus is distributed as modular meta-packages:

- ``vulcanexus-core``: Core communication and system libraries
- ``vulcanexus-tools``: Introspection and debugging utilities
- ``vulcanexus-cloud``: DDS Router and tools for cloud-based communication
- ``vulcanexus-micro``: Embedded ROS 2 with Micro XRCE-DDS support
- ``vulcanexus-simulation``: Gazebo/Ignition plugins and models
- ``vulcanexus-desktop``: Full-featured package with visualization and simulation tools

**5. Deployment and Packaging**

- Docker Images: https://hub.docker.com/r/eprosima/vulcanexus
- Debian Packages: http://repo.vulcanexus.org/debian/
- Snap Packages: https://snapcraft.io/publisher/eprosima

Compatibility with ROS 2 Applications
-------------------------------------

Vulcanexus is fully compatible with existing robotics applications developed using ROS 2. This means that developers can run their current ROS 2-based projects on Vulcanexus without modifying source code, launch files, or message/service definitions. All standard ROS 2 APIs, tools (such as ros2 topic, ros2 service, and ros2 launch), and communication patterns are supported out of the box. As a result, teams can migrate or integrate their applications with Vulcanexus seamlessly, benefiting from its enhanced features and performance improvements while maintaining interoperability with the broader ROS 2 ecosystem.

Use in ARISE
------------

Vulcanexus is central to the ARISE architecture and acts as the default middleware stack across several core components:

- **NGSI-LD Context Broker**: Vulcanexus provides the underlying Fast DDS runtime used to bind ROS 2 topics to NGSI-LD entities. The ARISE Broker natively supports this integration, enabling seamless OT/IT data flows.
- **ROS4HRI Module Execution**: All open-source HRI components developed within ARISE are deployed over Vulcanexus, ensuring compatibility, observability, and extensibility.
- **TEFs and FSTP Pilots**: Testing and experimentation facilities (TEFs) and third-party projects (FSTPs) rely on Vulcanexus as a consistent middleware base, guaranteeing that applications can be deployed and tested under uniform communication and runtime conditions.

Interoperability with FIWARE Ecosystem
--------------------------------------

One of Vulcanexus's standout capabilities in ARISE is its interoperability with the FIWARE ecosystem. This is accomplished through a native integration with the `DDS Enabler <https://dds-enabler.readthedocs.io/>`__, a component that bridges ROS 2 (based on DDS) with FIWARE’s NGSI-LD-based context management systems.

The DDS Enabler
^^^^^^^^^^^^^^^

The DDS Enabler is an open-source library and runtime developed by eProsima and the FIWARE Foundation. It has been integrated into the Orion-LD Context Broker to:

- Listen to DDS topics (used in ROS 2 systems)
- Translate data into NGSI-LD entities
- Publish and subscribe to context updates via HTTP APIs or ROS 2 DDS samples

This makes it possible for robots running ROS 2 (via Vulcanexus) to:

- Expose telemetry and perception data as NGSI-LD entities to FIWARE systems
- React to context changes (e.g., commands from a manufacturing execution system)
- Maintain state in a shared, federated information model

In ARISE, this integration enables robust OT/IT convergence:

- OT devices (e.g., robots, sensors, actuators) publish data to ROS 2 topics
- Vulcanexus and the DDS Enabler push this data into the NGSI-LD context broker
- FIWARE systems (e.g., digital twins, analytics dashboards) subscribe to context changes
- Commands issued through NGSI-LD can be mapped back into ROS 2 topics

This closed-loop system architecture allows the creation of real-time digital twins, predictive maintenance scenarios, and cloud-assisted robotics workflows.

Resources
---------

- Vulcanexus Docs: https://docs.vulcanexus.org/en/latest/
- Fast DDS: https://github.com/eProsima/Fast-DDS
- DDS Enabler: https://github.com/eProsima/FIWARE-DDS-Enabler
- Orion-LD: https://github.com/FIWARE/context.Orion-LD
- Docker Images: https://hub.docker.com/r/eprosima/vulcanexus
- ARISE Portal: https://arise-middleware.eu


