---
title: enode执行过程
date: 2019-06-13 14:58:08
tags: 
- enode
- DDD
categories:
- DDD
---

## 前言
- https://www.cnblogs.com/netfocus/p/3859371.html

ENode是一个基于消息的架构，使用ENode开发的系统，每个环节都是处理消息，处理完后产生新的消息。本篇文章我想详细分析一下ENode框架内部是如何实现整个消息处理流程的。为了更好的理解我后面的流程的描述，我觉得还是应该先把ENode的架构图贴出来，好让大家在看后面的分析时，可以对照这个架构图进行思考和理解。

### ENode架构图

![](/images/enode/enode-arch.jpg)

## ENode框架内部实现流程分析

1. Controller发送ICommand到分布式消息队列EQueue（可以是各类分布式MQ）；
2. 【从这一步开始处理Command】MQ中的CommandConsumer接收到该ICommand，先创建一个ICommandContext实例，然后调用ENode中的ICommandExecutor执行当前ICommand并将ICommandContext传递给ICommandExecutor；
3. ICommandExecutor根据当前ICommand的类型，获取到一个唯一的ICommandHandler，然后调用ICommandHandler的Handle方法处理当前ICommand，调用时传递当前的ICommandContext给ICommandHandler；
4. ICommandHandler处理完Command后，ICommandExecutor获取当前ICommandContext中新增或修改的聚合根；
5. 检查当前ICommandContext中是否只有一个新增或修改的聚合根；如果超过1个，则报错，通过这样的检查来从框架级别保证一个Command一次只能修改一个聚合根；
6. 如果发现当前新增或修改的聚合根为0个，则直接认为当前的ICommand已处理完成，就调用ICommandContext的OnCommandExecuted方法，该方法内部会通知EQueue发送CommandResult消息给Controller；然后Controller那边的进程，会有一个CommandResultProcessor接收到这个CommandResult的消息，然后就知道该ICommand的处理结果了；（update: 消息通知的方式已经修改为使用 Socket 发送结果）
7. ICommandExecutor从ICommandContext拿到当前唯一修改的聚合根后，取出该聚合根里产生的IDomainEvent。由于一个聚合根一次可能会长生多个IDomainEvent，所以我们会构建一个EventStream对象。这个对象包含了所有当前聚合根所产生的IDomainEvent。一个EventStream会包含很多重要的信息，包括当前ICommand的ID、聚合根的ID、聚合根的Version（版本号），以及所有的IDomainEvent，等等；【update:命令在确认事件持久化完成后则执行完成】
8. ICommandExecutor将该Command添加到ICommandStore。因为ICommandStore是以CommandId为主键（即Key），所以如果CommandId重复，框架就会知道，然后就会做重复时的逻辑处理，这点后面再详细分析；【update:基于性能考虑，废弃了这种处理方式，Command的幂等判断在持久化事件的时候来做了】
9. 如果Command成功添加到ICommandStore，则接下来调用IEventService的Commit方法将当前EventStream持久化到IEventStore；

10. IEventService内部主要做3件事情： 
    - 1）将EventStream持久化到IEventStore；
    - 2）持久化成功后调用IMemoryCache更新缓存（缓存可以配置为本地缓存也可以配置为分布式缓存Redis，如果Command的处理是集群处理的，那我们应该用共享缓存，也就是用Redis这种分布式缓存）；
    - 3）缓存更新好之后，调用IEventPublisher接口的Publish方法将EventStream发布出去，IEventPublisher的具体实现者会把当前的EventStream发送到EQueue。

    > 这3步是正常情况的流程。如果遇到持久化到IEventStore时遇到版本号重复（同一个聚合根ID+聚合根的Version相同，则认为有并发冲突），此时框架需要做不同的逻辑处理；这点也在后面详细分析。

11. 【从这一步开始处理Domain Event】EventStream被ENode.EQueue中的EventConsumer接收到，然后EventConsumer调用IEventProcessor处理当前的EventStream；

12.  IEventProcessor首先判断当前的EventStream是否可以被处理，这里我们需要保证的很关键的一点是，必须确保事件的持久化顺序和被事件的订阅者处理的顺序要严格一样，否则就会出现Command端的数据和Query端的Read DB中的数据不一致的情况。关于如何保证这个顺序的一致，后面我们在详细分析。这里先举个简单的例子来说明为什么要顺序一致。比如假如现在有一个聚合根的一个属性，该属性的默认值是0，然后该属性先后发生了三个Domain Event（代表的意思分别是对这个属性做+1,*2,-1）。这三个事件如果按照这样的顺序发生后，那这个属性最后的值是1；但是如果这3个事件被消费者消费的顺序是+1,-1,*2那最后的结果就不是1了，而是0了；所以通过这个例子，我想大家应该都知道了为什么要严格保证聚合根持久化事件的顺序必须和被消费的顺序要完全一致了；

13. 假如当前的EventStream允许被处理，则IEventProcessor对当前的EventStream中的每个IDomainEvent做如下处理：
    - 根据IDomainEvent的类型得到所有当前IEventProcessor节点上所有注册的IEventHandler，然后调用它们的Handle方法，完成比如Query端的Read DB的更新。但是事情还没那么简单，因为我们还需要保证当前的IDomainEvent只会被当前的IEventHandler处理一次，否则IEventHandler就会因为重复处理了IDomainEvent而导致最后的数据不对；这里的幂等也在后面详细讨论。

14. 有些IEventHandler处理完IDomainEvent后会产生新的ICommand（就是Saga Process Manager）的情况。这种情况下，我们还需要把这些产生的ICommand由框架自动发送到消息队列（EQueue）；但是事情也没那么简单，要是发送这些ICommand失败了呢？那就需要重发，那重发如何设计才能保证不管重发多少次，也不会导致ICommand的重复执行呢？这里其实最关键的一点是要保证你每次重发的ICommand的Id总是和第一次发送时要相同的，否则框架就无法知道是否是同一个Command了。这里的具体设计后面再分析。
 
## Command的幂等处理

首先Command的幂等判断在持久化事件的时候来做了，怎么做的呢？

将事件流提交到EventStore时，EventStream存储表中定义了聚合根ID和CommadnId的唯一索引，根据insert语句抛出的异常类型来判断是不是DuplicateCommand。

这里涉及到两种持久化方式的处理（批量处理，顺序处理）
1. 未开启批量处理，顺序写入db，通过SQLException来判断唯一索引的冲突情况（Command重复 or Event重复）。
2. 当开启批量处理时，写入失败时，重试每个写SQL，然后执行1。

上面流程中的第8步，Command会被添加到ICommandStore。这里，实际上我添加到ICommandStore的是一个HandleCommand对象，该对象包含当前的Command之外，还有当前被修改的聚合根ID。这样做的理由请看我后面的解释。我们知道ICommandStore会对CommandId作为主键，这样我们就能绝对保证一个Command不会被重复添加。如果Command添加到ICommandStore成功，那自然最好了，直接进入后续的步骤即可；但是如果出现CommandId重复的时候，我们需要做怎么样的处理呢？

如果出现重复，则需要根据CommandId（主键），把之前已经持久化过的HandledCommand取出来；然后然后我们从HandledCommand拿到被修改的聚合根ID，然后最关键的一步：我们将该聚合根ID以及CommandId作为条件从IEventStore中查询出一个可能存在的EventStream。如果存在，就说明这个Command所产生的Domain Event已经被持久化了，所以我们只要再做一遍发布事件的操作即可。即调用IEventPublisher.Publish方法来发布事件到Query Side。那么为什么要发布呢？因为虽然事件被持久化了，但并不代表已经成功被发布出去了。因为理论上有可能Domain Event被持久化成功了，但是在要发布事件的时候，断电了！所以这种情况下，重启服务器就会出现这里讨论的情况了。所以我们需要再次Publish事件。

然后，如果没有根据CommandId和聚合根ID查找到EventStream呢？那也好办，因为这种情况就说明这个Command虽然被持久化了，但是它所产生的EventStream却没有被持久化到EventStore，所以我们需要将当前的EventStream调用IEventService.Commit方法进行持久化事件。

另外，这里其实有一个疑问，为什么查找EventStream不能仅仅根据CommandId呢？原因是：从技术上来说，我们可以只根据CommandId来查找到一个唯一的EventStream，但这样设计的话，就要求EventStore必须要支持通过一个CommandId来全局唯一定位到一个EventStream了。但是因为考虑到EventStore的数据量是非常大的，我们以后可能会根据聚合根ID做水平拆分（sharding）。这样的话，我们仅仅靠CommandId就无法知道到哪个分片下去查找对应的EventStream了。所以，如果查询时，能同时指定聚合根ID，那我们就能轻松知道首先到哪个分片下去找EventStream，然后再根据CommandId就能轻松定位到一个唯一的EventStream了。

既然说到这里，我再说一下CommandStore的水平分割的设计吧，CommandStore的数据量也是非常大的，因为它会存储所有的Command。不过幸好，我们对于CommandStore只需要根据CommandId去查找即可，所以我们可以根据CommandId来做Hash取模的方式来水平拆分。这样即便是分片了，我们只要知道了一个给定的CommandId，也能知道它当前是在哪个分片下的，就很容易找到该Command了。

所以，通过上面的分析，我们知道了CommandStore和EventStore在设计上不仅仅考虑了如何存储数据，还考虑了未来大数据量时如何分片，以及如何在分片的情况下仍然能方便的查找到我们的数据。

最后，上面还有一种情况没有说明，就是当出现Command添加到CommandStore时发现重复，但是尝试从CommandStore中根据CommandId查询该Command时，发现查不到，天哪！这种情况实际上不应该出现，如果出现，那说明CommandStore内部有问题了。因为为什么添加时说有重复，而查询却差不多来呢？呵呵。这种情况就无法处理了，我们只能记录错误日志，然后进行后续的排查。

## Domain Event持久化时的并发冲突检测和处理

上面流程中的第10步，我们提到：如果遇到EventStream持久化到IEventStore时遇到版本号重复（同一个聚合根ID+聚合根的Version相同，则认为有并发冲突），此时框架需要做不同的逻辑处理。具体是：

首先，我们可以先想想为什么会出现同一个聚合根会在几乎同一时刻产生两个版本号一样的领域事件，并持久化到EventStore。首先，我先说一下这种情况几乎不会出现的理由：ENode中，在ICommandExecutor在处理一个Command时，会检查当前该Command所要修改的聚合根是否已经有至少一个聚合根正在被处理，如果有，则会将当前Command排入到这个聚合根所对应的等候队列。也就是说，它暂时不会被执行。然后当当前聚合根的前面的Command被执行完了后才会从这个等候队列取出下一个等待的Command进行处理。通过这样的设计，我们保证了，对一个聚合根的所有Command，不会并行被执行，只会按照顺序被执行。因为每个ICommandExecutor会在需要的时候，为某个聚合根自动创建这种等候队列，只要对该聚合根的Command同一时刻进来2个或以上。

那么，要是集群的时候呢？你一台机器的话，通过上面的方式可以确保一个聚合根实例的所有的Command会被顺序处理。但是集群的时候，可能一个聚合根会在多台机器被同时处理了。要解决这个问题的思路就是对Command按照聚合根ID进行路由了，因为一般只要是修改聚合根的Command总是会带有一个聚合根ID，所以我们可以按照这个特性，对被发送的Command按照聚合根ID进行路由。只要CommandId相同，则总是会被路由到同一个队列，然后因为一个队列总是只会被一台机器消费，从而我们能保证对同一个聚合根的Command总是会落到一台机器上被处理。那么你可能会说，要是热点数据呢？比如有些聚合根突然对他修改的Command可能非常多（增加了一倍），而有些则很少，那怎么办呢？没关系，我们还有消息队列的监控平台。当出现某个聚合根的Command突然非常多的时候，我们可以借助于EQueue的Topic的Queue可以随时进行增加的特性来应付这个问题。比如原来这个Topic下只有4个Queue，那现在增加到8个，然后消费者机器也从4台增加到8台。这样相当于Command的处理能力也增加了一倍。从而可以方便的解决热点数据问题。因此，这也是我想要自己实现分布式消息队列EQueue的原因啊！有些场景，要是自己没有办法完全掌控，会很被动，直接导致整个架构的严重缺陷，最后导致系统瘫痪，而自己却无能为了。当然你可以说我们可以使用Kafka, Rocketmq这样的高性能分布式队列，确实。但是毕竟这种高大上的队列非常复杂，且都是非.NET平台。除了问题，维护起来肯定比自己开发的要难维护。当然除非你对它们非常精通且有自信的运维能力。

通过上面的思路实现的，确保聚合根的Command总是被顺序线性处理的设计，对EventStore有非常大的意义。因为这样可以让EventStore不会出现并发冲突，从而不会造成无谓的对EventStore的访问，也可以极大的降低EventStore的压力。

但是什么时候还是可能会出现并发冲突呢？因为：

1）当处理Command的某台机器挂了，然后这台机器所消费的Queue里的消息就会被其他机器接着消费。其他机器可能会从这个Queue里批量拉取一些Command消息来消费。然后此时假如我们重启了这台有问题的服务器，重启完之后，因为又会开始消费这个Queue。然后一个关键的点是，每次一台机器启动时，会从EQueue的Broker拉取这个Queue最后一个被消费的消息的位置，也就是Offset，而由于这个Offset的更新是异步的，比如5s才会更新到EQueue的Broker，所以导致这台重启后的服务器从Broker上拉取到的消费位置其实是有延迟的，从而就可能会去消费在那台之前接替你的服务器已经消费过的或者正在消费的Command消息了。当然这种情况因为条件太苛刻，所以基本不会发生，即便会发生，一般也不会导致Command的并发执行。但是这毕竟也是一种可能性。实际上这里不仅仅是某个服务器挂掉后再重启的情况会导致并发冲突，只要是处理Comand的机器的集群中有任何的机器的增加或减少，由于都会导致Command消息的消费者集群重新负载均衡。在这个负载均衡的过程中，就会导致同一个Topic下的同一个Queue里的部分消息可能会在两台服务器上被消费。原因是Queue的消费位置（offset）的更新不是实时的，而是定时的。所以，我们一般建议，尽量不要在消息很多的时候做消费者集群内机器的变动，而是尽量在没什么消息的时候，比如凌晨4点时，做集群的扩容操作。这样可以尽量避免所有可能带来的消息重复消费或者并发冲突的可能性。呵呵，这段话也许很多人看的云里雾里，我只能说到这个程度了，也许要完全理解，大家还需要对EQueue的设计很清楚才行！

2）就算同一个机器内，其实也是有可能出现对同一个聚合根的并发修改，也就是针对同一个聚合根的两个Command被同时执行。原因是：当一个Command所对应的EventStream在被持久化时出现重复，然后我就会放在一个本地的内存队列进行重试，然后重试由于是在另一个专门的重试线程里，该线程不是正常处理Command的线程。所以假如对该聚合根后续还有Command要被处理，那就有可能会出现同一时刻，一个聚合根被两个Command修改的情况了。

现在，我们在回来讨论，假如遇到冲突时，要怎么做？这个上面我简单提到过，就是需要重试Command。但也不是这么简单的逻辑。我们需要：

a. 先检查当前的EventStream的Version是否为1，假如为1，说明有一个创建聚合根的Command被并发执行了。此时我们无须在重试了，因为即便再重试，那最后产生的EventStream的版本号也总是1，因为只要是第一次创建聚合根，那这个聚合根所产生的DomainEvent的版本总是1。所以这种情况下，我们只需要直接从EventStore拿出这个已经存在的EventStream，然后通过IEventPublisher.Publish方法发布该EventStream即可。为什么要再次发布，上面解释Command的幂等时，也解释了原因，这里是一样的原因。这里也有一个小的点需要注意，就是假如尝试从EventStore拿出这个EventStream时，假如没获取到呢？这个问题实际上不应该出现，原因就像上面分析Command幂等时一样，为什么会出现添加时提示存在，但查询时却查不到的情况呢？这种情况就是EventStore的设计有问题了，读写存在非强一致性的情况了。

b. 如果当前的EventStream的Version大于1，则我们需要先更新内存缓存（Redis），然后做Command的重试处理。为什么要先更新缓存呢？因为如果不更新，有可能重试时，拿到的聚合根的状态还是旧的，所以重试后还是导致版本号冲突。那为什么从缓存中拿到的聚合根的状态可能还是旧的呢？因为EventStream已经存在于EventStore并不代表这个EventStream的修改已经更新到缓存了。因为我们是先持久化到EventStore，在更新缓存的。完全有可能你还没来得及更新缓存的时候，另一个Command正好需要重试呢！所以，最保险的做法，就是再重试的时候将缓存中的聚合根状态更新到最新值。那怎么更新呢？呵呵，很简单，就是通过事件溯源（即Event Sourcing技术）了。我们只要从Event Store获取当前聚合根的所有的Event Stream，然后溯源这些事件，最后就能得到聚合根的最新版本的状态了，然后更新到缓存即可。

最后，如果需要重试的话，要怎么重试呢？很简单，只要扔到一个本地的基于内存的重试队列即可。我现在是用BlockingCollection的。

## 如何保证事件产生的顺序和被消费的顺序相同

为什么要保证这个相同的顺序，在上面的流程步骤介绍里已经说明了。这里我们分析一下如何实现这个顺序的一致。基本的思路是用一个表，存放所有聚合根当前已经处理过的最大版本号，假如当前已经处理过的最大版本号是10，那接下来只能处理这个聚合根版本号为11的EventStream。即便Version=12或者更后面的先过来，也只能等着。那怎么等呢？也是类似Command的重试队列一样，在一个本地的内存队列等就行了。比如现在最大已处理的版本号是10，然后现在12,13这两个版本号的EventStream先过来，那就先到队列等着，然后版本号是11的这个事件过来了，就可以处理。处理好之后，当前最大已处理的版本号就编程11了，所以等候队列中的版本号为12的EventStream就可以允许被处理了。整个控制逻辑就是这样。那么这是单机的算法，要是集群呢？实际上这不必考虑集群的情况，因为我们每台机器上都是这个顺序控制逻辑，所以如果是集群，那最多可能出现的情况（实际上这种情况存在的可能性也是非常的低）是，版本号为11的EventStream被并发的处理。这种情况就是我下面要分析的。

这里实际上还有一个细节我还没说到，这个细节和EQueue的Consumer的ConsumerGroup相关，就是假如一种消息，有很多Consumer消费，然后这些Consumer假如分为两个ConsumerGroup，那这两个ConsumerGroup的消费是相互隔离的。也就是说，所有这些消息，两个ConsumerGroup内的Consumer都会消费到。这里如果不做一些其他的设计，可能会在用户使用时遇到潜在的问题。这里我没办法说的很清楚，说的太清楚估计会让大家思维更混乱，且因为这个点不是重点。所以就不展开了。有兴趣的朋友可以看一下ENode中的EventPublishInfo表中的EventProcessorName字段的用意。

## 如何保证一个IDomainEvent只会被一个IEventHandler处理一次
这个只是提供一个思路，框架未实现。
这一条的原因，我想大家都能理解。比如一个Event Handler是更新读库的，可能我们会执行一条有副作用的SQL，比如update product set price = price + 1 where id = 1000。这条SQL如果被重复执行一次，那price字段的值就多了1了，这不是我们所期望的结果。所以框架需要有基本的责任可以基本避免这种情况的发生。那怎么实现呢？思路也是用一张表，记录被执行的DomainEvent的ID以及当前处理这个DomainEvent的Event Handler的类型的一个Code，对这两个联合字段做联合唯一索引。每次当一个Event Handler要处理一个Domain Event时，先判断是否已经处理过，如果没处理过，则处理，处理过之后把被处理的Domain Event ID和EventHandler Type Code添加到这个表里即可。那假如添加的时候失败了呢？因为有可能也会有并发的情况，导致Event Handler重复处理同一个Domain Event。这种情况框架就不做严谨的处理了，因为框架本身也无法做到。因为框架式无法知道Event Handler里面到底在做什么的。有可能是发送邮件，也有可能是记录日志，也可能是更新读取（Read DB）。所以，最根本的，还是要求Event Handler内部，也就是开发这自己需要考虑幂等的实现。当然框架会提供给开发者必要的信息来帮助他们完成严谨幂等控制。比如框架会提供当前Domain Event 的版本号给Event Handler，这样Event Handler里就能在Update SQL的Where部分把这个Version带上，从而实现乐观并发控制。比如下面的代码示例：

```java
public void Handle(IEventContext context, SectionNameChangedEvent evnt)
{
    TryUpdateRecord(connection =>
    {
        return connection.Update(
            new
            {
                Name = evnt.Name,
                UpdatedOn = evnt.Timestamp,
                Version = evnt.Version
            },
            new
            {
                Id = evnt.AggregateRootId,
                Version = evnt.Version - 1
            }, Constants.SectionTable);
    });
}
```
上面的代码中，当我们更新一个论坛的版块时，我们可以在sql的where条件中，用version = evnt.Verion - 1这样的条件。从而确保当前你要处理的事件一定是上一次已处理的事件的版本号的下一个版本号，也就是保证了Query Side的更新的顺序严格和事件产生的顺序一致。这样即便框架在有漏网之鱼的时候，Event Handler内部也能做严谨的顺序控制。当然如果你的Event Handler是发送邮件，那我还真不知道该如何进一步保证这个严谨的顺序或者并发冲突了，呵呵。有兴趣的朋友可以和我交流。

## 在Saga Process Manager中产生的ICommand如何能够支持重试发送而不导致操作的重复

终于到最后一点了，好累。坚持就是胜利！假如现在的Saga Event Handler里是会产生Command，那框架在发送这些Command时，要确保不能重复执行。怎么办呢？假如在Saga Event Handler里产生的Command的Id每次都是新new出来的一个唯一的值，那框架就无法知道这个Command是否和之前的重复了，框架会认为这是两个不同的Command。这里其实有两种做法：

1. 框架先把Saga Event Handler中产生的Command保存起来，然后慢慢发送到EQueue。发送成功一个，就删除一个。直到全部发送完为止。这种做法是可行的，因为这样一来，我们要发送的Command就总是从存储这些Command的地方去拿了，所以不会出现每次要发送的同一个Command的ID都是不同的情况。但是这种设计性能不是太好，因为要发送的Command必须要先被保存起来，然后再发送，发送完之后还要删除。性能上肯定不会太高。

2. 第二种做法是，Command不存储起来，而是直接把Saga Event Handler中产生的Command拿去发送。但这种设计要求：框架对这种要发送的Command的ID总是按照某个特定的规则来产生的。这种规则要保证产生的CommandId首先是要唯一的，其次是确定的。下面我们看一下下面的代码：

```java
private string BuildCommandId(ICommand command, IDomainEvent evnt, int eventHandlerTypeCode)
{
    var key = command.GetKey();
    var commandKey = key == null ? string.Empty : key.ToString();
    var commandTypeCode = _commandTypeCodeProvider.GetTypeCode(command.GetType());
    return string.Format("{0}{1}{2}{3}", evnt.Id, commandKey, eventHandlerTypeCode, commandTypeCode);
}
```

上面这个代码是一个函数，用来构建要被发送的Command的Id的，我们可以看到ID是由Command的一个key+要被发送的Command的类型的code+当前被处理的Domain Event的ID，以及当前的Saga Event Handler的类型的code这四个信息组成。对于同一个Domain Event被同一个Event Handler处理，然后如果产生的Command的类型也是一样的，那我们基本可以通过这三个信息构建一个唯一的CommandId了，但是有时这样还不够，因为我们可能在一个Event Handler里构建两个类型完全一样的Command，但是他们修改的聚合根的ID不同。所以，我上面才有一个commandKey的组成部分。这个key默认就是这个Command要修改的聚合根的ID。这样，通过这样4个信息的组合，我们可以确保不管某个Domain Event被某个Saga Event Handler处理多少次，最后产生的Command的ID总是确定的，不变的。当然上面的commandKey有时仅仅考虑聚合根ID可能还不够，虽然我还没遇到过这种情况，呵呵。所以我框架设计上，就允许开发者能重新GetKey方法，开发者需要理解何时需要重写这个方法。看了这里的说明应该就知道了！
