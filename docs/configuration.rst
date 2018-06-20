.. _config_reference:

Configuring Docker Images
=========================

You can install the |cp| using Docker images. This section provides an overview of Confluent's Docker images for the |cp|.


.. contents::
    :depth: 2

Confluent Docker Images
-----------------------
The |cp| Docker images support passing configuration variables dynamically using environment variables.  When starting
the images use the Docker ``-e`` or ``--env`` flags for setting various settings.

Images are available on `DockerHub <https://hub.docker.com/u/confluentinc/>`_ for each component of the |cp|. The source
files for the images are `available on GitHub <https://github.com/confluentinc/cp-docker-images>`_. From GitHub you can
extend and rebuild the images and upload them to your own DockerHub repository.

This table lists the available images and the Confluent software packages that they contain.  Some images are identified
as ```cp-enterprise-${component_name}```.   These images include proprietary components that must be licensed from Confluent
when deployed.

+------------------+------------------------------+--------------+-----------------------------------------+
| Component        | Image Name                   | Type         | Packages Included                       |
+==================+==============================+==============+=========================================+
| Base Image       | cp-base                      | |cos|        | - zulu-openjdk-8                        |
+------------------+------------------------------+--------------+-----------------------------------------+
| Kafka            | cp-kafka                     | |cos|        | - confluent-kafka-*                     |
+------------------+------------------------------+--------------+-----------------------------------------+
| Kafka            | cp-enterprise-kafka          | |cpe|        | - confluent-kafka-*                     |
|                  |                              |              | - confluent-rebalancer                  |
|                  |                              |              | - confluent-support-metrics             |
+------------------+------------------------------+--------------+-----------------------------------------+
| Control Center   | cp-enterprise-control-center | |cpe|        | - confluent-control-center              |
+------------------+------------------------------+--------------+-----------------------------------------+
| Replicator       | cp-enterprise-replicator     | |cpe|        | - confluent-kafka-replicator            |
|                  |                              |              | - confluent-schema-registry             |
|                  |                              |              | - confluent-control-center              |
+------------------+------------------------------+--------------+-----------------------------------------+
| Kafka Connect    | cp-kafka-connect             | |cpe|        | - confluent-kafka-connect-jdbc          |
|                  |                              |              | - confluent-kafka-connect-hdfs          |
|                  |                              |              | - confluent-schema-registry             |
|                  |                              |              | - confluent-control-center              |
|                  |                              |              | - confluent-kafka-connect-elasticsearch |
|                  |                              |              | - confluent-kafka-connect-s3            |
+------------------+------------------------------+--------------+-----------------------------------------+
| Schema Registry  | cp-schema-registry           | |cos|        | - confluent-schema-registry             |
+------------------+------------------------------+--------------+-----------------------------------------+
| REST Proxy       | cp-kafka-rest                | |cos|        | - confluent-kafka-rest                  |
+------------------+------------------------------+--------------+-----------------------------------------+

Note: The Kafka Connect image is labeled as |cpe| because it contains the |c3| package. That package enables the deployed
connectors to collect the metrics visualized in |c3|.   No explicit license is required when using the Kafka Connect image
on its own.

Persistent Data (Mounted Volumes)
    When deploying the Kafka and |zk| images, you should always use `mounted volumes <operations/external-volumes.html>`_
    for the file systems those images use for their persistent data.  This ensures that the containers will retain their
    proper state when stopped and restarted.  The other images maintain their state directly in Kafka topics, so mounted
    volumes are not usually required for those containers.

Bridge Networking vs. Host Networking
    Bridge networking is currently only supported on a single host.  For multiple hosts, you must use overlay networks which
    are not currently supported. To expose Kafka to clients outside of the bridge network, you must find the container
    IP and put it in ``advertised.listeners``.  This can be difficult to achieve depending on how you're using the images.
    Furthermore, it can add a network hop and may not be as performant as the host network, which shares the network stack.
    Host networking is the recommended option in the following cases:

    * Multi-host clusters without using Swarm/Kubernetes host network is the best approach
    * If you need clients to be able to access Kafka outside the bridge/overlay network

Adding Connectors to the Kafka Connect Image
    Here are the methods to add new connectors to the Kafka Connect image.

    * Build a new Docker image that has the connector installed. You can follow the examples found in
      `Extending Images <development.html#extending-the-docker-images>`_. You must make sure that the connector
      JARs are on the CLASSPATH for the Connect service (the default location of ``/usr/share/java/kafka-connect-*`` is the
      recommended location).
    * Add the connector JARs via volumes.  If you don't want to create a new Docker image, please see our documentation
      on `Configuring Kafka Connect with External Jars <operations/external-volumes.html>`_ to configure the ``cp-kafka-connect``
      container with external JARs.

Supported Java
    The Confluent Docker images are tested and shipped with `Azul Zulu OpenJDK <https://www.azul.com/products/zulu/>`_.
    Other JDK's (including Oracle Java) are supported, but you must extend the images yourself to implement that change.

Untested Features
    The images are not currently tested on Docker Swarm.

Configuration Parameters
------------------------

The variables that are required when starting up the Docker images are described with an example of how to pass them.  

.. contents::
    :depth: 1
    :local:

-------------------
|zk| (cp-zookeeper)
-------------------

The |zk| image uses variables prefixed with ``ZOOKEEPER_`` with the variables expressed exactly as they would appear in the ``zookeeper.properties`` file.  As an example, to set ``clientPort``, ``tickTime``, and ``syncLimit`` run the command below:

.. sourcecode:: bash

    docker run -d \
    --net=host \
    --name=zookeeper \
    -e ZOOKEEPER_CLIENT_PORT=32181 \
    -e ZOOKEEPER_TICK_TIME=2000 \
    -e ZOOKEEPER_SYNC_LIMIT=2 \
    confluentinc/cp-zookeeper:4.0.0

Required |zk| Settings
""""""""""""""""""""""

These are the required |zk| settings. Some are only required, dependent on your configuration.

"""""""""""""""""""""""""
``ZOOKEEPER_CLIENT_PORT``
"""""""""""""""""""""""""
Instructs |zk| where to listen for connections by clients such as Kafka.

"""""""""""""""""""""""
``ZOOKEEPER_SERVER_ID``
"""""""""""""""""""""""
This is only required when running in clustered mode.  Sets the server ID in the ``myid`` file, which consists of a single
line that contains only the text of that machine's ID. For example, the ``myid`` of server 1 would only contain the text ``"1"``.
The ID must be unique within the ensemble and should have a value between 1 and 255.

--------------------------
Confluent Kafka (cp-kafka)
--------------------------

The Kafka image uses variables prefixed with ``KAFKA_`` with an underscore (``_``) separating each word. For example,
run the following commands to set ``broker.id``, ``advertised.listeners``, ``zookeeper.connect``, and
``offsets.topic.replication.factor``:

.. sourcecode:: bash

  docker run -d \
      --net=host \
      --name=kafka \
      -e KAFKA_ZOOKEEPER_CONNECT=localhost:32181 \
      -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092 \
      -e KAFKA_BROKER_ID=2 \
      -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
      confluentinc/cp-kafka:4.0.0

.. note:: The ``KAFKA_ADVERTISED_LISTENERS`` variable is set to ``localhost:29092``.  This makes Kafka accessible from
          outside the container by advertising its location on the Docker host.

.. include:: includes/config-shared.rst
    :start-line: 2
    :end-line: 4

Required Confluent Kafka Settings
"""""""""""""""""""""""""""""""""

"""""""""""""""""""""""""""
``KAFKA_ZOOKEEPER_CONNECT``
"""""""""""""""""""""""""""
Instructs Kafka how to get in touch with |zk|.

.. include:: includes/config-shared.rst
    :start-line: 7
    :end-line: 13

------------------------------------------------
Confluent Enterprise Kafka (cp-enterprise-kafka)
------------------------------------------------

The Enterprise Kafka image includes the packages for Confluent Auto Data Balancing and Proactive support in addition to
Kafka. The Enterprise Kafka image uses variables prefixed with ``KAFKA_`` for Apache Kafka and with ``CONFLUENT_`` for
Confluent components. These variables have an underscore (``_``) that seperates each word. For example,
run this command to set ``broker.id``, ``advertised.listeners``, ``zookeeper.connect``, ``offsets.topic.replication.factor``,
and ``confluent.support.customer.id``:

.. sourcecode:: bash

  docker run -d \
      --net=host \
      --name=kafka \
      -e KAFKA_ZOOKEEPER_CONNECT=localhost:32181 \
      -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092 \
      -e KAFKA_BROKER_ID=2 \
      -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
      -e CONFLUENT_SUPPORT_CUSTOMER_ID=c0 \
      confluentinc/cp-enterprise-kafka:4.0.0

.. note:: The ``KAFKA_ADVERTISED_LISTENERS`` variable is set to ``localhost:29092``.  It makes Kafka accessible from outside
          of the container by advertising its location on the Docker host.

If you want to enable Proactive support or use Confluent Auto Data Balancing features, see :ref:`cp-proactive-support` and
:ref:`rebalancer`.

.. include:: includes/config-shared.rst
    :start-line: 2
    :end-line: 4


Required Confluent Enterprise Kafka Settings
""""""""""""""""""""""""""""""""""""""""""""

"""""""""""""""""""""""""""
``KAFKA_ZOOKEEPER_CONNECT``
"""""""""""""""""""""""""""
Tells Kafka how to get in touch with |zk|.

.. include:: includes/config-shared.rst
    :start-line: 7
    :end-line: 13


------------------------------------
Schema Registry (cp-schema-registry)
------------------------------------

For the Schema Registry image, use variables prefixed with ``SCHEMA_REGISTRY_`` with an underscore (``_``) to separate each
word. For example, run the following to set ``kafkastore.connection.url``, ``host.name``, ``listeners``
and ``debug``:

.. sourcecode:: bash

    docker run -d \
      --net=host \
      --name=schema-registry \
      -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=localhost:32181 \
      -e SCHEMA_REGISTRY_HOST_NAME=localhost \
      -e SCHEMA_REGISTRY_LISTENERS=http://localhost:8081 \
      -e SCHEMA_REGISTRY_DEBUG=true \
      confluentinc/cp-schema-registry:4.0.0

Required Schema Registry Settings
"""""""""""""""""""""""""""""""""

"""""""""""""""""""""""""""""""""""""""""""""
``SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL``
"""""""""""""""""""""""""""""""""""""""""""""
|zk| URL for the Kafka cluster.

"""""""""""""""""""""""""""""
``SCHEMA_REGISTRY_HOST_NAME``
"""""""""""""""""""""""""""""
The hostname advertised in |zk|. This is required if if you are running Schema Registry with multiple nodes. Hostname is
required because it defaults to the Java canonical hostname for the container, which may not always be resolvable in a
Docker environment.  Hostname must be resolveable because slave nodes serve registration requests indirectly by simply
forwarding them to the current master, and returning the response supplied by the master.  For more information, see the
Schema Registry documentation on :ref:`Single Master Architecture <schemaregistry_single_master>`.

--------------------------------
Kafka REST Proxy (cp-kafka-rest)
--------------------------------
The variables used in the Kafka REST Proxy image are prefixed with ``KAFKA_REST_``, with an underscore (``_``) that separates
each word. For example, run this command to set the ``listeners``, ``schema.registry.url`` and
``zookeeper.connect``:

.. sourcecode:: bash

    docker run -d \
      --net=host \
      --name=kafka-rest \
      -e KAFKA_REST_ZOOKEEPER_CONNECT=localhost:32181 \
      -e KAFKA_REST_LISTENERS=http://localhost:8082 \
      -e KAFKA_REST_SCHEMA_REGISTRY_URL=http://localhost:8081 \
      confluentinc/cp-kafka-rest:4.0.0

Required Kafka REST Proxy Settings
""""""""""""""""""""""""""""""""""

The following settings must be passed to run the REST Proxy Docker image.

""""""""""""""""""""""""
``KAFKA_REST_HOST_NAME``
""""""""""""""""""""""""
The hostname used to generate absolute URLs in responses.  Hostname is required because it defaults to the Java canonical
hostname for the container, which may not always be resolvable in a Docker environment.  For more information, see the |cp|
documentation on :ref:`REST proxy deployment <kafka-rest-deployment>`.

""""""""""""""""""""""""""""""""
``KAFKA_REST_ZOOKEEPER_CONNECT``
""""""""""""""""""""""""""""""""
The |zk| connection string in the form ``hostname:port`` where host and port are the host and port of a |zk| server. To
allow connecting through other |zk| nodes when that |zk| machine is down you can also specify multiple hosts in the form
``hostname1:port1,hostname2:port2,hostname3:port3``.

The server may also have a |zk| ``chroot`` path as part of its |zk| connection string which puts its data under some path
in the global |zk| namespace. If so the consumer should use the same chroot path in its connection string. For example to
give a chroot path of ``/chroot/path`` you would give the connection string as ``hostname1:port1,hostname2:port2,hostname3:port3/chroot/path``.

--------------------------------
Kafka Connect (cp-kafka-connect)
--------------------------------

The variables used in the Kafka Connect image are prefixed with ``CONNECT_``, with an underscore (``_``) that separates
each word. For example, run this command to set the required properties like ``bootstrap.servers``,
the topic names for ``config``, ``offsets`` and ``status`` as well the ``key`` or ``value`` converter:

.. sourcecode:: bash

    docker run -d \
      --name=kafka-connect \
      --net=host \
      -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
      -e CONNECT_REST_PORT=28082 \
      -e CONNECT_GROUP_ID="quickstart" \
      -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-config" \
      -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-offsets" \
      -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-status" \
      -e CONNECT_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
      -e CONNECT_LOG4J_LOGGERS=org.reflections=ERROR \
      -e CONNECT_PLUGIN_PATH=/usr/share/java \
      confluentinc/cp-kafka-connect:4.0.0


Required Kafka Connect Settings
"""""""""""""""""""""""""""""""
The following settings must be passed to run the Kafka Connect Docker image.

"""""""""""""""""""""""""""""
``CONNECT_BOOTSTRAP_SERVERS``
"""""""""""""""""""""""""""""
A unique string that identifies the Connect cluster group this worker belongs to.

""""""""""""""""""""
``CONNECT_GROUP_ID``
""""""""""""""""""""
A unique string that identifies the Connect cluster group this worker belongs to.

""""""""""""""""""""""""""""""""
``CONNECT_CONFIG_STORAGE_TOPIC``
""""""""""""""""""""""""""""""""
The name of the topic in which to store connector and task configuration data. This must be the same for all workers with
the same ``group.id``

""""""""""""""""""""""""""""""""
``CONNECT_OFFSET_STORAGE_TOPIC``
""""""""""""""""""""""""""""""""
The name of the topic in which to store offset data for connectors. This must be the same for all workers with the same ``group.id``

""""""""""""""""""""""""""""""""
``CONNECT_STATUS_STORAGE_TOPIC``
""""""""""""""""""""""""""""""""
The name of the topic in which to store state for connectors. This must be the same for all workers with the same ``group.id``

"""""""""""""""""""""""""
``CONNECT_KEY_CONVERTER``
"""""""""""""""""""""""""
Converter class for keys. This controls the format of the data that will be written to Kafka for source connectors or read from Kafka for sink connectors.

"""""""""""""""""""""""""""
``CONNECT_VALUE_CONVERTER``
"""""""""""""""""""""""""""
Converter class for values. This controls the format of the data that will be written to Kafka for source connectors or read from Kafka for sink connectors.

""""""""""""""""""""""""""""""""""
``CONNECT_INTERNAL_KEY_CONVERTER``
""""""""""""""""""""""""""""""""""
Converter class for internal keys that implements the ``Converter`` interface.

""""""""""""""""""""""""""""""""""""
``CONNECT_INTERNAL_VALUE_CONVERTER``
""""""""""""""""""""""""""""""""""""
Converter class for internal values that implements the ``Converter`` interface.

"""""""""""""""""""""""""""""""""""""
``CONNECT_REST_ADVERTISED_HOST_NAME``
"""""""""""""""""""""""""""""""""""""
The hostname that is given out to other workers to connect to. In a Docker environment, your clients must be able to connect
to the Connect and other services.  Advertised hostname is how Connect gives out a hostname that can be reached by the client.

"""""""""""""""""""""""
``CONNECT_PLUGIN_PATH``
"""""""""""""""""""""""
The plugin.path value that indicates the location from which to load Connect plugins in classloading isolation.

Optional Kafka Connect Settings
"""""""""""""""""""""""""""""""
All other settings for Connect like security, monitoring interceptors, producer and consumer overrides can be passed to
the Docker images as environment variables. The names of these environment variables are derived by replacing ``.`` with ``_``,
converting the resulting string to uppercase and prefixing it with ``CONNECT_``. For example, if you need to set
``ssl.key.password``, the environment variable name would be ``CONNECT_SSL_KEY_PASSWORD``.

The image will then convert these environment variables to corresponding Connect config variables.

--------------------------------------------
Confluent Control Center (cp-control-center)
--------------------------------------------

The |c3| image use variables prefixed with ``CONTROL_CENTER_`` with an underscore (``_``) that separates each word, instead
of periods. For example, the following command runs |c3-short|, passing in its |zk|, Kafka, and Connect configuration parameters.

.. sourcecode:: bash

  docker run -d \
    --net=host \
    --name=control-center \
    --ulimit nofile=16384:16384 \
    -e CONTROL_CENTER_ZOOKEEPER_CONNECT=localhost:32181 \
    -e CONTROL_CENTER_BOOTSTRAP_SERVERS=localhost:29092 \
    -e CONTROL_CENTER_REPLICATION_FACTOR=1 \
    -e CONTROL_CENTER_CONNECT_CLUSTER=http://localhost:28082 \
    -v /mnt/control-center/data:/var/lib/confluent-control-center \
    confluentinc/cp-enterprise-control-center:4.0.0

|c3-short| Docker Options
"""""""""""""""""""""""""

* File descriptor limit:  |c3-short| may require many open files so we recommend setting the file descriptor limit to at least 16384

* Data persistence: the |c3-short| image stores its data in the /var/lib/confluent-control-center directory. We recommend that you bind this to a volume on the host machine so that data is persisted across runs.

|c3-short| Required Settings
""""""""""""""""""""""""""""
The following settings must be passed to run the |c3| image.

``CONTROL_CENTER_ZOOKEEPER_CONNECT``

Specifies the |zk| connection string in the form hostname:port where host and port are the host and port of a |zk| server.
To allow connecting through other |zk| nodes when that |zk| machine is down you can also specify multiple hosts in the form
``hostname1:port1,hostname2:port2,hostname3:port3``.

The server may also have a |zk| ``chroot`` path as part of its |zk| connection string which puts its data under some path
in the global |zk| namespace. If so the consumer should use the same chroot path in its connection string. For example to
give a chroot path of /chroot/path you would give the connection string as
``hostname1:port1,hostname2:port2,hostname3:port3/chroot/path``.

``CONTROL_CENTER_BOOTSTRAP_SERVERS``

A list of host/port pairs to use for establishing the initial connection to the Kafka cluster. The client will use
all servers irrespective of which servers are specified here for bootstrapping; this list only impacts the initial hosts
used to discover the full set of servers. This list should be in the form ``host1:port1,host2:port2,....`` Since these servers
are only used for the initial connection to discover the full cluster membership (which may change dynamically), this list
does not require the full set of servers. You might want more than one, in case a server is down.

``CONTROL_CENTER_REPLICATION_FACTOR``

Replication factor for |c3-short| topics.  We recommend setting this to 3 in a production environment.

|c3-short| Optional Settings
""""""""""""""""""""""""""""

``CONTROL_CENTER_CONNECT_CLUSTER``

  To enable |c3-short| to interact with a Kafka Connect cluster, set this parameter to the REST endpoint URL for the Kafka Connect cluster.

----------------------------------------------------------
Confluent Enterprise Replicator (cp-enterprise-replicator)
----------------------------------------------------------

Confluent Kafka Replicator is a Kafka connector and runs on a Kafka Connect cluster. The image uses variables prefixed
with ``CONNECT_`` with an underscore (``_``) separating each word. For example, run the following commands to set the
required properties like ``bootstrap.servers``, the topic names for ``config``, ``offsets`` and ``status`` and the
``key`` or ``value`` converter:

.. sourcecode:: bash

    docker run -d \
      --name=cp-enterprise-replicator \
      --net=host \
      -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
      -e CONNECT_REST_PORT=28082 \
      -e CONNECT_GROUP_ID="quickstart" \
      -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-config" \
      -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-offsets" \
      -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-status" \
      -e CONNECT_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
      -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
      confluentinc/cp-enterprise-replicator:4.0.0

The following example shows how to create a Confluent Kafka Replicator connector which replicates topic "confluent" from
source Kafka cluster (src) to a destination Kafka cluster (dest).

.. sourcecode:: bash

    curl -X POST \
         -H "Content-Type: application/json" \
         --data '{
            "name": "confluent-src-to-dest",
            "config": {
              "connector.class":"io.confluent.connect.replicator.ReplicatorSourceConnector",
              "key.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
              "value.converter": "io.confluent.connect.replicator.util.ByteArrayConverter",
              "src.zookeeper.connect": "zookeeper-src:2181",
              "src.kafka.bootstrap.servers": "kafka-src:9082",
              "dest.zookeeper.connect": "zookeeper-dest:2181",
              "topic.whitelist": "confluent",
              "topic.rename.format": "${topic}.replica"}}'  \
                http://localhost:28082/connectors

Required Confluent Enterprise Replicator Settings
"""""""""""""""""""""""""""""""""""""""""""""""""
The following settings must be passed to run the Kafka Connect Docker image:

"""""""""""""""""""""""""""""
``CONNECT_BOOTSTRAP_SERVERS``
"""""""""""""""""""""""""""""
A unique string that identifies the Connect cluster group this worker belongs to.

""""""""""""""""""""
``CONNECT_GROUP_ID``
""""""""""""""""""""
A unique string that identifies the Connect cluster group this worker belongs to.

""""""""""""""""""""""""""""""""
``CONNECT_CONFIG_STORAGE_TOPIC``
""""""""""""""""""""""""""""""""
The name of the topic where connector and task configuration data is stored. This must be the same for all workers with the same ``group.id``

""""""""""""""""""""""""""""""""
``CONNECT_OFFSET_STORAGE_TOPIC``
""""""""""""""""""""""""""""""""
The name of the topic where offset data for connectors is stored. This must be the same for all workers with the same ``group.id``

""""""""""""""""""""""""""""""""
``CONNECT_STATUS_STORAGE_TOPIC``
""""""""""""""""""""""""""""""""
The name of the topic where state for connectors is stored. This must be the same for all workers with the same ``group.id``

"""""""""""""""""""""""""
``CONNECT_KEY_CONVERTER``
"""""""""""""""""""""""""
Converter class for keys. This controls the format of the data that will be written to Kafka for source connectors or read from Kafka for sink connectors.

"""""""""""""""""""""""""""
``CONNECT_VALUE_CONVERTER``
"""""""""""""""""""""""""""
Converter class for values. This controls the format of the data that will be written to Kafka for source connectors or read from Kafka for sink connectors.

""""""""""""""""""""""""""""""""""
``CONNECT_INTERNAL_KEY_CONVERTER``
""""""""""""""""""""""""""""""""""
Converter class for internal keys that implements the ``Converter`` interface.

""""""""""""""""""""""""""""""""""""
``CONNECT_INTERNAL_VALUE_CONVERTER``
""""""""""""""""""""""""""""""""""""
Converter class for internal values that implements the ``Converter`` interface.

"""""""""""""""""""""""""""""""""""""
``CONNECT_REST_ADVERTISED_HOST_NAME``
"""""""""""""""""""""""""""""""""""""
The hostname that will be given out to other workers to connect to. In a Docker environment, your clients must be able to connect
to the Connect and other services.  Advertised hostname is how Connect gives out a hostname that can be reached by the client.

Optional Settings
"""""""""""""""""
All other settings for Connect like security, monitoring interceptors, producer and consumer overrides can be passed to
the Docker images as environment variables. The names of these environment variables are derived by replacing ``.`` with
``_``, converting the resulting string to uppercase and prefixing it with ``CONNECT_``. For example, if you must set
``ssl.key.password``, the environment variable name would be ``CONNECT_SSL_KEY_PASSWORD``.

The image will then convert these environment variables to corresponding Connect config variables.
