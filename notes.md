# Show code
# Show workings on commandline with 2 instances

sbt run

sbt -Dakka_port=2551 -Dhttp_port=8081 run


http localhost:8080/services/myservice

http localhost:8080/services/myservice host=different port:=9876

http localhost:8080/services/myservice
http localhost:8081/services/myservice

# replicator
Explain that ORSet needs node info for administration

place in SDA
```scala
val replicator = DistributedData(context.system).replicator
implicit val node = Cluster(context.system)
```

# Start with Get
Explain the need for the key.
```scala
val ServicesKey: ORSetKey[Service] = ORSetKey[Service]("services")
```

```scala
case Get(name) ⇒
  replicator ! Replicator.Get(ServicesKey, Replicator.ReadLocal, Some((name, sender())))
case result @ Replicator.GetSuccess(key, Some((name, sndr: ActorRef))) ⇒
  sndr ! Result(result.get(ServicesKey).elements.filter(_.name == name))
case Replicator.NotFound(_, Some((_, sndr: ActorRef))) ⇒
  // Will only happen when no services are registered
  sndr ! Result(Set.empty)
```

# Register

explain that you interact with the data via the replicator and you can't directly modify the data. Thus you send
a function to the replicator which modifies the data. The replicator takes the function and applies it to local
value and then replicates it according to the Write consistency. For service discovery we want eventual consistency,
so we write to local only.

Because of the asynchronous communication a message will be sent back to indicate whether the update was successful

```scala
    case Register(s, h, p) ⇒
      replicator ! Replicator.Update(ServicesKey, ORSet.empty[Service], Replicator.WriteLocal, Some(sender())) { set ⇒
        set + Service(s, h, p)
      }
    case Replicator.UpdateSuccess(key, Some(sndr: ActorRef)) ⇒
      sndr ! Updated
    case Replicator.ModifyFailure(ServicesKey, errorMessage, throwable, requestContext) ⇒
    // happens when the update function throws an exception
    case Replicator.UpdateTimeout(ServicesKey, requestContext) ⇒
    // happens when write consistence > local and nodes don't respond

```

# update application.conf

provider = "akka.cluster.ClusterActorRefProvider"


