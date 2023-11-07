# Kafka

## Data Transfer from Production Cluster to New Cluster
---------------------------------------------------------

###Replicating Production Data

  MirrorMaker2 will be used to replicate data from production to the new production region cluster and will be utilized as a active-passive setup in case of disaster recovery for production without any data loss. The Cluster Operator deploys one or more Kafka MirrorMaker replicas to replicate data between Kafka clusters. This process is called mirroring to avoid confusion with the Kafka partitions replication concept. MirrorMaker consumes messages from the source cluster and republishes those messages to the target cluster.
  
  MirrorMaker2 is an improvement over its predecessor MirrorMaker and has ability to synchronize topic configuration and tracks offsets ( This feature is very important so that the consumers which cutover in case of failure of the source cluster , can resume as if the source never went away).Also MirrorMaker2 provides an opportunity to replicate data bidirectionally , in that source can be a target and target can be a source , so we will start calling these clusters as remote instead of source and target. The general affinity of installing the mirrormaker2 is on the target cluster if only a single direction replication as in the case of the active-passive setup is considered.

###MirrorCheckpointConnector
MirrorCheckpointConnector tracks and maps offsets for specified consumer groups using an offset sync topic and checkpoint topic. The offset sync topic maps the source and target offsets for replicated topic partitions from record metadata. A checkpoint is emitted from each source cluster and replicated in the target cluster through the checkpoint topic. The checkpoint topic maps the last committed offset in the source and target cluster for replicated topic partitions in each consumer group. We can use RemoteClusterUtils.java class by adding connect-mirror-client as dependency to the consumers and achieve automatic failover.The class translates the consumer group offset from source cluster to the correspoding target offset for the target cluster.
The consumer groups tracked by MirrorCheckpointConnector are dependent on the listed patterns

Copy the user that has access on the topics , groups that need to be transferred to the destination cluster and create the user on the destination cluster . Also get the source cluster certificate by executing the below commands

        oc get secret <source-cluster>-ca-cert -o yaml > sourceclustercert.yaml
        oc get secret sourceuser -o yaml> sourceuser.yaml

Important Properties
• clusters: Define the Kafka clusters being synchronized
• mirrors: Define the MirrorMaker 2.0 connectors.
• spec : Spec includes the version of the kafka and as MirrorMaker2 is dependent on the Connector Framework the cluster alias is depicted by the connectCluster value.

    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaMirrorMaker2
    metadata:
      name: my-mm2-cluster
      namespace: kafka-dest
    spec:
      clusters:
      ############### SOURCE CLUSTER (Active) ################
        - alias: source
          authentication:
            certificateAndKey:
              certificate: user.crt
              key: user.key
              secretName: source-user                                       -----------------  1
            type: tls
          bootstrapServers: >-
            #source route#                                                  ------------------ 2
          tls:
            trustedCertificates:
              - certificate: ca.crt
                secretName: source-cluster-cluster-ca-cert                  ------------------- 3  
          config:
            config.storage.replication.factor: -1              
            offset.storage.replication.factor: -1
            status.storage.replication.factor: -1
        - alias: dest
          authentication:
            certificateAndKey:
              certificate: user.crt
              key: user.key
              secretName: dest-user                                        -------------------- 4
            type: tls
          bootstrapServers: #Dest service#                                  ------------------- 5
          config:
            config.storage.replication.factor: 3
            offset.storage.replication.factor: 3
            status.storage.replication.factor: 3
          tls:
            trustedCertificates:
              - certificate: ca.crt
                secretName: dest-cluster-cluster-ca-cert                   ---------------------- 6
      connectCluster: dest
      mirrors:
        - checkpointConnector:
            config:
              checkpoints.topic.replication.factor: 3
              sync.group.offsets.enabled: true
              refresh.groups.interval.seconds: 60
              replication.policy.class: org.apache.kafka.connect.mirror.IdentityReplicationPolicy ---------- 7
          heartbeatConnector:
            config:
              heartbeats.topic.replication.factor: 1
          sourceCluster: source
          targetCluster: dest
          sourceConnector:
            tasksMax: 12
            config:
              offset-syncs.topic.replication.factor: 3
              replication.factor: 3
              replication.policy.class: org.apache.kafka.connect.mirror.IdentityReplicationPolicy
              sync.topic.acls.enabled: 'false'
          topicsPattern: .*
          groupsPattern: .*
      replicas: 1
      version: 3.4.0
	  

1. User that has permission on the source cluster to topics and groups 
2. Production cluster Boot Strap route.
3. Source Cluster CA certificate.
4. Destination user that has permissions to create topics and routes with the tls authentication.
5. Service / Route of the destination server.
6. Destination cluster CA certificate.
7. IdentityReplicationFactor to keep the names of the topics same.

With the above setup complete run the below command to create mirrormaker2 in the target cluster

    oc create -f mirrormaker2.yaml
	

The above command will then create the mirrormaker2 connector and start replicating the source cluster on to the destination cluster.Mirror maker will create the below internal topics in the target cluster
• heartbeats : MirrorHeartbeatConnector periodically checks connectivity between clusters. A heartbeat is produced every second by the MirrorHeartbeatConnector into a heartbeat topic that is created on the local cluster.
• mirrormaker2-cluster-status : Kafka topic that stores connector and task status updates.
• mirrormaker2-cluster-configs : Kafka topic that stores connector and task status configurations
• mirrormaker2-cluster-offsets : Kafka topic that stores connector offsets.
And it will create the below internal topics on the source cluster mm2-offset-syncs.target-cluster.internal
With the above set up the production environment will be copied to the Q cluster.
