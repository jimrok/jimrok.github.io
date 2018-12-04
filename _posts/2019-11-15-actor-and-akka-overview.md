---
layout: post
title:  "actor和akka"
date:   2018-11-15 22:34:25
categories: Akka actor java
---

先说actor这个概念，这个模型是70年代Carl Hewitt首次提出的，然后Erlang这门语言成为了actor模型最典型的代表。你可以从网上找到关于这个模型的讲解。然后一个java程序员，如果想要体验一下这个actor编程模型，那么就必然会找到akka。akka作为一个框架，它是用scala语言实现的，但它也提供了java的编程接口。

目前使用java编写akka的actor的文章比较少，大部分都是scala的，对于想初步尝试akka编程的程序员来说，不太友好。所以，我准备了第一个文章，从一个简单的例子来学习akka的编程。同时每个编程里例子都是有实际意义的编程问题，而不是hello world这种无聊的例子。下面就开始吧，我尽量少用废话。

### akka的第一个例子--异步编程

1. 准备环境，在java的工程文件里增加下列的依赖

```xml

<!-- https://mvnrepository.com/artifact/com.typesafe.akka/akka-actor -->
<dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-actor_2.12</artifactId>
    <version>2.5.18</version>
</dependency>

```

2. 使用java8的开发环境来配置工程

3. 上代码。

```java

package learning.akka;

import java.util.ArrayList;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import akka.actor.AbstractActor;
import akka.actor.ActorRef;
import akka.actor.ActorSystem;
import akka.actor.Props;

import com.typesafe.config.Config;
import com.typesafe.config.ConfigFactory;

public class AkkaMessageActor extends AbstractActor {

	private static final Logger LOG = LoggerFactory
			.getLogger(AkkaMessageActor.class);
	private String name;

	public AkkaMessageActor(String name) {
		this.name = name; // "my test akka actor"
	}

	private static class SingletonHolder {
		private static ActorRef getActorRef() {

			Config config = ConfigFactory.load();
			ActorSystem system = ActorSystem.create("myakka", config);

			Props p = Props.create(AkkaMessageActor.class,"my test akka actor");
			ActorRef ref = system.actorOf(p, "message_actor");
			return ref;
		}

		private static final ActorRef INSTANCE = getActorRef();
	}

	public static ActorRef reference() {
		return SingletonHolder.INSTANCE;
	}

	@Override
	public Receive createReceive() {
		return receiveBuilder().match(String.class, msg -> {
			// 接收到了一个字符串的消息
				ActorRef sender = getSender();
				LOG.info("Got: {} from sender:{}", msg, sender);

			}).matchAny(msg -> {
			// 接收了一个其他的消息消息
				LOG.info("Got other message:" + msg);
			}).build();
	}

	public static void main(String[] args) {
		//发送一个字符串
		AkkaMessageActor.reference().tell("Hello World", ActorRef.noSender());
		//发送一个对象
		AkkaMessageActor.reference().tell(new ArrayList<String>(), ActorRef.noSender());
	}

}


```


 现在我们对这个代码做一个简单的说明：

 * 当你调用这句代码时候，AkkaMessageActor.reference().tell("Hello World", ActorRef.noSender()); 会发送一个消息给一个“Hello World"的String对象给AkkaMessageActor的对象。receiveBuilder中的方法将会异步处理接受到的消息。

 * 你可以发送其他的对象给AkkaMessageActor的对象，如果不是String的类型的，那么将会打印出“Got other message:”的信息。

 * 在这个例子中，AkkaMessageActor.reference()是一个单例的模式，你可以使用这个单例模式，两次把消息发给同一个Actor对象。

 * 最后，这个例子可以用来替代java的异步操作，对java稍微熟练的同学，都会使用ExecutorService.newSingleThreadExecutor创建一个异步执行的功能，akka的actor可以做一样的事情。

 ### 为啥要用Akka呢

 当你看完这个例子后，你可能会觉得，java的异步挺好的，为啥要用akka来替代呢？发送一个消息不就类似为异步的Task对象提供一个参数吗？直接用JDK里ExecutorService不是挺好，用把事情搞得那么复杂吗。那答案就是通常情况下，ExecutorService就够用了，但akka可以把更复杂的事情变得简单。


 1. 首先，一个Actor的内部，消息的处理是单线程的，也就是一个消息没有处理完，是不会同时处理第二个消息的。可以想象每个Actor有一个无限大的消息队列，Actor只会接受这个消息队列的消息。这样的话，你可以考虑下里面这些控制：

    * 第10个消息后，就不处理了，类似先到先得。
    * 新消息和上一个消息比较，如果没有变化，就不去更新系统的某个值。
    * 等待两个消息都到达后，再执行一个操作。
    * 某个资源不满足条件时，先把消息缓存着，等满足时，一次批处理。

    思考一下这些问题，是否在ExecutorService的方式下容易处理？在Actor的方式下呢？

2. Actor不但能控制状态，而且还有其他的比较厉害的地方。下次，我们再增加一些代码，看看
Akka框架下的actor还有什么有意思的地方。