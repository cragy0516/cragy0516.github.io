---
layout: post
title: ! "[Codegate 2019] KingMaker Write-up"
excerpt_separator: <!--more-->
comments : true
tags:
  - Codegate2019
  - Write-up
  - CTF
---

### Description

> nc 110.10.147.104 13152

<!--more-->

### 문제

Codegate2019 예선에 나왔던 문제. 리버싱 + 간단한 게임을 섞어놓은듯 한 문제였다. 한 왕자(플레이어)가 선택지를 골라 자신의 능력치를 키워가며 왕이 되어가는 과정에서, 적절한 리버싱을 통해 키 값을 유추하고 적절한 선택지를 통해 능력치를 조건에 맞게 키우면 된다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic01.PNG)

왕은 test 1, 2, 3... 에 있어서 플레이어에게 key값을 요구하며, 처음에는 이 값이 4 bytes였으나 나중으로 가면 더 길어진다. 먼저 key1을 받는 부분을 분석하면 해당 key 값을 통해서 sub_400AB9에서 무언가 조작을 한 뒤, 이를 토대로 loc_40341D 지점을 호출하는 것을 확인할 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic02.PNG)

sub_400AB9를 분석해보면 입력 값 및 미리 주어진 값을 이용해서 바이트마다 xor 연산을 수행한다. 그리고 해당 결과를 다시 저장하는것도 볼 수 있다. 이 지점은 key1 입력 시점 기준에서 loc_40341D 부분인데, 결국 자신이 입력한 key1 값과 loc_40341D을 xor한 뒤 해당 부분을 그대로 호출해주는 것을 알 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic03.PNG)

loc_40341D 부분은 알 수 없는 값으로 들어가있는데, 이는 xor 연산을 통해 올바른 인스트럭션으로 손쉽게 복호화할 수 있다. 함수의 처음은 보통 **push rbp**, **mov rbp, rsp**로 시작하는 것을 이용하자. 이를 바이트로 옮기면 **0x55 0x48 0x89 0xE5**가 되어야 한다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic04.PNG)

loc_40341D의 hex dump를 확인 해 보면 처음 4 bytes가 0x39 0x07 0xFF 0xD6임을 확인할 수 있다. 따라서 xor의 성질을 이용해서 key1의 4 bytes를 유추할 수 있다. 0x55^0x39=**0x6C**, 0x48^0x07=**0x4F**, 0x89^0xFF=**0x76**, 0xE5^0xD6=**0x33**의 결과를 얻을 수 있다. King에게 해당 key1을 넣으면 이후 게임이 진행된다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic05.PNG)

이후 진행되는 게임은 왕의 시험에 해당하는 일종의 대화문이 나온 뒤 선택지를 고르면, 해당 선택지에 맞춰 게임 오버가 되거나 캐릭터의 능력치가 정해진 수치에 맞추어 올라가는 형식이다.

key2까지는 key1과 같은 방식으로 해결이 가능한데, key3부터는 길이가 10 bytes로 다소 증가하였다. 따라서 프롤로그 말고도 다른 부분을 추가로 분석해서 추측해야 key 전체를 알아낼 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic06.PNG)

보통 에필로그 부분은 **call __stack_chk_fail** 이후 **leave ret** 명령으로 이루어져 있다. 다만 주의해야 하는게 call이 상대 주소를 이용해서 함수를 호출하기 때문에 위치마다 조금씩 달라진다는 것이다. 에필로그 부분은 **E8 ?? ?? ?? ?? C9 C3**의 6 bytes를 확인할 수 있는데, 물음표로 처리된 부분은 오프셋을 뜻한다. 오프셋은 `목적지 주소 - 현재 명령어 주소 -5`의 공식으로 손쉽게 구할 수 있으며, ___stack_chk_fail 함수의 주소는 **0x400850**이므로 위 그림에서 **0x4024D5** 위치의 바이트들을 역 추론하면 **0x400850 - 0x4024D5 - 5 = 0xFFFFE376**에 의해서

**E8 76 E3 FF FF C9 C3**

이 될 것이라고 예측할 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic07.PNG)

이렇게 역추론해서 key값을 구하는 방식으로 key3, key4, key5 역시 구할 수 있다. key를 모두 입력하여 게임을 계속 진행하면 마지막에 도달할 수 있다. 마지막까지 도달하면 지금까지 얻은 능력치를 이용하여 왕이 평가를 한다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic08.PNG)

모든 능력치가 5가 되어야 하는 조건이 있으며, 이를 위해서는 각 선택지를 적절하게 배분할 필요가 있다. 선택지에 따른 능력치 변화를 간략화하여 정리하여, 아래와 같은 결과를 얻을 수 있었다. 동시에 선택할 수 없는 선택지의 경우에는 빈 줄이 없으며, 선택할 때 상관이 없는 경우에는 빈 줄을 추가하였다. 또한 중간 중간 고르지 않으면 게임이 반드시 끝나버리는 경우에는 (필수)를 붙였고, 고를 경우 게임 오버되는 경우는 X를 붙였다.

```
1 (필수)

2 (필수)

1. (2/0/0/1/0)
2. (2/0/1/0/0)
3. (2/0/2/1/0)

1. (0/0/1/0/2)
2. (0/-1/0/0/-1)
3. (0/2/0/0/0)

test 1 끝

1 (필수)

1. (-1/0/-1/1/0)
2-1-1. (1/1/0/0/0)
2-1-2. (1/1/0/0/0)
3-1. (1/2/0/0/0)

1-1. (1/1/0/2/0)
1-2. (1/1/1/2/0)
2-1. (1/1/1/1/2)
2-2. (1/2/2/1/2)

test 2 끝

1. (0/0/1/1/0)
2. (0/-1/2/0/0)
3. (0/-1/1/1/0)

1-2. (1/-1/-1/2/2)
2-1. (0/0/0/0/0)
2-2. X
2-3. (1/0/0/0/1)

test 3 끝

1-1-AAAA.  (0/1/1/2/0)
2.X

1-1-1. X
1-1-2. (0/1/1/1/0)
1-2. (0/1/0/0/3)
2. (0/1/0/0/0)

test 4 끝

1. X
2-1-1. (-1/0/0/1/1)
2-1-2. (0/0/1/2/1)
2-2. X
3-1. X
3-2. (0/0/0/2/2)
3-3. X

2 (필수)

1 (필수)
```

이제 점수를 모두 정리했으니 전수조사를 통해 능력치를 5/5/5/5/5로 만드는 경우를 찾을 수 있다.

```python
from numpy import sum 

l = []
l.append ([[2,0,0,1,0],     [2,0,1,0,0],    [2,0,2,1,0]])
l.append ([[0,0,1,0,2],     [0,-1,0,0,-1],  [0,2,0,0,0]])
l.append ([[-1,0,-1,1,0],   [1,1,0,0,0],    [1,1,0,0,0],    [1,2,0,0,0]])
l.append ([[1,1,0,2,0],     [1,1,1,2,0],    [1,1,1,1,2],    [1,2,2,1,2]])
l.append ([[0,0,1,1,0],     [0,-1,2,0,0],   [0,-1,1,1,0]])
l.append ([[1,-1,-1,2,2],   [0,0,0,0,0],    [1,0,0,0,1]])
l.append ([[0,1,1,2,0]])
l.append ([[0,1,1,1,0],     [0,1,0,0,3],    [0,1,0,0,0]])
l.append ([[-1,0,0,1,1],    [0,0,1,2,1],    [0,0,0,2,2]])

for a in range(0, 3): 
    for b in range(0, 3): 
        for c in range(0, 4): 
            for d in range(0, 4): 
                for e in range (0,3):
                    for f in range (0, 3): 
                        for g in range (0, 1): 
                            for h in range (0, 3): 
                                for i in range (0,3):
                                    tmp = sum([l[0][a], l[1][b], l[2][c], l[3][d], l[4][e], l[5][f], l[6][g], l[7][h], l[8][i]], axis=0).tolist()
                                    if tmp.count(5) == 5:
                                        print (a, b, c, d, e, f, g, h, i)
                                        print tmp
```

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic09.PNG)

전수조사를 통해 5가 5개 들어가는 리스트를 찾았으므로, 얻은 결과를 토대로 exploit 코드를 작성하였다.

```python
from pwn import *

HOST = "110.10.147.104"
PORT = 13152
r = remote(HOST, PORT)
#r = process("./KingMaker")
#r = process(["gdb-peda","./KingMaker"])
#r.sendline("b *0x401CBA")
#r.sendline("r")

def sel(num):
    print r.recvuntil(">")
    r.sendline(str(num))

def main (): 
    r.recvuntil("around")
    r.sendline("1")
    print r.recvuntil("test 1")
    f = open("./key1", "r")
    #r.sendline("\x6C\x4F\x76\x33\x00")
    r.sendline(f.read())
    f.close()
    sel(1)
    sel(2)
    sel(2)
    sel(3)

    print r.recvuntil("test 2")
    #r.sendline("\x44\x30\x6c\x31\x00")
    f = open("./key2", "r")
    r.sendline(f.read())
    f.close()
    sel(1)
    sel(2)
    sel(1)
    sel(1)
    sel(2)
    sel(1)    print r.recvuntil("test 3")
    #r.sendline("\x48\x75\x4e\x67\x52\x59\x54\x31\x0D\xF6\x00")
    f = open("./key3", "r")
    r.sendline(f.read())
    f.close()
    sel(2)
    sel(2)
    sel(3)

    print r.recvuntil("test 4")
    f = open("./key4", "r")
    r.sendline(f.read())
    f.close()
    sel(1)
    sel(1)
    r.sendline("AAAA")
    sel(2)

    print r.recvuntil("test 5")
    f = open("./key5", "r")
    r.sendline(f.read())
    f.close()
    sel(3)
    sel(2)
    sel(2)
    sel(1)

    r.interactive()


if __name__ == "__main__" :
    main()
```

입력 시 플래그를 획득할 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-02-09-codegate2019-kingmaker-writeup/pic10.PNG)