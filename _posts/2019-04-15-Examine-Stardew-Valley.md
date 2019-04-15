---
layout: post
title: ! "Stardew Valley 분석해보기 -1-"
excerpt_separator: <!--more-->
comments : true
tags:
  - KRater
  - Stardew Valley
  - Game
  - For Fun
---

### 서론

동아리 세미나를 준비하면서 처음에 기획했던 주제였는데, 생각보다 시간이 부족해서 세미나는 원래 하던걸로 떼우고... 분석하던 내용은 블로그에 정리하기로 했다.

<!--more-->

요즘 `Stardew Valley`라는 게임을 다시 하고 있는데, 한 140시간정도 한 것 같다. 꽤 재밌게 하고 있는데 문득 이 게임은 어떤 구조로 되어있을지 궁금해서 분석해보기로 했다. 다만 할 시간이 부족해서 짬짬히 시간날때마다 하니까 분석 진행은 느리다. 그래서 그냥 분석할때마다 올리는 장편으로 구성하기로 결정했다. (2편은 안 올라올수도 있다...)

### 본론

일단 분석 환경은 Mac이다. 게임 로컬 콘텐츠 폴더로 들어가 보자.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic01.png)

`Stardew Valley/Contents/MacOS`에는 실제 바이너리 등이 있고, `Stardew Valley/Contents/Resources/Content`에는 `XNB`파일들이 존재한다. 이는 마이크로소프트 XNA 게임 스튜디오에서 사용되는 파일 확장자이며 다양한 텍스트, 이미지, 사운드 파일들이 압축되어 있다고 생각하면 쉽다.

스타듀 밸리 게임에서 NPC들의 얼굴 스프라이트를 변환하기 위해서는 바로 이 `.xnb`파일을 대치한다. 이를 토대로 해당 파일을 변조하면 실제 게임 데이터에도 영향을 미친다고 판단하였고, 이를 어떻게 변조할 지 검색하게 되었다.

먼저 XNB 파일을 우리가 사용할 수 있는 원래의 텍스트, 이미지, 사운드 파일로 변환해야 한다. 이를 도와주는 도구가 바로 `XNB-Converter for Mac/Linux`라는 도구이다. 해당 도구는 [링크](https://community.playstarbound.com/threads/xnb-convert-for-mac-linux.140272/)에서 받을 수 있으며, Windows의 경우에는 더 대중적으로 다른 도구들이 많이 있는 것 같다.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic02.png)

설치하면 위와 같이 뜬다. Pack 폴더에 `.xnb`파일을 넣고 'Unpack을 실행시키면 Unpack 폴더에 변환된 파일들이 뜨게 된다. 참고로 그냥은 실행이 안되서 `ctrl+click` 으로 실행시켜 주어야 한다. 한번 실행시켜주면 다음부터는 그냥 더블클릭으로도 실행 된다.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic03.png)

로컬 컨텐츠 폴더에서 `.xnb` 파일들을 긁어다 넣고 'Unpack을 실행시키자.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic04.png)

실행시키면 저렇게 변환되는 것을 볼 수 있다.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic05.png)

변환 결과 파일들은 Unpack 폴더에 저장된다. `.png`, `.yaml`등 원래의 파일 형식으로 알아서 변환된다.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic06.png)

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic07.png)

그중 `CraftingRecipes.ko-KR.yaml` 파일을 찾아서 텍스트 편집기 어플리케이션으로 열어봤더니, 위와 같이 나타나 있었다. 그런데 [스타듀밸리 아이템 코드를 정리해놓은 사이트](https://www.ign.com/wikis/stardew-valley/Item_Codes_for_Spawning_Cheat)에서 찾아보니 388은 Wood(목재), 322는 Wood Fence(나무 울타리)로 지정되어 있는 것을 알 수 있었다. 이를 토대로 해당 레시피는 Wood 2개로 Wood Fence를 만들 수 있는 레시피라고 추측하였다.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic08.png)

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic09.png)

그래서 해당 부분의 `388 2` 부분을 `388 0`으로 수정한 뒤, 이를 'Pack 파일을 실행시켜서 다시 `.xnb` 파일로 패킹한 뒤, 다시 원래의 게임 폴더에 넣어 보았다.

![]({{ site.baseurl }}/images/KRater/2019-04-15-Examine-Stardew-Valley/pic10.png)

실제 게임 레시피가 변화된 것을 볼 수 있었다.

### 결론

이를 토대로 실제 XNB 파일을 변조하면 게임 데이터에도 영향을 줄 수 있다는 것을 파악하였다. 뭔가 더 재밌는 영향을 주고싶어서 언패킹된 모든 `.xnb`파일들을 분석하고 있는데, 이게 생각보다 오래걸리는 작업이고 `.xnb` 파일에 포함된 실제 데이터는 미미한 수준인 것 같다. 아무튼 분석은 계속 진행해 봐야겠다. 끄읏~

