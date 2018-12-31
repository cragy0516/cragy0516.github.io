---
layout: post
title: ! 'Hook iOS applications using Frida'
excerpt_separator: <!--more-->
tags:
  - iOS
---

Frida를 이용한 iOS 어플리케이션 후킹에 대해 알아보도록 한다.

<!--more-->

# Frida

Frida는 Windows, macOS, GNU/Linux, iOS, Android, QNX 플랫폼 위의 다양한 어플리케이션에 자바스크립트 코드 조각들을 삽입할 수 있도록 도와주는 도구이다. Frida API를 활용해서 다양한 기능들을 사용할 수 있도록 만들어져 있다. 주로 어떠한 어플리케이션의 API를 추적(trace)하거나, 이미 개발된 어플리케이션에 로깅 함수를 삽입(inject)하여 번거로운 빌드 과정 없이도 테스트를 진행할 수 있도록 하는 데 Frida를 사용할 수 있다.

Firda의 core 부분은 [C로 작성](https://github.com/frida/frida-core)되었다. Frida는 동작 시 [구글 V8 엔진](https://developers.google.com/v8/)을 특정 프로세스에 삽입하여, JavaScript가 해당 프로세스 메모리의 모든 부분에 엑세스하고, 함수를 후킹하며 임의의 함수를 호출하도록 실행될 수 있게 돕는다.

본 포스팅에서는 Frida를 이용하여 iOS 어플리케이션에 다양한 조작을 하는 과정을 보일것이다. 본 예제는 Electra로 탈옥된 iOS 11.2.6 버전의 iPhone 8 기기에서 적용하였다.

# Frida 설치 및 명령어 사용

macOS에서 파이썬을 통해 Frida API를 활용하기 위해서, pip를 이용해서 Frida를 설치한다.

`$ pip install frida-tools`

만약 pip에서 권한 오류가 발생 할 경우에는 다음 경로의 프로그램을 실행한다.

`$ /Applications/Python\ 3.7/Install\ Certificates.command`

iPhone에도 Frida를 설치 해 놓아야 한다. `Cydia`를 설치한 뒤, `https://build.frida.re`를 소스에 추가하면 Frida 패키지를 내려받을 수 있으며, 이를 설치하면 된다.

Frida를 실행하는 방법은 크게 두 가지가 있는데, USB를 이용하는 방법과 remote로 이용하는 방법이다.

USB로 Frida를 이용하기 위해서는, 먼저 USB 케이블로 휴대폰과 Mac을 연결한다. 그 뒤 `-U` 옵션을 주어서 frida 도구를 실행하면 된다.

```bash
cragy0516-MacBookPro:~ cragy0516$ frida-ps -U
 PID  Name
----  --------------------------------------------------------
1776  Cydia
 189  Mail
​```
```

별도의 설정 없이도 가능하므로 간단하지만, USB 연결이 불가능할 경우가 존재한다. 그럴때는 휴대폰에서 특정 포트로 서버를 열어두고 remote로 접근하면 된다.

```bash
iPhone:~ root# frida-server --listen=192.168.0.39:27042
2018-12-07 19:06:43.230 frida-server[2025:304354] Frida: Unable to check in with launchd: are we running standalone?
```

오류메시지가 뜨는것을 확인했으나 작동에는 이상이 없다.

```bash
cragy0516-MacBookPro:~ cragy0516$ frida-ps --host=192.168.0.39:27042
 PID  Name
----  --------------------------------------------------------
1776  Cydia
 189  Mail
```

Mac으로 돌아와서 `--host=` 옵션으로 접속할 수 있다.

## Frida API 활용

Frida API를 활용하면 함수를 후킹하여 반환 값을 변조하거나, 심지어는 자신이 정의한 새로운 함수를 호출하도록 만들 수 있다.

### 시작하기

본 예제에서는 디바이스를 USB로 연결한 상태임을 가정한다. `device_manager`를 호출한 뒤, devices중 마지막을 가져온다. 마지막으로 연결된 USB 디바이스를 사용 할 것이기 때문이다. 그 뒤 PACKAGE NAME을 기반으로 새로운 프로세스를 `spawn`한다. 그 뒤 `do_hook()` 함수를 스크립트로 등록하고, logging을 위한 `on_message` 함수를 등록한다. 이에 대한 예시 코드는 아래와 같다.

```python
import frida
import sys

def on_message(message, data):
    try:
        if message:
            print("[log] {0}".format(message["payload"]))
    except Exception as e:
        print(message)
        print(e)

def do_hook():
    hook = """
    if(ObjC.available) {
        send("Ready to frida!");
    } else {
        console.log("Objective-C Runtime is not available!");
    }
    """

    return hook

if __name__ == '__main__' :
    PACKAGE_NAME = "com.xxx.yyy"
    try :
        print ("[log] devices info : {}".format(frida.get_device_manager().enumerate_devices()))
        device = frida.get_device_manager().enumerate_devices()[-1]

        pid = device.spawn([PACKAGE_NAME])
        print ("[log] {} is starting. (pid : {})".format(PACKAGE_NAME, pid))

        session = device.attach(pid)
        device.resume(pid)

        script = session.create_script(do_hook())
        script.on('message', on_message)
        script.load()
        sys.stdin.read()
    except KeyboardInterrupt:
        sys.exit(0)
```

이에 대한 출력 결과는 아래와 같다.

```bash
$ python3 my_hook.py
[log] devices info : [Device(id="local", name="Local System", type='local'), Device(id="tcp", name="Local TCP", type='remote'), Device(id="device_id_is_here", name="iPhone", type='usb')]
[log] com.xxx.yyy is starting. (pid : 5675)
[log] Ready to frida!
```

### 함수 반환값 변조하기

함수를 후킹하기 위해서는 **클래스 이름**과 **메소드 이름**을 알아야 한다. `IDA`, `hopper`등의 바이너리 분석 도구를 활용하면 클래스 이름을 알아낼 수 있는데, 해당 클래스가 가지고 있는 모든 메소드를 알아내는 간단한 방법은 다음과 같다. `ObjC.classes.ColorChecker`는 ColorChecker 클래스를 나타내며, `$ownMethods`는 부모 클래스에는 없으며 자신만이 가지고 있는 고유한 메소드의 집합을 나타낸다.

```python
def do_hook():
    hook = """
    if(ObjC.available) {
        var class_checker = ObjC.classes.ColorChecker;
        var methods_checker = class_checker.$ownMethods;
        methods_checker.forEach(function(m) {
            send(m);
        });
    } else {
        console.log("Objective-C Runtime is not available!");
    }
    """

    return hook
```

수정된 `do_hook()`메소드의 출력 결과는 다음과 같다.

```bash
$ python3 my_hook_2.py
[log] com.xxx.yyy is starting. (pid : 5690)
[log] + initialize
[log] - isSuperUser
[log] - isColorCorrect:
[log] - isColorChanged
[log] - isDirectory
[log] - isApplication
```

이제 함수를 후킹하여 함수의 반환 값을 조작 해 보도록 하자. `Interceptor`를 활용하면 함수를 후킹하여 반환 값을 변조하거나, 또는 새로운 사용자 정의 함수로 바꿔치기하거나, 새로운 함수를 등록할수도 있다. `do_hook()` 메소드를 아래와 같이 수정한다.

```python
def do_hook():
    hook = """
    if(ObjC.available) {
        var class_checker = ObjC.classes.ColorChecker;
        var methods_checker = class_checker.$ownMethods;
        var isApplication = class_checker['- isApplication'];

        Interceptor.attach(isApplication.implementation, {
            onEnter: function(args) {
                var target = new ObjC.Object(args[0]);
                var sel = ObjC.selectorAsString(args[1]);
                send("Target class : " + target.$className);
                send("Target selector : " + sel);
            },
            onLeave: function(retVal) {
                send("Old return : " + retVal);
                retVal.replace("0");
                send("New return : " + retVal);
            }
        });
    } else {
        console.log("Objective-C Runtime is not available!");
    }
    """

    return hook
```

`isApplication`은 사용자의 휴대포에 특정한 어플리케이션이 설치되어 있는지를 감지하여, 설치되어 있다면 True, 설치되어 있지 않다면 False를 반환하는 메소드이다. 현재 리스트 안의 어플리케이션을 감지하여 True (1)을 반환하였는데, `Interceptor`에 의해 반환 값이 False (0)으로 변경되었다. 이에 대한 출력은 아래와 같다.

```bash
$ python3 my_hook_3.py
[log] com.xxx.yyy is starting. (pid : 5702)
[log] Target class : ColorChecker
[log] Target selector : isApplication
[log] Old return : 0x1
[log] New return : 0x0
```

### 사용자 정의 함수 호출

Frida API를 활용하면 또한 실제 함수를 호출하는 대신, 사용자 정의 함수를 호출하도록 할 수도 있다. 가장 간단한 방법으로 malloc 함수를 후킹하여 할당하는 바이트와 할당 위치를 구하는 방법을 알아보자.

```python
def do_hook():
    hook = """
    if(ObjC.available) {
        var mallocPtr = Module.findExportByName('libsystem_c.dylib', 'malloc');
        var malloc = new NativeFunction(mallocPtr, 'pointer', ['int']);
        Interceptor.replace(mallocPtr, new NativeCallback(function (size) {
        var location = malloc(size)
        send("Allocated " + size + " bytes in " + location);
        return location;
    }, 'pointer', ['int']));

    } else {
        console.log("Objective-C Runtime is not available!");
    }
    """

    return hook
```

`mallocPtr` 변수는 `Module.findExportByName`을 활용하여 실제 라이브러리에서 이름을 가지고 malloc 함수의 포인터를 얻어내는 과정이다. 그 뒤 `malloc` 변수는 `NativeFunction`으로 선언하는데, `mallocPtr` 위치의 함수를 `pointer` 반환 값에 `int`인자 하나를 갖는다. 그 뒤에 `replace`를 이용하여 `mallocPtr`을 새로운 `NativeCallback`으로 교체한다. 기존의 `malloc`에 들어간 `size`를 가져오고, 실제 할당되는 위치를 로그로 남길 수 있도록 구현하였다. 이에 대한 출력 결과는 아래와 같다.

```bash
$ python3 my_hook_4.py
[log] Allocated 29 bytes in 0x1d4032680
[log] Allocated 16 bytes in 0x1d4016220
[log] Allocated 177 bytes in 0x1d41744c0
...
[log] Allocated 16 bytes in 0x1d4016260
[log] Allocated 53 bytes in 0x1d4274d00
[log] Allocated 16 bytes in 0x1d4016370
[log] Allocated 69 bytes in 0x1d408d070
[log] Allocated 16 bytes in 0x1d4016230
...
[log] Allocated 105 bytes in 0x1d40d4ac0
[log] Allocated 16 bytes in 0x1d40164b0
[log] Allocated 105 bytes in 0x1d40d4ba0
```

이상으로 Firda가 제공하는 Javascript API를 활용하여 다양한 조작을 해 보았다. 더 많은 정보는 공식 홈페이지에서 제공하는 [API Reference](https://www.frida.re/docs/javascript-api/)를 참조하면 된다.
