---
layout: post
title: ! ' [GoogleCTF2018 Quals] Begginers Quest (1) '
excerpt_separator: <!--more-->
comments: true
tags:
  - Write-up
  - CTF
---

구글 CTF 당일에 데통 데모 웹프 데모가 겹쳐서 CTF를 못했다. 그런데 서버가 열려있길래 문제나 풀어보려고 했는데 어렵더라.. 그래서 빠르게 풀 수 있는 `Begginer's Quest`로 눈을 돌렸다 ㅎ
물론 아직 푸는데 시간이 좀 걸린다. 그래도 풀 수 있는게 어디인가.. 까먹기전에 롸업을 정리해두려고 한다.

<!--more-->

![]({{ site.baseurl }}/images/KRater/2018-07-05-Begginers-Quest-1/pic01.PNG)

몇 문제 안풀었다. 가장 쉬운거 몇개? 한시간정도 걸렸는데 까먹기전에 정리해두자.



## Letter

> LETTER
>
> Garbo-can
> 
> misc
>
> You really went dumpster diving? Amazing. After many hours, SUCCESS! Between what looks like a three week old casserole and a copy of "Relative-Time Magazine", you found this important looking letter about the victims PC. However the credentials aren't readable - can you still obtain them?

pdf 파일을 하나 주는데, 패스워드가 가려져 있다. 그냥 메모장에 복사-붙여넣기하면 가려진게 보인다. 매우 쉬운 문제.



## OCR is cool

> OCR IS COOL!
>
> Foobanizer9000
>
> misc
>
> Caesar once said, don't stab me… but taking a screenshot of an image sure feels like being stabbed. You connected to a VNC server on the Foobanizer 9000, it was view only. This screenshot is all that was present but it's gibberish. Can you recover the original text?

힌트에서 보이듯 시져 암호 문제다. 이번엔 첨부파일로 이메일을 찍은 스크린샷을 주는데, { } 괄호를 찾아서 그 앞의 VMY가 CTF라는점에서 역으로 유추하면 플래그를 빨리 찾을 수 있다. 소문자로 작성해야 하는데 대문자로 작성해서 안되서 시간을 많이 잡아먹었다..



## Floppy

> FLOPPY
>
> Foobanizer9000
>
> misc
>
> Using the credentials from the letter, you logged in to the Foobanizer9000-PC. It has a floppy drive...why? There is an .ico file on the disk, but it doesn't smell right..

간단한 스테가노그래피. 파일 헤더를 잘 찾아보면 ZIP 파일 헤더가 있다. 잘라내서 새로운 압축 파일 만들고 안에 메모장을 보면 플래그가 있다.



## Moar

> MOAR
>
> Foobanizer9000
>
> pwn
>
> Finding yourself on the Foobanizer9000, a computer built by 9000 foos, this computer is so complicated luckily it serves manual pages through a network service. As the old saying goes, everything you need is in the manual.
>
> moar.ctfcompetition.com 1337

들어가면 대뜸 socket 메뉴얼을 보여준다. 메뉴얼에서 help로 잘 읽어보면

> MISCELLANEOUS COMMANDS
>
>   -\<flag\>              Toggle a command line option [see OPTIONS below].
>  ...
>
>   !command             Execute the shell command with $SHELL.
>
> ...

쉘 명령어을 실행시킬 수 있다. 이걸로 home directory를 찾아서 `.sh` 파일을 실행시키면 끝.



## Floppy2

> FLOPPY2
>
> Foobanizer9000
>
> misc
>
> Looks like you found a way to open the file in the floppy! But that www.com file looks suspicious.. Dive in and take another look?

Floppy 문제에서 얻은 파일 중에 www.com 이라는 파일이 있다. MS-DOS 파일이라 DOSBox로 열어야 한다. 열어보면 단순히

> The Foobanizer9000 is no longer on the OffHub DMZ.

라고만 나온다. 정확한 프로그램의 동작을 분석하기 위해서 DOSBox Debugger를 사용해야 하는데, 대부분 리눅스 전용으로 된 설치파일에 추가 옵션을 주어서 설치하도록 권장하고 있었고, Windows버전은 찾을 수 없었다. 그러다가 [한국 DOS 포럼 카페](http://cafe.daum.net/dosbox)에서 우연히 DOSBox 확장판(SVN)을 얻을 수 있었고, 여기서 Debugger를 찾아서 사용하였다. 명령어는 [외국 포럼](https://www.vogons.org/viewtopic.php?t=3944)을 참고하였다.

![]({{ site.baseurl }}/images/KRater/2018-07-05-Begginers-Quest-1/pic02.PNG)

`debug www.com` 명령어로 디버깅에 진입한 뒤, 프로그램의 끝까지 가 보니 `interrupt 21`으로 프로그램이 종료된다. 그 전에 여러 반복문과 함수를 호출하며 연산을 진행하는데, 아마 플래그를 만들어 주는 과정인 듯 해서 `BPINT 21` 명령어로 프로그램의 끝인 21번 인터럽트에 브레이크 포인트를 걸었다.

![]({{ site.baseurl }}/images/KRater/2018-07-05-Begginers-Quest-1/pic03.PNG)

그리고 `F5`를 눌러서 실행해보니 data 영역에 플래그가 들어온것을 볼 수 있었다.

계속 풀어볼 예정!
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzUyNjUyOTc4XX0=
-->