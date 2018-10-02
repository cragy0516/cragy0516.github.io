---
layout: post
title: ! ' Hello, PyJail! '
excerpt_separator: <!--more-->
tags:
  - KRater
  - Write-up
  - CTF
  - ChristmasCTF2017
  - noxCTF2018
  - python
---

저는 오늘 PyJail 문제 2개를 들고왔습니다. 그럼 저희 같이 PyJail이 뭔지 알아보고, 그리고 이와 관련된 문제들을 풀어보면서 실전 감각을 익혀보도록 합시다.

<!--more-->

## PyJail 이란?

Python Jail, 또는 PyJail 이라고 불리우는 분야는 그 이름에서도 유추할 수 있듯, 파이썬(Python) + 감옥(Jail)의 합성어입니다. 이 분야의 문제들은 파이썬으로 동작하는 어플리케이션에서 악의적인 입력값을 통해 제한 조건을 벗어나 악의적인 방향으로 흐름을 조작하는 것이 목표입니다. 기본적으로 CTF의 가장 쉬운 문제 중 일부를 담당하고 있죠. 대부분은 `misc`로서 출제되지만, 어려운 문제의 경우, 또는 출제자의 의도에 따라 `pwnable`로도 충분히 분류될 수 있는 분야입니다. PyJail 분야의 문제를 풀기 위해서는 먼저 `Python`의 잘 알려진 몇 가지 특징에 대해서 짚고 넘어갈 필요가 있습니다.

`Python`은 모든 분들이 알고 있듯이, `interpreted language`입니다. 이는 언어로 작성된 코드를 미리 컴파일하여 실행 코드로 변환한 뒤 실행하는 `compiled language`와는 상반되게, 인터프리터가 언어를 읽어들이면서 바로 해석하고 실행한다는 특징을 가지고 있습니다. 또한 `dynamic typing`을 지원합니다. 이는 변수, 표현식, 함수, 모듈 등이 런타임에 결정된다는 뜻입니다. 이 둘은 매우 중요한 특징인데, 어플리케이션에 악의적인 유저 입력을 넣고 이를 실행시키는 과정이 다른 언어들에 비해서 크게 자유로운 편이라는 것입니다. 이는 문제를 풀어보면서 자세히 알아가보도록 합시다.

마지막으로 Python 2 버전과 Python 3 버전은 약간의 문법 차이가 존재한다는 특징도 알아둬야 합니다. 최신 Python 3 문법을 많이 사용하기는 하지만, 여전히 몇 개의 모듈들은 Python 2 버전만을 지원하므로 이는 알아둬야 하는 특징입니다. 예를 들면 Python 2 버전에서 다음과 같은 코드가 동작합니다.

```python
>>> print "hello, python2!"
hello, python2!
```

그렇지만 Python 3 에서는 에러를 뿜어내면서 동작하지 않습니다. Python 3 버전은 다음과 같이 동작합니다.

```python
>>> print "hello, python3!"
  File "<stdin>", line 1
    print "hello, python3!"
                          ^
SyntaxError: Missing parentheses in call to 'print'
>>> 
>>> print ("hello, python3!")
hello, python3!
```

이러한 문법 차이는 PyJail 문제를 풀 때 고려해야 할 사항 중 하나입니다. 그렇다면 실제 CTF에서는 어떻게 문제가 나왔는지 확인 해 보도록 합시다. 문제를 풀 때에는 제 의식의 흐름을 좇아갑니다.

## [2017 Christmas CTF] Everyday is Christmas

지금 소개해드릴 문제는 2017년 Christmas CTF에 출제된 문제로, 제가 처음으로 CTF 기간 내에 푼 문제이기도 합니다. 왜 크리스마스에 동아리방에서 해당 문제를 붙들고 있었는지는 넘어가시고... 문제는 CTF 종료된 후 다른 서버에 이식해서 풀었습니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic01.PNG)

해당 프로그램의 컨트롤 플로우는 위 그림과 같습니다. 간단하게 다시 설명하면, `chdir` 명령어를 통해 `working directory`를 `/tmp` 디렉토리로 바꿔버린 후, 사용자 입력을 `get_clean_path`라는 함수를 통해 검열합니다. 검열을 마치면 사용자 입력을 `path`로 받아들여서 해당 디렉토리 안의 파일을 모조리 출력해주고, 다시 `get_clean_path`를 이용해서 사용자 입력을 검열한 뒤 입력과 일치하는 파일 이름을 찾아서 이 파일의 내용을 출력해주는, 아주 간단한 루틴을 가지고 있습니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic02.PNG)

`get_clean_path`는 위와 같군요. 필터되는 내용을 보니 etc, proc, dev, passwd... 등등 시스템과 관련있는 디렉토리/파일들에 대해서 검열을 수행합니다. 그런데 왜 `home`과 `~`까지 검열을 수행할까요? `home directory`쪽에 무언가 있을 것 같다는 예감을 들게 합니다. 우리의 `working directory`도 `home directory`와는 한참 떨어지는 `/tmp`에 있구요. `./`이 안되니까 이전 디렉토리로 가려고 `../` 도 사용하기 어렵습니다.

그런데 반환할 때, `path.expanduser`라는 함수를 이용해서 반환을 해 줍니다. 그럼 이 함수가 무엇을 하는 녀석인지 `documentation`을 찾아보는 게 좋겠군요.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic03.PNG)

Unix에서 `path.expanduser`는 `~`또는 `~user`를 `user`의 `home directory`로 바꿔주는 역할을 한다고 되어있네요. 예를 들면 `~KRater`와 같이 인자가 들어오면, 이를 `/home/KRater`와 같이 바꾸어주는 역할입니다. 이를 활용하면 필터링을 우회할 수 있지 않을 것 같나요? 먼저 `~`을 통해서 서버의 홈 디렉토리를 열어보려고 시도했습니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic04.PNG)

그러자 파이썬의 오류메시지가 친절하게도 `/home/user`가 디렉토리라서 읽을 수 없다고 알려줍니다. 이렇게 오류메시지를 주는 PyJail 문제는 매우 친절한 편입니다. 이를 이용해서 많은 정보를 얻을 수 있기 때문이죠. 그리고 아까 이전 디렉토리로 가려고 `../`을 사용하기 어렵다는 말을 했었는데, `/..`는 막혀있지 않는다는 점에 주목해서 다음과 같이 할 수 있습니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic05.PNG)

딱 봐도 수상해보이는 플래그 유저가 있습니다. (원래라면 플래그 유저랑 user 둘 만 있어야 하는데 다른 서버에다가 임시로 문제를 설치해서 유저가 좀 많습니다. 그래도 문제 푸는데는 지장이 없어요.) 그렇담 이제 해당 유저의 홈 디렉토리를 읽고, 그 안에 있는 파일도 읽을 수 있게됩니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic06.PNG)

어때요, 참 쉽죠?

## [2018 noxCTF] Python for fun

다음 문제는 2018 noxCTF에 출제되었던 Python for fun 이라는 문제입니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic07.PNG)

문제가 위와 같이 나와있는데, 요약해보면 파이썬 공부용 웹사이트를 만들었다는 것과, python3를 활용한다는 등의 정보를 알 수 있습니다. 해당 웹사이트로 들어가 볼까요?

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic08.PNG)

크게 세가지의 메뉴가 있습니다.

1. fix the code : 코드에서 틀린 부분을 찾습니다. (객관식)
2. match signature to body : 빠진 부분을 넣습니다. (**주관식**)
3. what the code returns : 코드가 무슨 값을 반환하는지 추측합니다. (객관식)

유저 입력을 객관식으로 넣는것도 가능하겠지마는, 원하는 값을 마음껏 넣을 수 있는건 주관식이겠죠? 주관식 문제를 들어가보았습니다.

![]({{ site.baseurl }}/images/KRater/2018-09-28-Hello_Pyjail/pic09.PNG)

함수 인자에 빠진 부분을 채워넣는 문제군요. 인터프리터가 서버에서 직접 실행되서 다음과 같은 코드를 실행하는 것 같습니다.

```python
def func(a, b) :
    c = a + b
    return c

print (func(10, 12) == 22)

True
```

이 함수가 실행된 결과가 True, False 등으로 나타나지만 별 의미없습니다. 여기서 중요한건 인터프리터가 서버에서 직접 우리가 입력한 값을 포함한 코드를 실행한다는 것이죠. 다음과 같은 코드는 어떨까요?

```python
def func(x, y, a=20, b=2, d=1, e=1, f=1) :
	c = a + b
	return c

print (func(10, 12) == 22)

True
```

역시 True로 나타나는 구문입니다. 어, 뭔가 이상하죠? 저는 x와 y에 10, 12를 넣었는데 True가 나타납니다. 이는 a와 b를 미리 결정된 값을 갖는 인자로 선언했기 때문입니다. 파이썬은 인자에 default값을 결정할 수 있습니다. 이는 호출 시 인자의 값을 결정하지 않더라도, 이를 자동으로 default값을 따르도록 만들어주는 아주 편리한 기능입니다. 그렇다면 이렇게 생각해 봅시다. 호출할 때 함수의 리턴값을 default로 만들 순 없을까요?

```python
def func(x, y, a=20, b=2, d=print("hello, world!")) :
    c = a + b
    return c

print (func(10, 12) == 22)

hello, world!
True
```

아주 잘 작동하는군요. print 함수의 결과로 출력된 문자열을 볼 수 있습니다. 그렇다면 그냥 우리가 원하는 함수를 실행시킬 수 있게 된 것이죠.

```python
def fun(a, b, x=exec('import os'), y=os.system('ls -la')):
    c = a + b
    return c

print(fun(10, 12) == 22)

total 8
drwxr-xr-x 1 root root  18 Sep 13 06:46 .
drwxr-xr-x 1 root root  17 Sep  8 23:27 ..
-r--r--r-- 1 root root  23 Sep  5 16:45 FLAG
dr--r--r-- 1 root root 168 Sep 13 06:46 learn_python.py
-r--r--r-- 1 root root 563 Sep  5 16:45 manage.py
dr--r--r-- 1 root root  74 Sep 13 06:46 python_ctf_thing
dr--r--r-- 1 root root 136 Sep 13 06:46 templates
True
```

`exec` 함수로 `import os`구문을 실행시킨 뒤 `os.system` 함수를 실행하면 쉘 명령을 실행시킬 수 있습니다. FLAG를 찾을 수 있네요. cat 명령어를 실행시켜서 플래그를 읽으면 됩니다.



## 마치며

지금까지 기본적인 PyJail 문제들을 풀어보았습니다. PyJail 분야가 CTF에서 가장 쉬운 문제들을 담당하고 있다고 말은 했지만, 생각보다 훨씬 점수 배율이 높은 문제들도 있습니다. 파이썬에 익숙한 사람일수록 이 문제들의 체감 난이도는 급락하기 때문에 CTF에서 점수를 내기에 좋을것입니다. 그럼, 모두 즐거운 공부 되시기 바랍니다 ^0^