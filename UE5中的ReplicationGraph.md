UReplicationGraph::GlobalActorReplicationInfoMap，存储了两类信息：
1. ClassMap：UClass->FClassReplicationInfo的Map信息，表示该UClass对应的同步信息FClassReplicationInfo
2. ActorMap: 每个Actor实体以及相关的同步信息FGlobalActorReplicationInfo

UReplicationGraph::RPC_Multicast_OpenChannelForClass ？？？


UShooterReplicationGraph::ClassRepNodePolicies存储了UClass以及对应的RepNode信息，RepNode可以为下面的值：
1. NotRouted: 不会路由到任何Node下
2. RelevantAllConnections：该Node始终同步给所有Connections
3. Spatialize_Static: 需要路由到GridNode上，但这些Actors不会移动，所以不需要每帧都更新位置
4. Spatialize_Dynamic：需要路由到GridNode上，但这些Actors一直在移动，所以需要实时更新在Grid中的位置
5. Spatialize_Dormancy：需要路由到GridNode上，当休眠(Actor的休眠状态？)时视为static,移动时视为dynamic


ReplicationGraph.h

Subclasses should implement these functions:
UReplicationGraph::InitGlobalActorClassSettings
	Initialize UReplicationGraph::GlobalActorReplicationInfoMap.

UReplicationGraph::InitGlobalGraphNodes
	Instantiate new UGraphNodes via ::CreateNewNode. Use ::AddGlobalGraphNode if they are global (for all connections).

UReplicationGraph::RouteAddNetworkActorToNodes/::RouteRemoveNetworkActorToNodes
	Route actor spawning/despawning to the right node. (Or your nodes can gather the actors themselves)

UReplicationGraph::InitConnectionGraphNodes
	Initialize per-connection nodes (or associate shared nodes with them via ::AddConnectionGraphNode)


