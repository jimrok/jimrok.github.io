---
layout: post
title:  "Actor和Let it crash"
date:   2018-12-16 22:34:25
categories: Akka actor java
---

谈论Akka的Actor，不得不谈论Erlang的Let it crash，在Erlang的错误处理中，推崇Let it crash这一处理哲学。对于没有接触过Erlang语言的人，很难理解这一哲学。程序崩溃了还不用管，这能够保证程序的健壮性吗？我想，如果要了解这个机制，需要从下面几个方面去理解：

1. Actor的编程模型里，Actor之间是相互隔离的，不会共享状态，数据通过消息进行共享。所以，一个Actor如果发生崩溃，不会想C程序那样，摧毁整个程序栈。崩溃的Actor会死掉，监督机制会处理崩溃的Actor。

2. Actor之间可以相互监督，这种监督就像Linux进程中的watch dog进程那样，监控死去的Actor，如果发生错误，监督者可以让崩溃的Actor再复活。Actor的消息由系统管理，不会丢失。

3. 那如果监督的Actor死了怎么办？哈哈，这个通常是设计的问题，监督的Actor通常什么业务也不会做，不会带有让自身崩溃的代码。你想，警察局长做为监督者，怎么会跑到前线去跟绑匪开火。


那么下面，我们说一下，如何实现这个监督的Actor。我们会在代码里面创建两个Actor：

* Master Actor，负责进行监督，他启动后，会创建一个worker的Actor。

* Worker Actor，负责执行一个计算任务，发送过来的数，会被当作分母做除法计算。


如果我们发送一个0作为消息给worker，那么就会发生除法的0除错误，导致Worker Actor的崩溃。那么我们希望Master这时候知道Worker崩溃，然后重新启动他。

上代码：


--- master actor 部分 ---
```java
package learning.akka;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import akka.actor.AbstractActor;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;
import akka.actor.StoppingSupervisorStrategy;
import akka.actor.SupervisorStrategy;
import akka.actor.Terminated;

import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;

public class MasterActor extends AbstractActor {

	private static final Logger LOG = LoggerFactory
			.getLogger(MasterActor.class);
	private String name;
	private ActorRef worker_ref;

	public MasterActor(String name) {
		this.name = name; // "my test akka actor"
	}
	
	private static class SingletonHolder {
		private static ActorRef getActorRef() {

			Config config = ConfigFactory.load();
			ActorSystem system = ActorSystem.create("myakka", config);

			Props p = Props.create(MasterActor.class, "master actor");
			ActorRef ref = system.actorOf(p, "message_actor");
			return ref;
		}

		private static final ActorRef INSTANCE = getActorRef();
	}

	public static ActorRef reference() {
		return SingletonHolder.INSTANCE;
	}

	@Override
	public void preStart() throws Exception {
		//创建一个worker actor
		Props p = Props.create(WorkerActor.class, "worker:" + System.currentTimeMillis() );
		worker_ref = getContext().actorOf(p);

		// 监督这个worker，如果worker挂了，那么master能收到Terminate的消息
		getContext().watch(worker_ref);
		LOG.info("created worker actor");
		super.preStart();
	}

	@Override
	public SupervisorStrategy supervisorStrategy() {
		StoppingSupervisorStrategy s = new StoppingSupervisorStrategy();
		return s.create();
	}

	@Override
	public Receive createReceive() {
		return receiveBuilder().match(Integer.class, msg -> {
			// 将消息转发给worker
				if (worker_ref != null) {
					worker_ref.forward(msg, getContext());
				}

			}).match(Terminated.class, t -> {
			LOG.info("Got Terminate message:" + t);
		}).matchAny(msg -> {
			// 接收了一个其他的消息消息
				LOG.info("Got other message:" + msg);
			}).build();
	}

	public static void main(String[] args) throws Exception {
		// 发送一个字符串
		MasterActor.reference().tell(1, ActorRef.noSender());
		// 发送一个对象
		Thread.sleep(1000);
		LOG.info("Send zero message...");
		MasterActor.reference().tell(0, ActorRef.noSender());
		MasterActor.reference().tell(2, ActorRef.noSender());
	}

}


```

--- worker部分 ---

```java
package learning.akka;

import java.math.BigDecimal;
import java.util.Optional;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import akka.actor.AbstractActor;
import akka.actor.ActorRef;

public class WorkerActor extends AbstractActor {

	private static final Logger LOG = LoggerFactory
			.getLogger(WorkerActor.class);
	private String name;

	public WorkerActor(String name) {
		this.name = name; // "my test akka actor"
	}

	@Override
	public void preRestart(Throwable reason, Optional<Object> message)
			throws Exception {
		LOG.info(name + " actor started");
		super.preRestart(reason, message);
	}

	@Override
	public void postStop() throws Exception {
		LOG.info(name + " actor stopped");
		super.postStop();
	}

	@Override
	public Receive createReceive() {
		return receiveBuilder().match(Integer.class, msg -> {
			// 接收到了一个字符串的消息
				ActorRef sender = getSender();
				LOG.info(name + " got: {} from sender:{}", msg, sender);
				
				BigDecimal result = BigDecimal.ONE.divide(new BigDecimal(msg));
				LOG.info("Got divide result:{}", result);

			}).matchAny(msg -> {
			// 接收了一个其他的消息消息
				LOG.info("Got other message:" + msg);
			}).build();
	}

}


```

运行MasterActor的代码，你会看到

```
22:11:00.296 [myakka-akka.actor.default-dispatcher-2] INFO learning.akka.MasterActor - created worker actor
22:11:00.299 [myakka-akka.actor.default-dispatcher-5] INFO learning.akka.WorkerActor - worker:1545747060293 got: 1 from sender:Actor[akka://myakka/deadLetters]
22:11:00.303 [myakka-akka.actor.default-dispatcher-5] INFO learning.akka.WorkerActor - Got divide result:1
22:11:01.284 [main] INFO learning.akka.MasterActor - Send zero message...
22:11:01.285 [myakka-akka.actor.default-dispatcher-2] INFO learning.akka.WorkerActor - worker:1545747060293 got: 0 from sender:Actor[akka://myakka/deadLetters]
22:11:01.302 [myakka-akka.actor.default-dispatcher-4] INFO learning.akka.WorkerActor - worker:1545747060293 actor stopped
[ERROR] [12/25/2018 22:11:01.298] [myakka-akka.actor.default-dispatcher-5] [akka://myakka/user/message_actor/$a] Division by zero
java.lang.ArithmeticException: Division by zero
	at java.math.BigDecimal.divide(BigDecimal.java:1665)
	at learning.akka.WorkerActor.lambda$0(WorkerActor.java:42)
	at akka.japi.pf.UnitCaseStatement.apply(CaseStatements.scala:26)
	at akka.japi.pf.UnitCaseStatement.apply(CaseStatements.scala:21)
	at scala.PartialFunction.applyOrElse(PartialFunction.scala:123)
	at scala.PartialFunction.applyOrElse$(PartialFunction.scala:122)
	at akka.japi.pf.UnitCaseStatement.applyOrElse(CaseStatements.scala:21)
	at scala.PartialFunction$OrElse.applyOrElse(PartialFunction.scala:171)
	at akka.actor.Actor.aroundReceive(Actor.scala:517)
	at akka.actor.Actor.aroundReceive$(Actor.scala:515)
	at akka.actor.AbstractActor.aroundReceive(AbstractActor.scala:147)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:588)
	at akka.actor.ActorCell.invoke(ActorCell.scala:557)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:258)
	at akka.dispatch.Mailbox.run(Mailbox.scala:225)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:235)
	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

22:11:01.317 [myakka-akka.actor.default-dispatcher-3] INFO learning.akka.MasterActor - Got Terminate message:Terminated(Actor[akka://myakka/user/message_actor/$a#867853221])
[INFO] [12/25/2018 22:11:01.304] [myakka-akka.actor.default-dispatcher-5] [akka://myakka/user/message_actor/$a] Message [java.lang.Integer] without sender to Actor[akka://myakka/user/message_actor/$a#867853221] was not delivered. [1] dead letters encountered. This logging can be turned off or adjusted with configuration settings 'akka.log-dead-letters' and 'akka.log-dead-letters-during-shutdown'.

```

对这个程序进行几点说明

* 首先需要覆盖一下supervisorStrategy方法，默认的监督策略是不会在Exception出现时杀死WorkerActor的，感兴趣的话可以试试。

* MasterActor的forward的方法负责转发消息。forward和tell有什么区别？如果A发消息M给B，B forward消息给C，那么C用getSender()的方法得到的是A，而tell方式会返回B。

* Akka的监督机制是比较负责的，可以查看Akka的文档，默认是一种One2One的监督机制，如果一个Actor挂了，那就是对这个Actor进行处理。还有种One2All的策略，如果一个挂了，整个下级的Worker都可以进行罢工。