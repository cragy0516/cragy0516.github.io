---
layout: post
title: Development Grade Server with Docker and Flask
excerpt_separator: <!--more-->
comments: true
tags:
  - Development
---

`ARGOS Freshman Festival`이 한 달 남짓 남음에 따라, 채점 서버를 만들 필요가 생겼습니다. 기본적으로 `AFF`대회의 속성을 설명하자면 채점 언어는 C언어에만 한정하며, 소수의 인원들로 구성된 대회에서 알고리즘 문제를 해결하고 이에 따른 점수를 획득해 최종적으로 합산된 점수를 기준으로 순위를 평가하는 방식입니다.

<!--more-->

## 구상

### 채점 방법

알고리즘 문제를 평가하는 방식은 여러가지가 있을 수 있습니다. 그 중에서도 대표적으로 생각한 방식은 **1. 소스 코드를 구문 분석하여 정답인지 판단하는 방식**과 **2. 소스 코드를 컴파일하여 정답인지 판단하는 방식**을 꼽을 수 있었습니다. 전자의 경우에는 구문 분석이란게 그리 수월하지는 않아 거의 컴파일러를 하나 더 만드는 수준이라고 생각해서 진작에 포기했고, 빠르게 후자의 방식을 택했습니다.

소스 코드를 컴파일하면 유저가 만든 어떠한 프로그램이 탄생할 것이고, 서버에서 이를 실행하여 정답인지를 판별하면 채점이 가능할 것입니다. 그렇다면 어떤식으로 이 채점을 할 것인가에 대해서도 생각해 보아야 합니다. 기본적으로 여러 개의 `테스트 케이스`를 만들어 놓은 뒤, 유저가 만든 프로그램에 이를 대입해서 서로 같은 지 확인하여 모든 테스트 케이스를 통과하면 정답이라고 판단하면 됩니다.

유저가 만드는 프로그램은 두 가지 경우가 있을 수 있습니다. 첫 번째는 입력 없이 단순히 출력만 가지고 정답인지를 판단할 수 있습니다. 이러한 경우에는 다음과 같은 문제들이 있겠죠?

> 다음과 같이 출력하는 프로그램을 만드시오.
>
> hello, world!

두 번째는 입력을 가지고 출력을 만드는 프로그램이 있을 수 있습니다. 이 경우에는 다음과 같은 문제를 낼 수 있습니다.

> [피보나치 수열](https://ko.wikipedia.org/wiki/%ED%94%BC%EB%B3%B4%EB%82%98%EC%B9%98_%EC%88%98)에 대해서, n을 입력했을 때 F(n)을 출력하는 프로그램을 만드시오.
>
> 9
>
> 34

두 가지 케이스를 생각해서, 테스트 케이스를 어떻게 만들 지 설계했습니다. 이에 대한 내용은 후술하겠습니다.

### 보안 위협

그렇지만 다시 한가지 문제에 직면했습니다. 기본적으로 컴파일한 코드를 그대로 실행하는 것은 아주 큰 보안 위협에 직면합니다. 프로그램을 컴파일하는 권한에 따라 다르겠지만, 기본적으로 채점 서버에 `채점 전용 유저`를 만들어놓으면 서버에 미치는 영향을 크게 제한할 수 있습니다. 하지만 그럼에도 불구하고 **모든 유저가 같은 채점 전용 유저를 공유한다는 문제점** 때문에, 다른 유저의 소스, 프로그램, 또는 결과 정보 파일까지도 접근할 수 있는 악용 가능성이 생깁니다. 만약 다음과 같은 `C언어` 코드가 있다고 생각해봅시다.

```c
#include <stdlib.h>
int main ()
{
    system("rm -rf /");
    return 0;
}
```

해당 코드를 실행하면 어떻게 될까요? 기본적으로 `채점 전용 유저`가 가지고 있는 모든 파일을 삭제하려고 들 것입니다. 이를 막기 위해서는 `Sandbox` 방법을 사용해야 합니다. 유저마다 고유한 채점 환경을 갖도록 서버를 구성하여, 다른 유저의 채점 환경에 간섭하는 것을 막아야 합니다. 이를 구현하는 방법은 생각보다 여러가지가 있습니다. 하지만 전통적인 방법보다는, 비교적 최신 기술이자 굉장히 유용한 `Docker`를 이용하여 채점 환경을 가상화하도록 하겠습니다.

### 프론트 엔드

이제 `front-end`에 대해서 생각해봅시다. 여러가지 방법으로 유저들에게 소스 코드를 받을 수 있겠지만, 가장 접근성 높고 편리한 인터페이스를 가진 것은 역시 웹 브라우저겠죠. 따라서 **동적인 웹 페이지**를 작성해야 하는데, 규모가 작기 때문에 이에 대해서는 가볍게 `flask`와 파이썬 `SQLite`을 활용하도록 하겠습니다.

기본적인 구상이 끝났으니, 이제 각각 구성요소에 대해 어떻게 구현하였는지 자세히 설명하겠습니다.

## 안전한 채점 환경

상술했듯 저는 `Docker`를 이용해서 채점 환경을 가상화했습니다. 이 경우 모든 `Docker Container`는 분리된 환경에서 동작합니다. 모든 코드는 `Container` 안에서 컴파일되고, 실행됩니다. 컴파일을 포함한 **모든 채점 프로세스를 가상 환경에서 진행하는 것**입니다. 채점 환경이 분리되었습니다. 그렇다면 어떤 기준으로 채점 환경을 분리해야 할까요?

채점 환경을 분리하는 기준을 유저마다로 나누었다고 생각 해 봅시다. 그러면 유저가 위의 악의적인 코드를 발송했을 때, 자신의 채점 환경이 완전히 망가져 새로운 채점을 받을 수 없게 될 것입니다. 물론 이를 악의적인 행동에 대한 처벌이라고 생각할수도 있겠죠. 하지만 자신의 채점 환경에서 뭔가 악의적인 환경으로 조작한 뒤, 이를 이용해 채점 서버가 잘못 채점하도록 속일수도 있습니다. 개발단계에서 이를 일일히 신경쓰고 모든 보안 위협을 방어하는 것은 소모적이죠. 심지어 큰 대회도 아니고, 작은 내부 대회에서 말입니다.

결국 채점 환경을 유저로 구분하여 전적으로 유저에게 환경에 대한 소유권을 주는 것은 많은 문제점을 가지고 있습니다. 그렇다면 어떤 기준을 세워야 할까요? 저는 코드를 제출할 때 마다 새로운 채점 환경을 제공하도록 하였습니다. 유저는 환경에 악의적인 영향을 미칠 수 있지만, 해당 영향은 영구적이지 않습니다. 즉, 코드를 채점하면 해당 환경은 버려집니다. 유저가 환경에 미칠 수 있는 영향력을 제한하지는 않았지만, 짧고 일시적으로만 영향을 미칠 수 있는것이죠.
성능에 대해서는 깊게 분석해보지 않았지만, `Docker` 자체가 컨테이너를 생성하고 삭제하는것에 대해 큰 부하를 일으키지 않을것이라고 판단했습니다. 물론 대규모 인원을 운용하는 서버에서는 충분한 테스트가 필요하지만, 현 상황에서는 크게 의미가 없다고 생각합니다.

여기서 채점이 끝난 후, 버려진 채점 환경에서 채점 서버가 회수해야 하는 것이 단 하나 있습니다. 바로 채점에 대한 결과죠. 이는 간단하게 `Docker run` 명령어에 대한 결과를 `Redirection` 하는것으로 충분합니다. 결과적으로 유저가 채점 서버에 보낼 수 있는 유일한 것은 단순히 프로그램을 실행한 결과입니다. 그것이 `rm -rf * && ls -la`과 같은 **악의적인 명령어를 실행한 결과**일지라도 채점 서버는 상관하지 않습니다. 단순히 해당 명령어를 실행함으로서 나온 파일 목록을 받아서 틀린것으로 판단하면 그만이니까요. 다른 유저들에게도 전혀 영향을 미치지 않습니다. 물론 새로운 컨테이너를 생성해주는 정도의 부하는 있겠지만, 이로 인하여 얻는 이득은 비교할 수 없을정도로 어마어마하게 큽니다.

그렇다면 채점 프로세스는 어떻게 이루어져 있을까요? 확인 해 봅시다.

### 채점 프로세스

유저가 웹 브라우저에서 코드를 작성해서 보냅니다. 그러면 채점 서버는 서비스를 위한 쉘 스크립트를 실행합니다. 해당 쉘 스크립트는 아주 간단한 내용을 담고 있습니다. 한번 확인 해 볼까요?

```bash
#!/bin/bash

if [ "$1" == "" -o "$2" == "" ];then
	echo "Usage: $0 <id>_<username> <id>"
	exit
fi

cd "${0%/*}"

docker run -it --rm \
-v "$PWD"/src/$1.c:/usr/src/myapp/$1.c \
-v "$PWD"/cases/check.sh:/usr/src/myapp/check.sh \
-v "$PWD"/cases/case_$2.txt:/usr/src/myapp/case_tmp.txt \
-v "$PWD"/cases/programs/case_$2:/usr/src/myapp/programs/solution \
-w /usr/src/myapp gcc:4.9 \
timeout 5s \
/bin/bash -c "gcc -std=c99 -w -o $1 $1.c && ./check.sh $1" \
> results/result_$1.txt
```

여러가지가 있지만, 인자에 대한 검증과 Working Directory를 설정해 주는 부분을 제외하면 실행하는 명령은 단 하나입니다. 바로 `docker run` 명령이죠. 이에 대한 출력 결과는 `results` 디렉토리 안의 특정 파일로 `Redirection` 해 주고 있습니다. 웹 브라우저에서는 인자로 `id_username`과 문제의 `id`를 인자로 제공해줍니다. 다음은 웹 브라우저를 생성해주는 `flask` 관련 소스 코드 일부입니다.

```python
...
            subprocess.call(filepath + "compile.sh " + filename + " " + str(id), shell=True)
            resf = open(filepath + "results/result_" + filename + ".txt", 'r')
            result = resf.readline().strip()
            if (result == "correct"):
                print(str(id) + " has been solved by " + g.user['username'])
...
```

`compile.sh` 파일이 바로 위의 쉘 스크립트입니다. 이 파일에 `filename (id_username format)` 및 문제의 `id`를 인자로 제공해 주는 것을 볼 수 있습니다. 또한 위에서 `Redirection`한 파일을 참조해서 해당 결과에 `correct`라고 적혀있으면 정답이라고 간주합니다. 그렇다면 해당 파일은 어떻게 값이 쓰여지는 것 일까요?

해답은 `docker run` 명령어에서 찾아볼 수 있습니다. `docker run` 명령어는 총 4개의 `-v` 옵션을 사용합니다. 한번 해당 명령을 확인 해 봅시다.

```bash
-v "$PWD"/src/$1.c:/usr/src/myapp/$1.c \
-v "$PWD"/cases/check.sh:/usr/src/myapp/check.sh \
-v "$PWD"/cases/case_$2.txt:/usr/src/myapp/case_tmp.txt \
-v "$PWD"/cases/programs/case_$2:/usr/src/myapp/programs/solution \
```

`-v` 옵션은 호스트의 파일 시스템을 컨테이너에 연결합니다. `host file:guest file`의 형식으로 작성하면 됩니다. 위에서는 총 4개의 파일을 연결하고 있는데요, 한 줄씩 확인 해 봅시다.

```bash
-v "$PWD"/src/$1.c:/usr/src/myapp/$1.c \
```

첫 번째 줄은 첫 번째 인자로 받은 `id_username` 를 이용해서 소스 코드를 맵핑합니다. 유저가 작성한 소스 코드가 /src 디렉토리의 `id_username.c` 형식으로 저장되는데, 이를 컴파일용 디렉토리에 맵핑해서 컴파일 가능하게 만들어 줍니다.

```bash
-v "$PWD"/cases/check.sh:/usr/src/myapp/check.sh \
```

두 번째 줄은 `check.sh` 파일에 대한 맵핑입니다. 갑자기 나온 이 쉘 스크립트 파일은 무엇일까요? 바로 이 파일이 결과에 correct를 작성한 주범입니다. 이 파일은 조금 뒤에 다시 나옵니다. 다음 줄을 봅시다.

```bash
-v "$PWD"/cases/case_$2.txt:/usr/src/myapp/case_tmp.txt \
```

세 번째 줄은 test case를 저장한 파일에 대한 맵핑입니다. $2 인자로 받은 `challenge id`를 기반으로 테스트 케이스 txt 파일을 찾고, 이를 `case_tmp.txt`라는 파일에 맵핑합니다. 테스트 케이스는 입력에 한정되는데, 이를 이용해서 출력 결과를 찾는 부분은 후술하겠습니다.

```bash
-v "$PWD"/cases/programs/case_$2:/usr/src/myapp/programs/solution \
```

마지막으로 맵핑하는 파일은 `solution` 파일입니다. 해당 파일은 정답 소스코드를 통해 컴파일된 실행 파일입니다. 호스트 파일 시스템에서는 `case_<challenge id>`의 파일 이름으로 찾을 수 있습니다.

```bash
-w /usr/src/myapp gcc:4.9 \
timeout 5s \
/bin/bash -c "gcc -std=c99 -w -o $1 $1.c && ./check.sh $1" \
> results/result_$1.txt
```

그럼 이제 이렇게 맵핑한 파일을 어떻게 사용하는지 알아봅시다. 일단 `gcc:4.9 Image`를 이용해서 컨테이너를 생성하는데요, 무한루프 등을 예방하기 위해서 `timeout` 5초를 가지고, gcc 명령어로 첫 번째에 맵핑한 소스 코드를 컴파일하는것을 볼 수 있습니다. 그 뒤에는 두 번째에 맵핑한 `check.sh` 파일을 `username_id` 인자와 함께 실행시키는군요. 나머지 맵핑한 파일은 보이지가 않는데, 바로 이 `check.sh` 파일에서 사용하기 때문입니다. 그럼 슬슬 `check.sh` 파일을 보도록 합시다.

```bash
#!/bin/bash

cd "${0%/*}"

if [ ! -e ./$1 ] || [ ! -e ./programs/solution ] || [ ! -e case_tmp.txt ];then
	echo "error"
	exit
fi

read case_tmp < case_tmp.txt
if [ "${case_tmp}" == "" ]; then
	res_out=`./$1`
	sol_out=`./programs/solution`
	if [ "${res_out}" != "${sol_out}" ];then
		# it means result is different.
		# otherwise, it will echo 'error' to file.
		echo "fail : must ${sol_out} but give ${res_out}"
		exit
	fi
fi

while read case_in
do
	res_out=`echo ${case_in} | ./$1`
	sol_out=`echo ${case_in} | ./programs/solution`
	if [ "${res_out}" != "${sol_out}" ];then
		# it means result is different.
		# otherwise, it will echo 'error' to file.
		echo "fail : must ${sol_out} but give ${res_out}"
		exit
	fi
done < case_tmp.txt

echo "correct"
```

조금 기네요. 하지만 확인 해보면 간단합니다. 인자로 하나를 받는데, 이는 위에서 받은 `username_id` 입니다. 만약 컴파일된 파일이나 위에서 세번째에 맵핑한 `case_tmp.txt`, 네번째에 맵핑한 `solution` 파일 중 하나라도 없으면 error를 출력하고 종료합니다. 여기서 이 파일들이 쓰이는군요. 일단 case_tmp.txt를 읽어서 비어있으면 이 문제는 입력이 없는 문제임을 알 수 있습니다. 위에서 본 `"hello, world!"` 출력과 같은 문제죠. 그렇다면 단순히 solution을 실행하여 해당 출력 결과와 유저의 코드를 컴파일한 실행 파일의 출력 결과를 비교합니다.

만약 입력이 있는 문제일 경우에는 어떨까요? `case_tmp.txt`에는 입력에 대한 테스트 케이스가 저장되어 있다고 했습니다. 따라서 해당 케이스를 한줄씩 읽으면서 유저 파일을 실행하고, 이를 solution 파일의 출력 결과와 비교합니다. 모든 케이스를 통과하면 맞는 답이라고 생각하고, 이를 정답으로 인정합니다.

최종적으로 정답이라면 `correct`를 출력합니다. 바로 이것이 `Redirection` 되어 `result.txt` 파일에 저장됩니다. 만약 컴파일 에러가 발생했거나 결과값이 다르다면 어떻게 될까요? 컴파일 에러가 그대로 반환되거나, `fail : must ${sol_out} but give ${res_out}` 등 적절한 처리 구문으로, correct 외의 값이 반환됩니다. 물론, 해당 반환값은 (correct를 포함하여) 유저가 볼 수 없으므로 유저가 system 함수의 인자로 `cat case_tmp.txt` 등을 넣어 테스트 케이스를 읽으려고 시도해도 읽을 수 없습니다.

위와 같이 채점 프로세스를 안전하게 제작하였습니다. 채점 결과를 실제로 반영하는 부분은 **프론트 엔드**에서 계속됩니다.

## 프론트 엔드

### Flask

동적인 웹 페이지를 제작하기 위해서는 많은 기술이 사용되는데, 역시 상술했듯 소규모 서비스이기 때문에 다루기 쉬운 `flask`를 사용하였습니다. 물론 이번이 `flask`를 처음 다뤄보는거라서, [공식 튜토리얼 문서](http://flask.pocoo.org/docs/1.0/)를 이용해서 제작한 블로그에 덧붙이는 식으로 진행했습니다. 이를 조금만 응용하면 되었기에 제작 과정이 간단했습니다.

```python
def must_admin(view):
    @functools.wraps(view)
    def wrapped_view(**kwargs):
        if g.user['id'] is not 1:
            return redirect(url_for('index'))
        return view(**kwargs)
    return wrapped_view
```

물론 블로그를 그대로 사용하는것은 무리였습니다. 기본적으로 블로그는 개인 공간이기 때문이죠. 많은 사람들이 공유하되, 채점자와 수험생이 구분되는 정도의 구분 등은 필요했습니다. 이를 해결하기 위해서 `user id`가 1인 사람을 `admin`으로 생각하도록 하였습니다. 따로 DB 컬럼을 이용해 구분하지 않고서도 충분히 직관적이게 구현한 것 이지요. 물론 서비스 규모가 커지면 이를 대대적으로 손볼 필요는 있겠지만요.

### SQLite

프론트 엔드 부분은 아직도 손보는 중이기에 보여드릴것이 많지는 않지만, 주요 기능은 이미 구현이 되었습니다. 예를 들어 상술한 채점 결과를 DB에 반영하는 부분을 보여드리겠습니다.

```python
            if (result == "correct"):
                print(str(id) + " has been solved by " + g.user['username'])
                db = get_db();
                alreadySolved = db.execute(
                    'SELECT s.id'
                    ' FROM solved s'
                    ' WHERE s.chall_id = ? AND s.solver_id = ?',
                    (id, g.user['id'])
                ).fetchone()
                # print (alreadySolved)
                if (alreadySolved is None) :
                    # First solved!
                    db.execute(
                        'INSERT INTO solved (solver_id, chall_id)'
                        ' VALUES (?, ?)',
                        (g.user['id'], id,)
                    )
                    chall = db.execute(
                        'SELECT score'
                        ' FROM challenge c'
                        ' WHERE c.id = ?',
                        (id,)
                    ).fetchone()
                    newScore = chall['score'] + g.user['score']

                    db.execute(
                        'UPDATE user SET score = ?'
                        ' WHERE id = ?',
                        (newScore, g.user['id'])
                    )
                    print("Gained " + str(chall['score']) + " points. (" + str(newScore) + " points now)")
                    db.commit()
                else :
                    # Already Solved!
                    # do nothing...?
                    print("Already solved")
            resf.close()
            return redirect(url_for('blog.index'))
```

코드가 조금 깁니다만, 직관적입니다. 먼저 DB는 크게 3개의 테이블을 이용합니다. `user`, `challenge`, 그리고 `solved` 테이블이죠. `solved` 테이블은 어떤 유저가 문제를 풀 때마다 업데이트됩니다. 나중에 유저가 문제를 풀면, 해당 테이블을 참조해서 이미 풀었는지 확인하는 식으로 중복 채점을 검사합니다. 사실 이 부분의 구현에서 오랫동안 고민을 했었는데, 괜찮은 솔루션을 찾아서 다행입니다.

### 마치며

이상 AFF 서버를 제작하는 데 시작하면서 생각했던 구상과, 실제로 구현하면서 사용한 기술들에 대해서 정리해 보았습니다. 궁금한 사항은 언제나 메일로 보내주시면 답변해드리도록 하겠습니다. 긴 글 읽어주셔서 감사합니다!