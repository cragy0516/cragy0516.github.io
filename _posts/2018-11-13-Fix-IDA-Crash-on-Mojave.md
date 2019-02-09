---
layout: post
title: ! 'Fix the problem that IDA Pro 7.0 Crash on MacOS Mojave'
excerpt_separator: <!--more-->
comments: true
tags:
  - Bug Fix
---

MacOS Mojave에서 IDA pro 7.0을 구동하면 크래쉬가 터지는 버그가 존재합니다. 이번 포스팅에서는 이에 대한 패치를 안내해드리도록 하겠습니다.

<!--more-->

## 개요

최근에 Mac을 Mojave로 업그레이드 한 이후, IDA pro가 원인모를 크래쉬에 의해 터지면서 어려움을 겪었습니다. 이와 비슷한 이슈를 겪는 분들을 위해서 해결방법을 알려드리도록 하겠습니다. 저는 해당 오류를 검색 시 한글로 입력하기 때문에 발생한다고 생각했었는데, 원인은 이와 비슷하지만 조금 더 범위가 넓었습니다. 어쩐지 오류가 발생하는 시점이 제 입장에서는 무작위 같아 보이더군요.

![]({{ site.baseurl }}/images/KRater/2018-11-13-Fix-IDA-Crash-on-Mojave/pic01.png)

해당 이슈는 MacOS Mojave에서 IDA pro 7.0에 영어가 아닌 입력 형식을 적용할 경우에 발생합니다. 따라서 한글 입력을 엄청난 정신력으로(?) 막아내면서 영어만 사용하시면 그냥 그대로 사용하실 수 있습니다. 하지만 카카오톡 등등을 사용하느라 깜빡하고 한글 입력 상태를 영어 입력으로 바꾸지 않았다는 이유만으로 크래쉬가 터지는 것은 굉장히 불합리한 일입니다. 다행히 이러한 점을 규명한 분이 [Github에 IDA Pro 7.0을 위한 긴급 패치](https://github.com/fjh658/IDA7.0_SP)를 만들어 적용법까지 상세하게 올려주셨습니다. 원인 파악도 꽤 자세하게 적혀있는 편 이네요.

적용 방법은 Github에 나와있듯이 `libqcocoa.dylib` 파일을 다운로드 받아서 `/Applications/IDA Pro 7.0/ida.app/Contents/PlugIns/platforms/libqcocoa.dylib` 파일을 덮어쓰면 됩니다. 아주 깔끔하게 적용되네요! 이에 대한 원인 분석도 나와있는데 나중에 시간이 좀 여유로울 때 한번 확인 해 보아야 겠습니다~