---
layout: post
title:  "在actor中使用timer"
date:   2018-12-16 22:34:25
categories: Akka actor java
---

在编程中，经常需要使用定时器来完成某些任务，例如定时发送一个邮件给用户，告诉他们今天他们完成了多少运动量。或者定时清理系统中的一些资源，让系统的可用空间保持在一个合理的水平上。那么在java开发中，使用timer完成任务，大概有两个方案：
1. 使用java自带的Timer定时器。
2. 使用一些第三方的库，例如Quartz。


对于Akka的Actor来说，Timer的定时任务是内建的机制，你可以用Actor的Timer来实现定时任务。Akka的Timer有这样一些特点：

* Timer的实现是基于时间轮算法的，所以，你不用担心上百万个Actor的Timer设置导致很高的资源消耗。在时间轮的方式下，每秒中要进行调度的actor都是在固定集合里的，不会为查找消耗大量的计算。

* Actor的Timer最原始的用途在于设置一个固定的频率，当这个时间段内Actor没有收到任何消息时，可以让Timer触发一个Timeout消息，Actor可以根据这个消息关闭一些资源。利用这一机制，可以实现系统定时的任务。

* 由于Actor的内部机制是单线程的，所以当actor不断有消息进入actor时，这个Timer就不会被触发，所以，不要让一个繁忙的Actor充当Timer。

怎么用，上代码：

```java
package learning.akka;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import scala.concurrent.duration.Duration;
import akka.actor.AbstractActor;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;
import akka.actor.dungeon.ReceiveTimeout;

import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;

public class AkkaTimerActor extends AbstractActor {

	private static final Logger LOG = LoggerFactory
			.getLogger(AkkaTimerActor.class);
	private String name;

	public AkkaTimerActor(String name) {
		this.name = name; // "my test akka actor"
	}

	private static class SingletonHolder {
		private static ActorRef getActorRef() {

			Config config = ConfigFactory.load();
			ActorSystem system = ActorSystem.create("myakka", config);

			Props p = Props.create(AkkaTimerActor.class, "my test akka actor");
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

		// 设置一个定时器，每5秒执行一次
		Duration timeout = Duration.create(5, "seconds");
		getContext().setReceiveTimeout(timeout);
		LOG.info("AkkaTimerActor started");
		super.preStart();
	}

	@Override
	public void postStop() throws Exception {
		super.postStop();
	}

	@Override
	public Receive createReceive() {
		return receiveBuilder().match(ReceiveTimeout.class, msg -> {
			// 接收到了一个字符串的消息
				ActorRef sender = getSender();
				LOG.info("Got: {} from sender:{}", msg, sender);

			}).matchAny(msg -> {
			// 接收了一个其他的消息消息
				LOG.info("Got other message:" + msg);
			}).build();
	}

	public static void main(String[] args) {
		// 启动Timer，只要actor初始化了，那么定时器就被激活了
		AkkaTimerActor.reference();
		try {
			Thread.sleep(20000); //主线程休息20秒，等待输出
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

}



```

运行这个Actor的代码，可以看到下面的结果。

```bash
23:39:18.800 [myakka-akka.actor.default-dispatcher-5] INFO learning.akka.AkkaTimerActor - AkkaTimerActor started
23:39:23.819 [myakka-akka.actor.default-dispatcher-4] INFO learning.akka.AkkaTimerActor - Got other message:ReceiveTimeout
23:39:28.838 [myakka-akka.actor.default-dispatcher-4] INFO learning.akka.AkkaTimerActor - Got other message:ReceiveTimeout
23:39:33.857 [myakka-akka.actor.default-dispatcher-4] INFO learning.akka.AkkaTimerActor - Got other message:ReceiveTimeout
```

所以，如果使用Akka的Actor开发，定时任务也是很容易实现的。甚至，你可以从外部发送消息，让Timer的执行进行判断，打开或者关闭特定时间的运行。