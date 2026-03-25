---
title: 인터럽트와 시스템 호출
weight: 9
draft: true
---

```asm
global _start

section .text

_start:
  mov rax, 1        ; write(
  mov rdi, 1        ;   STDOUT_FILENO,
  mov rsi, msg      ;   "Hello, world!\n",
  mov rdx, msglen   ;   sizeof("Hello, world!\n")
  syscall           ; );

  mov rax, 60       ; exit(
  mov rdi, 0        ;   EXIT_SUCCESS
  syscall           ; );

section .rodata
  msg: db "Hello, world!", 10
  msglen: equ $ - msg
```

인터럽트(Interrupt)는 비용이 많이 듭니다. 인터럽트는 정상적인 실행 경로에 있어서는 안 됩니다. 예외(Exception)도 마찬가지입니다.

시스템 호출(system call)을 수행하는 데에는 어느 정도의 오버헤드가 따르므로 일반적으로 이를 피합니다. 예를 들어, 모든 I/O는 보통 버퍼링되어, OS에 4KB와 같은 단일 데이터 조각을 한 번에 보냅니다.
