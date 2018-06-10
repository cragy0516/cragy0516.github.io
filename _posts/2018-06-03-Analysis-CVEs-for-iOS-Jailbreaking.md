---
layout: post
title: Analysis CVEs for iOS Jailbreaking
excerpt_separator: <!--more-->
tags:
  - Analysis
  - iOS
---

안녕하세요, `MadHat`팀에서 `pwnable`을 공부하고 있는 `KRater`입니다. 저는 현재 [iOS 탈옥](https://en.wikipedia.org/wiki/IOS_jailbreaking)에 대한 공부를 진행하고 있는데, 이를 위해서 분석한 CVE 두 가지와, 분석에 필요한 선행지식을 소개 해 드리도록 하겠습니다.

<!--more-->

## iOS 11 탈옥

iOS는 안드로이드와는 다르게 각종 강력한 보안 매커니즘을 제공합니다. 이에 대한 자세한 정보는 애플이 제공하는 [보안 백서](https://www.apple.com/business/docs/iOS_Security_Guide.pdf)에서 확인하실 수 있습니다. 이 때문에 단순히 루트 권한을 획득하는 취약점으로는 탈옥을 수행할 수 없고, 다른 보안 장치를 우회하는 방법이 필요합니다. 그러나 루트 권한을 획득하는 것은 탈옥의 가장 중요한 동기이기도 하죠.

iOS 11 탈옥은 *현재 (2018-06-03)*를 기준으로, 굉장히 최근에 일어난 일입니다. 현재 iOS 버전은 iOS 11.4.1 버전 까지 공개가 되었는데요, 이에 반해서 최신 탈옥 도구는  iOS 11.0 - 11.1.2 버전에 사용되는 `Electra`라는 도구입니다. `Electra`는 일련의 취약점 체인을 이용하여 탈옥을 수행하는데, 이 때 **루트 권한을 획득**하기 위해서 사용되는 주요 아이디어 2개가 바로 구글의 프로젝트 제로 팀의 `Ian Beer`라는 사람이 공개한 CVE에서 발췌했다고 합니다.

서론은 이쯤하면 됬고, 지금부터는 `Ian Beer`가 공개한 취약점에 대해 설명을 드리도록 하겠습니다.

|    CVE 이름    |                           CVE 개요                           |
| :------------: | :----------------------------------------------------------: |
| CVE-2017-13861 | IOSurfaceRootUserClient가 MIG 소유 규칙을 따르지 않아 발생하는 더블 프리 버그. |
| CVE-2017-13865 | 유저 공간 포인터의 유출을 검사하는 커널 API의 버그로 커널 메모리 유출. |

해당 취약점들은 iOS 11.2 버전에 패치되었습니다. 그래서 `Electra` 탈옥 도구는 iOS 11.1.2 버전까지만 지원을 하는 것이구요. 하지만 해당 취약점을 바로 분석하고자 하니 ~~팔자에도 없던 iOS라서~~ 너무 어렵더군요. 그래서 다음 취약점들에 대한 분석을 진행하게 되었습니다.

|   CVE 이름    |                           CVE 개요                           |
| :-----------: | :----------------------------------------------------------: |
| CVE-2016-7612 | ipc_port_t 필드의 참조 카운트 오버 플로우로 인한 커널 UAF 발생. |
| CVE-2016-7633 | 유저 공간 MIG 코드에서 vm_deallocate의 중복으로 인한 UAF 발생. |

위 취약점들은 엄연히 `Electra`에 사용된 취약점 목록은 아닙니다. `Electra`에서 사용한 `CVE-2017-13861` 취약점을 설명한 `Ian Beer`의 기술 문서에서 언급한 두 개의 취약점입니다. 이 역시 `Ian Beer`가 발견한 취약점들인데, 해당 취약점들에 대한 이해가 선행되어야 한다고 판단해서 해당 취약점들을 분석하게 되었습니다.



## 선행지식

취약점을 분석하기에 앞서, 우리는 iOS 커널의 구성 요소들에 대한 선행지식이 필요합니다. 해당 취약점을 분석하는 데 필요한 구성 요소를 설명해 드리도록 하겠습니다.

### Mach Port

`Mach`는 주로 분산 및 병렬 연산을 지원하기 위해 카네기 멜론 대학교에서 개발한 운영 체제 커널인데, 이에 대한 파생물로는 macOS가 있습니다. 따라서 macOS의 형제격이라고 할 수 있는 iOS도 Mach 커널의 일부를 담습하고 있습니다.

![]({{ site.baseurl }}/images/KRater/2018-06-03-Analysis-CVEs-for-iOS-Jailbreaking/pic01.PNG) 

`Mach Port`란, 해당 커널에서 사용하는 `프로세스 간 커뮤니케이션(Inter Process Communication, IPC)` 채널의 말단 부분을 뜻합니다. 위 그림에서 Task a는 Task b에 `message`를 보내고 있는데, 이는 커널 영역에 있는 `Message Queue`를 이용합니다.

각 Task는 자신만의 `port`를 소유하고 있는것을 보실 수 있습니다. 해당 Port는 Task a가 보내는 `message`가 정당한 발신 권한을 가지고 있는지를 확인한 뒤, `Message Queue`를 이용해서 Task b에 전달합니다. `Message Queue`에 의해서 처리된 `message`는 Task b의 `port`로 이동합니다. `port`에서는 해당 `message`를 확인한 뒤 Task b가 적절한 수신 권한을 가지고 있는 지 확인하고, 확인이 끝나면 이를 수신합니다.

이쯤에서 눈치채신 분들도 있겠지만, UNIX에서 사용하는 파이프 기능과 본질적으로 매우 유사하다는것을 확인하실 수 있습니다. 그렇다면 Mach Port가 전송하는 `message`에 대해서 알아볼까요?

### Message

`Message`는 프로세스간에 통신에서 전달되는 객체입니다. Message는 `Header`와 `Body`로 나눌 수 있는데, `Header`에는 도착지 포트에 대한 Task의 권한, 수신하는 Task가 응답해야 하는 경우에, 응답할 수 있도록 전달되는 송신 권한, Body에 OOL (Out-of-Line) 데이터가 있는지에 대한 여부 등이 포함됩니다.

`Body`는 포함되는 데이터의 종류에 따라서 `In-Line Message Body`와 `Out-of-Line Message Body`로 나눌 수 있습니다. `In-Line Message Body`는 데이터가 body에 모두 포함이 되어 있습니다. 따라서 보낼 수 있는 메시지의 길이는 한정적이죠. 반면에 `Out-of-Line Message Body`는 데이터가 모두 포함이 되어있는 대신, 해당 데이터의 주소가 포함되어 있습니다. 따라서 보낼 수 있는 크기에 제한이 없습니다. 그럼에도 불구하고 OOL 데이터가 실제로 존재하고 유효한 주소 공간안에 있는지를 커널이 검사해야 한다는 번거로움은 존재합니다.

### 참조-카운트 방식

`참조-카운트` 방식은 `쓰레기 수집(Garbage Collection)` 정책의 일부입니다. 오브젝트는 자신이 `참조된 (referenced)` 횟수를 기록합니다. 만약 참조된 횟수가 0이라면, 이 오브젝트는 더 이상 사용되지 않는다고 판단하고 운영체제가 이를 회수합니다.

이 방식은 여러가지 문제를 가지고 있습니다만, 여전히 직관적이고 간단하기 때문에 널리 쓰입니다.

### 기타

해당 CVE에서는 `IOKit`과 `MIG`에 대한 선행지식을 요구합니다. `IOKit`은 하드웨어 드라이버 개발을 위해 애플에서 지원하는 프레임워크입니다. `MIG`는 커널 소유의 `message port`가 [사용하는 도구](https://googleprojectzero.blogspot.com/2016/10/)인데, 주로 `코드 직렬화(Code Serialization)`에 활용한다고 합니다.

![]({{ site.baseurl }}/images/KRater/2018-06-03-Analysis-CVEs-for-iOS-Jailbreaking/pic02.PNG) 

`코드 직렬화`란, 일반적인 `데이터 구조`나 `오브젝트 상태`를 다른 포맷으로 변환하는것을 말합니다. 이 때 중요한것은 변환된 포맷은 또다시 원래의 데이터로 재구성할 수 있어야 한다는 것입니다. 이렇게 직렬화를 수행하는 이유는 데이터 송수신에 용이하도록 하는 이유입니다. 즉, MIG는 커널 소유의 `message port`가 메시지를 수월하게 송수신 하기 위해 사용하는 도구라고 보시면 될 것 같습니다.

## CVE 분석

그렇다면 이제부터 위에서 설명한 4개의 CVE중, 2016년에 공개된 CVE 2개에 대해서 분석한 내용을 설명드리도록 하겠습니다. 설명드릴 내용은 `Ian Beer`가 공개한 [기술](https://bugs.chromium.org/p/project-zero/issues/detail?id=926) [문서](https://bugs.chromium.org/p/project-zero/issues/detail?id=1372)를 참조하여 번역하고, 애매한 부분은 제가 공부한 내용을 첨가하였습니다.

### CVE-2016-7612

> ipc_port_t 필드의 참조 카운트 오버 플로우로 인한 커널 UAF 발생.

CVE-2016-7612의 주요 골자는 위와 같습니다. ipc_port_t는 커널에서 mach port를 정의한 구조체입니다. 해당 구조체는 `참조-카운트` 방식을 사용하므로, 구조체의 필드에는 `참조-카운트 변수(io_references)`가 존재합니다. `ip_reference`와 `ip_release` 함수는 이 변수를 원자적으로 증가시키거나 감소시킵니다. 또한 `참조-카운트 변수`는 32비트 변수인데, 값이 `오버플로우`를 검사하지 않습니다. 그렇다면 이 참조-카운트 변수가 어떻게 오버플로우 되어서 커널 UAF가 발생하는지 알아보도록 합시다.

Port 권한은 자신에 대한 참조 카운트를 가지고 있습니다. 어떠한 메시지가 권한을 참조하면 해당 참조 카운트가 1 증가할것이고, 메시지가 소멸하면 참조 카운트는 1 감소할것입니다. 그런데 `ipc_kobject_server` 함수에는 다음과 같은 [코드](https://opensource.apple.com/source/xnu/xnu-124.7/osfmk/kern/ipc_kobject.c)가 있습니다.

```javascript
	if ((kr == KERN_SUCCESS) || (kr == MIG_NO_REPLY)) {
		/*
		 *	The server function is responsible for the contents
		 *	of the message.  The reply port right is moved
		 *	to the reply message, and we have deallocated
		 *	the destination port right, so we just need
		 *	to free the kmsg.
		 */
		ipc_kmsg_free(request);

	} else {
		/*
		 *	The message contents of the request are intact.
		 *	Destroy everthing except the reply port right,
		 *	which is needed in the reply message.
		 */
		request->ikm_header.msgh_local_port = MACH_PORT_NULL;
		ipc_kmsg_destroy(request);
	}
```

현재 상태가 KERN_SUCCESS 또는 MIG_NO_REPLY라면, `ipc_kmsg_free` 함수가 호출됩니다. 그 밖의 경우에 대해서는 `ipc_kmsg_destroy`함수가 호출됩니다. 그런데 `ipc_kmsg_free` 함수가 단순히 메시지 헤더만을 파괴하여 메시지 내용에 포트 권한이 남아있다면 메시지 참조 횟수는 줄어들지 않습니다. 반면에, `ipc_kmsg_destroy` 함수는 메시지에 있는 모든 포트 권한에 대한 참조 카운터를 줄입니다.

![]({{ site.baseurl }}/images/KRater/2018-06-03-Analysis-CVEs-for-iOS-Jailbreaking/pic03.PNG)

만약 MIG 메소드가 KERN_SUCCESS를 반환하지만, 실제로는 포트 권한을 파괴하지 않는다면 해당 포트 권한에 대한 참조는 유지될것이고, 이를 악용하는 데 사용할 수 있을것입니다. 다음은 내부 함수인 `internal_io_service_add_notification`을 확인해 봅시다.

```
static kern_return_t internal_io_service_add_notification(
    mach_port_t master_port,
	io_name_t notification_type,
	io_buf_ptr_t matching,
	mach_msg_type_number_t matchingCnt,
	mach_port_t wake_port,
	void * reference,
	vm_size_t referenceSize,
	bool client64,
	kern_return_t *result,
	io_object_t *notification )
{
      ...
      if( master_port != master_device_port)
        return( kIOReturnNotPrivileged);
      
      do {
        err = kIOReturnNoResources;
        
        if( !(sym = OSSymbol::withCString( notification_type )))
          err = kIOReturnNoResources;
        
        if (matching_size)
        {
          dict = OSDynamicCast(OSDictionary, OSUnserializeXML(matching, matching_size));
        }
        else
        {
          dict = OSDynamicCast(OSDictionary, OSUnserializeXML(matching));
        }
        
        if (!dict) {
          err = kIOReturnBadArgument;
          continue;
        }
      ...
      } while( false );
      
      return( err );
```

해당 함수는 잘못된 커널 포트, 유효하지 않은 직렬화된 데이터 등의 케이스에 대한 에러를 가지고 있습니다. 이 경우에는 이 내부 함수가 인수로 전달된 `wake_port`에 대한 소유권을 취하지 않을것입니다. 그런데, MIG는 `internal_io_service_add_notification_ool` 함수의 반환값을 취합니다. 해당 함수를 보면

```
static kern_return_t internal_io_service_add_notification_ool(
  ...
    kr = vm_map_copyout( kernel_map, &map_data, (vm_map_copy_t) matching );
    data = CAST_DOWN(vm_offset_t, map_data);
    
    if( KERN_SUCCESS == kr) {
      // must return success after vm_map_copyout() succeeds
      // and mig will copy out objects on success
      *notification = 0;
      *result = internal_io_service_add_notification( master_port, notification_type,
                                                     (char *) data, matchingCnt, wake_port, reference, referenceSize, client64, notification );
      vm_deallocate( kernel_map, data, matchingCnt );
    }
    return( kr );
  }
```

위와 같은 코드를 확인할 수 있습니다. 우리가 유효한 `OOL Memory Descriptor`를 제공한다면 `internal_io_service_add_notification_ool` 함수가 `KERN_SUCCESS`를 반환할 것이고, 이를 이용하면 MIG는 `ipc_kmsg_free`함수를 호출하려고 할 것입니다. `io_service_add_notification_ool` 함수에 유효한 `OOL Memory Descriptor`를 제공하되, 안에 있는 값이 직렬화되지 않은 데이터라서 `OSUnserializeXML` 에러를 반환하도록 하면 해당 메모리는 해제되지만, `wake_port`에 대한 `참조-카운트`는 유지될 것 입니다.
![]({{ site.baseurl }}/images/KRater/2018-06-03-Analysis-CVEs-for-iOS-Jailbreaking/pic04.PNG)

우리가 이러한 악의적인 요청을 `0xFFFFFFFF`회 반복하면, `ipc_port_t`의 `io_references` 필드는 오버플로우되어 `0`이 됩니다. 그러면 운영체제가 쓸모없어진 메모리라고 생각하고, 이를 회수할 것 입니다. 하지만 여전히 우리는 포트 테이블에 해당 포트에 대한 포인터를 가지고 있으며, 이로 인하여 `UAF 버그`가 발생하게 됩니다.

### CVE-2016-7633

> 유저 공간 MIG 코드에서 vm_deallocate의 중복으로 인한 UAF 버그 발생.

CVE-2016-7633 취약점을 요약하면 위와 같습니다. 이전 취약점에 비해서 내용이 짧은 편입니다. `mach_msg_server` 또는 `mach_msg_server_once`는 `원격 프로시저 호출(RPC)` 서버를 구현하는 데 사용됩니다. 이들은 유저 공간 MIG 서비스에서 사용되는데, MIG 핸들러 메소드가 에러코드를 반환하면 메시지의 자원에 대한 소유권이 없다고 생각하고 `mach_msg_server`와 `mach_msg_server_once`  함수는 `mach_msg_destroy` 함수를 수행하고자 합니다.

기본적으로 위의 `RPC 서버` 모델에서 MIG 클라이언트가 복사 유형이 `MACH_MSG_PHYSICAL_COPY`로 설정 된 `OOL 메모리`를 전송하는 데, 수신자는 이를 `deallocate 0`으로 받아들입니다. 만약 보낸 사람이 `MACH_MSG_VIRTUAL_COPY`로 복사 유형을 설정하면, 수신자는 이를 `deallocate 1`로 받아들입니다. 그로 인해서 `mach_msg_destroy` 함수에서 인자로 받은 `OOL Descriptor`가 필드로 가지고 있는 `deallocate flag`가 세팅되면, `vm_deallocate`가 수행됩니다.

그런데, `vm_deallocate`가 오류 코드를 반환하면 `mach_msg_server` 혹은 `mach_msg_server_once` 함수는 이를 다시 처리해 주어야 합니다. 그래서 다시 한번 `vm_deallocate`를 호출하죠. 이 부분에서 문제가 발생합니다.

![]({{ site.baseurl }}/images/KRater/2018-06-03-Analysis-CVEs-for-iOS-Jailbreaking/pic05.PNG)
만약 프로그램의 실행 흐름이 하나라면 위 방법은 문제가 없습니다. 그러나 두 번째 쓰레드가 두개의 `vm_deallocate` 사이에 끼어들어 메모리를 재할당 받는다면, 두 번째 `vm_deallocate`는 이 공간을 다시한번 해제 해 버립니다. 이렇게 되면 두 번째 쓰레드는 해당 영역에 대한 유효한 포인터를 가지고 있음에도 불구하고 할당 해제된 영역을 참조하고 있는 것이죠. 이로서 `UAF 버그`가 발생합니다.

## 마치며

지금까지 iOS에 관련된 취약점 2개를 알아보았습니다. 최종적으로 `Electra`에 사용된 취약점 체인을 분석하기에 앞서 워밍업이라고 생각하면 될 것 같네요. 앞으로는 실제로 `Electra`에 사용된 취약점들에 대한 분석을 진행 해 보도록 하겠습니다. 공부하면서 적은 내용이라 틀린점이 있을 수 있으니, 해당 부분들은 메일로 보내주시면 감사히 검토하도록 하겠습니다. 긴 글 읽어주셔서 감사합니다 :D
