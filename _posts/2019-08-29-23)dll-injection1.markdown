---
layout: post
title:  "23. DLL injection 1st"
date:   2019-08-19 16:13:00 +0900
categories: Reversing
comments: true
---
*DLL 인젝션 기법을 통하여 API 후킹, 프로그램 기능 개선 및 버그 패치 등의 작업을 수행할 수 있다.*  

## 23.1) DLL 인젝션  
* DLL 인젝션이란? 실행 중인 다른 프로세스에 특정 DLL 파일을 강제로 삽입하는 행위이다. -> 다른 프로세스에게 LoadLibrary() API를 스스로 호출하도록 명령하여 사용자가 원하는 DLL을 로딩하는 것이다.  
* 일반적인 DLL 로딩과 뭐가 다르죠? **로딩 대상이 되는 프로세스가 내 자신이냐 아니면 다른 프로세스냐** 하는 것이다.  
* 어떤 프로세스가 원래 로딩하지 않는 DLL을 DllMain()에 코드가 추가되면 자연스럽게 해당 코드가 실행되면서 **프로세스 메모리에 대한 (정당한) 접근 권한**을 가진다. 따라서 **사용자가 원하는 어떤 일이라도 수행 가능하다.**  

```  
프로세스에 DLL이 로딩되면 자동으로 DllMain() 함수가 실행된다. 따라서 DllMain()에 사용자가 원하는 코드를 추가하면 DLL이 로딩될 때 자연스럽게 해당 코드가 실행된다. 이 특성을 이용하면 기존 애플리케이션의 버그를 수정하거나 새로운 기능을 추가 가능하다.  

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD dwReason, LPVOID lpvReserved)  
{  
    switch(dwReason)  
    {  
        case DLL_PROCESS_ATTACH:  
        //원하는 코드 추가  
        break;  
          
        case DLL_THREAD_ATTACH:  
        break;  
          
        case DLL_THREAD_DETACH:  
        break;  
          
        case DLL_PROCESS_DETACH:  
        break;  
    }  

    return TRUE;  
}  
```  

## 23.2) DLL 인젝션 활용 예  
* LoadLibrary() API를 이용해서 어떤 DLL을 로딩하면 해당 DLL의 DllMain() 함수가 실행된다. DLL 인젝션의 동작 원리는 외부에서 하여금 **LoadLibrary() API를 호출하도록 만드는 것이기 때문에** 강제 삽입된 DLL의 DllMain() 함수가 실행된다.  
* 삽입된 DLL은 해당 프로세스의 메모리에 대한 접근 권한을 갖기 때문에 사용자가 원하는 다양한 일(버그 패치, 기능 추가, 기타)을 수행 가능하다.  

# 23.2.1) 기능 개선 및 버그 패치  
* 어떤 프로그램의 소스코드가 없거나 수정이 여의치 않을 때 이용해서 Plugin 개념으로 전혀 새로운 기능을 추가하거나 문제가 있는 코드와 데이터를 수정할 수 있다.  
# 23.2.2) 메시지 후킹  
* Windows OS에서 기본으로 제공하는 메시지 후킹 기능도 일종의 DLL 인젝션 기법이다.  
* 차이점은 등록된 후킹 DLL을 OS에서 직접 인젝션 시켜준다는 것이다.  

# 23.2.3) API 후킹  
* 후킹 함수를 DLL 형태로 만든 후 후킹을 원하는 프로세스에 간단히 인젝션 하는것만으로 완성된다.  

# 23.2.4) 기타 응용 프로그램  
* 주로 PC 사용자들을 감시하고 관리하기 위한 애플리케이션들에서 사용된다. -> 유해 사이트 차단이나 사용 시간 제한 등등 -> 이 파트는 잘 배워둬서 동생들 컴퓨터에 써먹어야겠다.  

# 23.2.5) 악성 코드  
* 정상적인 프로세스(winlogon.exe, services.exe, svchost.exe, explorer.exe 등)에 숨어들어가 백도어 포트를 열어서 외부에서 접속을 시도하거나, 키로깅 기능으로 사용자의 개인 정보를 훔치는 등의 악의적인 행동을 한다.  

## 23.3) DLL 인젝션 구현 방법  
1. 원격 스레드 생성(CreateRemoteThread() API)  
2. 레지스트리 이용(AppInit_DLLs 값)  
3. 메시지 후킹(SetWindowsHookEx() API)  

## 23.4) CreateRemoteThread()  
# 23.4.1) 실습 예제 myhack.dll  
* 목표:  
1. notepad.exe 프로세스에 myhack.dll 인젝션  
2. 인젝션된 myhack.dll은 인터넷에 접속하여 http://www.naver.com/index.html 파일을 다운받도록 함.  

&nbsp;&nbsp;**notepad.exe 프로그램 실행**  
* notepad.exe 실행 후 Process Explorer를 실행하여 PID를 알아낸다.  
![pid](/asset/pid.JPG)  

* PID는 1212로 확인되었다.  

&nbsp;&nbsp;**DebugView 실행**  
* DebugView는 시스템에서 실행되는 프로세스들이 출력하는 **모든 디버그 문자열을 가로채서 보여주는** 툴이다.  
* 실습 예제 DLL 파일은 notepad.exe 프로세스에 성공적으로 인젝션되면 디버그 문자열을 출력하는데 DebugView로 그 순간 확인 가능하다.  

&nbsp;&nbsp;**myhack.dll 인젝션**  
* injectDll.exe 프로그램은 DLL 인젝션 유틸리티이다. 커맨드 창을 띄워서 실행 파라미터를 입력하면 실행된다.  
![inject](/asset/inject.JPG)  

&nbsp;&nbsp;**DLL 인젝션 확인**  
* 위 사진에서 DebugView에 문자열이 출력되어있다. 1212번을 notepad.exe의 PID로 알고 있으므로 여기에 inject 한 것이다.  
* Process Explorer에서도 myhack.dll이 인젝션된 것을 확인가능하다. Process Explorer의 View 매뉴에서 Show Lower Pane 항목과 Lower Pane Views - DLLs 항목을 선택하고 notepad.exe 프로세스를 선택하면 myhack.dll을 확인할 수 있다.  
![success](/asset/success.JPG)  

&nbsp;&nbsp;**결과 확인**  
* index.html이 정상적으로 다운로드 된 것을 확인할 수 있었다.  
![naver](/asset/naver.JPG)  