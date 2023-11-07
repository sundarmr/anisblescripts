# Kafka

## Data Transfer from Production Cluster to New Cluster
---------------------------------------------------------

### Replicating Production Data

  MirrorMaker2 will be used to replicate data from production to the new production region cluster and will be utilized as a active-passive setup in case of disaster recovery for production without any data loss. The Cluster Operator deploys one or more Kafka MirrorMaker replicas to replicate data between Kafka clusters. This process is called mirroring to avoid confusion with the Kafka partitions replication concept. MirrorMaker consumes messages from the source cluster and republishes those messages to the target cluster.
  
  MirrorMaker2 is an improvement over its predecessor MirrorMaker and has ability to synchronize topic configuration and tracks offsets ( This feature is very important so that the consumers which cutover in case of failure of the source cluster , can resume as if the source never went away).Also MirrorMaker2 provides an opportunity to replicate data bidirectionally , in that source can be a target and target can be a source , so we will start calling these clusters as remote instead of source and target. The general affinity of installing the mirrormaker2 is on the target cluster if only a single direction replication as in the case of the active-passive setup is considered.

### MirrorCheckpointConnector
MirrorCheckpointConnector tracks and maps offsets for specified consumer groups using an offset sync topic and checkpoint topic. The offset sync topic maps the source and target offsets for replicated topic partitions from record metadata. A checkpoint is emitted from each source cluster and replicated in the target cluster through the checkpoint topic. The checkpoint topic maps the last committed offset in the source and target cluster for replicated topic partitions in each consumer group. We can use RemoteClusterUtils.java class by adding connect-mirror-client as dependency to the consumers and achieve automatic failover.The class translates the consumer group offset from source cluster to the correspoding target offset for the target cluster.
The consumer groups tracked by MirrorCheckpointConnector are dependent on the listed patterns

Copy the user that has access on the topics , groups that need to be transferred to the destination cluster and create the user on the destination cluster . Also get the source cluster certificate by executing the below commands

        oc get secret <source-cluster>-ca-cert -o yaml > sourceclustercert.yaml
        oc get secret sourceuser -o yaml> sourceuser.yaml
		
