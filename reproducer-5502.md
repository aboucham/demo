Env: AMQ Streams 2.5.1, OCP 4.13

1. Create namespaces myproject, myproject2 and myproject3
   ```
   oc new-project myproject
   oc new-project myproject2
   oc new-project myproject3
   ```
3. Created the following Kafka clusters
    ```yaml
    oc create -f - <<EOF
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    metadata:
      name: my-cluster
      namespace: myproject
    spec:
      kafka:
        version: 3.5.0
        replicas: 3
        listeners:
          - name: plain
            port: 9092
            type: internal
            tls: false
          - name: tls
            port: 9093
            type: internal
            tls: true
        config:
          offsets.topic.replication.factor: 3
          transaction.state.log.replication.factor: 3
          transaction.state.log.min.isr: 2
          default.replication.factor: 3
          min.insync.replicas: 2
          inter.broker.protocol.version: "3.5"
        storage:
          type: jbod
          volumes:
          - id: 0
            type: persistent-claim
            size: 100Gi
            deleteClaim: false
      zookeeper:
        replicas: 3
        storage:
          type: persistent-claim
          size: 100Gi
          deleteClaim: false
      entityOperator:
        topicOperator: {}
        userOperator: {}
    ---
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    metadata:
      name: my-cluster
      namespace: myproject2
    spec:
      kafka:
        version: 3.5.0
        replicas: 3
        listeners:
          - name: plain
            port: 9092
            type: internal
            tls: false
          - name: tls
            port: 9093
            type: internal
            tls: true
        config:
          offsets.topic.replication.factor: 3
          transaction.state.log.replication.factor: 3
          transaction.state.log.min.isr: 2
          default.replication.factor: 3
          min.insync.replicas: 2
          inter.broker.protocol.version: "3.5"
        storage:
          type: jbod
          volumes:
          - id: 0
            type: persistent-claim
            size: 100Gi
            deleteClaim: false
      zookeeper:
        replicas: 3
        storage:
          type: persistent-claim
          size: 100Gi
          deleteClaim: false
      entityOperator:
        topicOperator: {}
        userOperator: {}
    EOF
    ```
4. Deploy the MM2 cluster:
    ```yaml
    oc create -f - <<EOF
    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaMirrorMaker2
    metadata:
      name: my-mirror-maker-2
      namespace: myproject3
    spec:
      version: 3.5.0
      replicas: 1
      connectCluster: "cluster-b" # Must be the target custer
      clusters:
      - alias: "cluster-a" # Source cluster
        bootstrapServers: my-cluster-kafka-bootstrap.myproject.svc:9092
      - alias: "cluster-b" # Target cluster
        bootstrapServers: my-cluster-kafka-bootstrap.myproject2.svc:9092
        config:
          # -1 means it will use the default replication factor configured in the broker
          config.storage.replication.factor: -1
          offset.storage.replication.factor: -1
          status.storage.replication.factor: -1
      mirrors:
      - sourceCluster: "cluster-a"
        targetCluster: "cluster-b"
        sourceConnector:
          tasksMax: 1
          config:
            # -1 means it will use the default replication factor configured in the broker
            replication.factor: -1
            offset-syncs.topic.replication.factor: -1
            sync.topic.acls.enabled: "false"
            replication.policy.class: "org.apache.kafka.connect.mirror.IdentityReplicationPolicy"
            offset.lag.max: 500
        checkpointConnector:
          tasksMax: 1
          config:
            checkpoints.topic.replication.factor: -1
            sync.group.offsets.enabled: "true"
            replication.policy.class: "org.apache.kafka.connect.mirror.IdentityReplicationPolicy"
        topicsPattern: ".*"
        groupsPattern: ".*"
    EOF
    ```


5. Produce 1M records to the source cluster in namespace myproject:
   ```
   kubectl exec my-cluster-kafka-0 -n myproject -ti -- bin/kafka-producer-perf-test.sh --record-size=10 --num-records 1000000 --topic my-topic --throughput -1 --producer-props bootstrap.servers=my-cluster-kafka-bootstrap.myproject.svc:9092
   ```
6. Consume 500000 of them:
   ```
   kubectl exec my-cluster-kafka-0 -n myproject -ti -- bin/kafka-console-consumer.sh --max-messages 500000 --bootstrap-server my-cluster-kafka-bootstrap.myproject.svc:9092 --topic my-topic --group my-group --from-beginning
   ```
7. Verify committed offsets:
   ```
   kubectl exec my-cluster-kafka-0 -n myproject -ti -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --describe
   ```
   You should see something like this:
   ```
   GROUP                             TOPIC                 PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                                                                                         HOST            CLIENT-ID
   __strimzi-topic-operator-kstreams __strimzi_store_topic 0          5               5               0               __strimzi-topic-operator-kstreams-d08b5603-5040-45fd-b0d6-556f63e6cdcd-StreamThread-1-consumer-23d27182-c379-4238-b17d-173c9ee0a0d8 /10.129.2.22    __strimzi-topic-operator-kstreams-d08b5603-5040-45fd-b0d6-556f63e6cdcd-StreamThread-1-consumer

   Consumer group 'my-group' has no active members.

   GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
   my-group       my-topic        0          500000          1000000         500000          -               -               -
   ```
8. Wait some time ... and check the mirrored offset on the target cluster
   ```
   kubectl exec my-cluster-kafka-0 -n myproject2 -ti -- bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --all-groups --describe
   ```
   You should see something like this:
   ```
   GROUP                             TOPIC                 PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                                                                                                         HOST            CLIENT-ID
   __strimzi-topic-operator-kstreams __strimzi_store_topic 0          8               8               0               __strimzi-topic-operator-kstreams-e0c983dc-ad01-4ac0-a074-e082caff8bca-StreamThread-1-consumer-62413ed1-3787-4c2c-b112-4fc7ca1b48e9 /10.131.0.29    __strimzi-topic-operator-kstreams-e0c983dc-ad01-4ac0-a074-e082caff8bca-StreamThread-1-consumer

   Consumer group 'my-group' has no active members.

   GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
   my-group        my-topic        0          423393          1000000         576607          -               -               -
   ```
