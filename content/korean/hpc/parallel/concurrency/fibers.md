---
title: 파이버 (Fibers)
weight: 3
---

*파이버(Fibers)*는 프로그래밍 언어 자체에서 구현된 경량 스레드입니다. 이것은 실행 중단된 지점에서 다시 작업을 재개하는 방식으로 작동하며, 따라서 언어 차원에서 자체적인 런타임을 유지해야 합니다.

```go
package main

import (
	"fmt"
	"time"
)

func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
	go say("world")
	say("hello")
}
```

파이버가 작동하는 방식은 언어가 중단된 지점부터 다시 시작할 준비가 된 스레드 그룹을 관리하는 것입니다. 이를 N:M 스케줄링이라고 부릅니다.

C++나 Rust와 같은 다른 언어들에도 이와 유사한 런타임이 존재합니다.
