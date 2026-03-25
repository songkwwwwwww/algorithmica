---
title: 이벤트 기반 동시성 (Event-driven Concurrency)
weight: 4
---

JavaScript에서 이러한 방식을 보셨을 것입니다. 이러한 언어는 지속적으로 실행되는 이벤트 루프(event loop)를 가집니다. 또한 대부분의 블로킹 연산이 I/O이기 때문에 단일 스레드를 사용합니다.

```js
var callback = function() {
    console.log("Button clicked")
}

document.getElementById('someButton').addEventListener("click", callback)
```

이벤트 기반 환경은 대개 단일 스레드이며, 요청을 "샤딩(sharding)"하여 멀티스레딩과 같은 효과를 냅니다.

## 액터 모델 (Actor Model)

보다 일반화된 접근 방식을 *액터 모델(actor model)*이라고 합니다.

이것은 JVM 생태계에서 매우 인기가 있습니다.

```scala
import akka.actor.Actor
import akka.actor.ActorSystem
import akka.actor.Props

class HelloActor extends Actor {
  def receive = {
    case "hello" => println("hello back at you")
    case _       => println("huh?")
  }
}

object Main extends App {
  val system = ActorSystem("HelloSystem")
  // default Actor constructor
  val helloActor = system.actorOf(Props[HelloActor], name = "helloactor")
  helloActor ! "hello"
  helloActor ! "buenos dias"
}
```

메시지 브로커를 사용하는 매우 중요한 장점 중 하나는 통신을 분리(decouple)할 수 있고, 액터를 다른 네트워크 노드로 이동시켜 분산 컴퓨팅을 가능하게 한다는 것입니다.
