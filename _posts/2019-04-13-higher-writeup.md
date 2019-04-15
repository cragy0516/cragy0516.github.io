---
layout: post
title: ! "[volgaCTF 2019] higher"
excerpt_separator: <!--more-->
comments : true
tags:
  - KRater
  - volgaCTF2019
  - Write-up
  - CTF
---

### Description

> Higher
>
> Take higher
>
> [recorded.mp3](https://q.2019.volgactf.ru/files/7863b4733a16ece2b4cefd899489017d/recorded.mp3)


<!--more-->

### 문제 풀이

처음에 무엇을 써야 할 지 잘 몰라서 못 푼 문제. 단순한 Audio Steganography인데, 대회 도중에는 다른 문제 보느라 이거 볼 겨를이 없었다. 문제 해결에는 [Sonic Visualiser](https://www.sonicvisualiser.org/)를 사용하였다.

![]({{ site.baseurl }}/images/KRater/2019-04-13-higher-writeup/pic01.PNG)

스펙트럼을 확인 해 보면 위와 같다. 짧고 긴 파장을 0과 1로 생각해서 비트를 모두 구한 뒤, 이를 텍스트로 변환하면 된다. 하지만 위와 같은 상태에서는 옮겨적다가 실수하기가 쉽다. 따라서 Sonic Visualiser에서 제공하는 도구를 통해 파장을 분리하기 쉽게 변형하는 과정이 필요했다.

![]({{ site.baseurl }}/images/KRater/2019-04-13-higher-writeup/pic02.PNG)

오른쪽에 있는 도구를 이용해서 파장을 변환하니 소리는 금색, 소리 사이의 여백이 파란색으로 좀 더 눈에 띄도록 바뀐 것을 볼 수 있다. 이를 통해 모든 비트를 구하면 다음과 같다.

```
010101100110111101101100011001110110000101000011010101000100011001111011010011100011000001110100010111110011010001101100011011000101111101100011001101000110111001011111011000100011001101011111011010000011001100110100011100100110010001111101
```

이를 텍스트로 변조하자. 사이트는 [이곳](https://cryptii.com/pipes/binary-decoder)을 활용하였다.

```VolgaCTF{N0t_4ll_c4n_b3_h34rd}```