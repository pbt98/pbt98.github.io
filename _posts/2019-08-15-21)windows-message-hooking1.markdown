---
layout: post
title:  "21. Windows Message Hooking 1st"
date:   2019-08-15 20:10:00 +0900
categories: Reversing
comments: true
---
## 21.1) 훅  
* 정보를 엿보거나 가로채는 행위를 훅이라고 한다.  

## 21.2) 메시지 훅  
* Windows 운영체제는 GUI를 제공하고, 이는 **Event Driven** 방식으로 작동한다.  
* 이벤트란 키보드/마우스를 이용하여 매뉴 선택, 버튼 선택, 마우스 이동, 창 크기 변경 등등의 작업을 말한다.  
* 이런 이벤트가 발생 시, 운영체제는 **미리 정의된 메시지**를 해당 응용 프로그램으로 통보한다.  
* 키보드를 입력할 때 OS -> Application 으로 가는 메시지를 중간에서 엿보는 것이 **메시지 훅**이다.  
* 훅 기능은 Windows 운영체제에서 제공하는 기본 기능이다.(자체로는 불법이 아니다.)  

```  
일반적인 경우의 Windows 메시지 흐름  
1. 키보드 입력 이벤트 발생  
2. [OS messaage queue]에 WM_KEYDOWN 메시지 추가  
3. OS는 어느 응용 프로그램에서 이벤트가 발생했는지 파악해서 dequeue후 해당 프로그램의 [application message queue]에 추가  
4. 응용 프로그램은 자신의 [application message queue]를 모니터링하고 있다가 WM_KEYDOWN 메시지가 추가된 걸 확인하고 해당 event handler를 호출  
참고: 같은 키보드 메시지 훅이라도 여러 개를 동시에 설치 가능함. 이런 훅은 설치대로 호출되므로 이를 훅 체인이라고 일컫는다.  
```  

## 21.3) SetWindowsHookEx()  
* 메시지 훅은 SetWindowsHookEx() API를 사용해서 간단히 구현 가능하다.  

```  
HHOOK SetWindowsHookEx (
    int idHook,        // hook type  
    HOOKPROC lpfn,     // hook procedure  
    HINSTANCE hMod,    // 위 hook procedure가 속해 있는 DLL 핸들  
    DWORD dwThwreadId  // hook을 걸고 싶은 thread의 ID  
)  
```  

* hook procedure는 운영체제가 호출해주는 콜백 함수다.  
* 메시지 훅을 걸 때 hook procedure는 DLL 내부에 존재해야 하며, 그 DLL의 인스턴스 핸들[^1.]이 바로 hMod이다.  
* SetWindowsHookEx()를 이용해서 훅을 설치해 놓으면, 어떤 프로세스에서 해당 메시지가 발생했을 때 운영체제가 해당 DLL 파일을 해당 프로세스에 강제로 인젝션하고 등록된 hook procedure를 호출한다.  

## 21.4) 키보드 메시지 후킹 실습  
* KeyHook.dll 파일은 훅 프로시저(KeyboardProc)가 존재하는 DLL 파일이다.  
* HookMain.exe는 KeyHook.dll을 최초로 로딩하여 키보드 훅을 **설치하는 프로그램**이다.  
* HookMain.exe에서 KeyHook.dll 파일을 로딩한 후 SetWindowsHookEx()를 이용하여 키보드 훅(KeyboaardProc)을 설치한다.  
* 만약 다른 프로세스(explorer.exe, iexplore.exe, notepad.exe 등)에서 키 입력 이벤트가 발생하면 OS에서 **해당 프로세스의 메모리 공간에 KeyHook.dll을 강제로 로딩하고 KeyboardProc() 함수가 호출된다.**  

# 21.4.1) 실습 예제 HookMain.exe  

![keyhook](/asset/keyhook.JPG)  

* HookMain.exe를 실행 후, 키 입력을 했더니 keyhook.dll이 인젝션 되면서 키 입력이 먹지 않는 것을 확인할 수 있었다.  

![injected](/asset/injected.JPG)  

* KeyHOok.dll이 인젝션된 프로세스들을 볼 수 있다.  

# 21.4.2) 소스코드 분석  
* HookStart() 함수가 호출되면 SetWindowsHookEx()에 의해서 키보드 훅 체인에 KeyboardProc이 추가된다.  
* 소스코드는 길어지므로 일부러 올리지 않았다.  

```  
MSDN의 KeyboardProc 함수의 정의  
LRESULT CALLBACK KeyboardProc(
    int code,  // HC_ACTION(0), HC_NOREMOVE(3)  
    WPARAM wParam,  // virtual-key code  
    LPARAM lParam  // extra information  
);  
  
위 파라미터 중에서 wParam은 사용자가 누른 키보드의 virtual key code를 의미한다. 키보드의 하드웨어적인 의미로써 영문 'A' 'a' 그리고 한글 'ㅁ'은 모두 같은 virtual key code를 가진다. 또한 lParam 값에는 각 비트별로 다양한 의미를 가진다. ToAscii() API 함수를 사용하면 실제 눌린 키보드의 ASCII 값을 구할 수 있다.  
```  

* 키보드 훅이 설치된 상황에서 어떤 프로세스에서 키 입력 이벤트가 발생하면 OS는 해당 프로세스에게 **강제로** KeyHook.dll을 인젝션한다. 이제 KeyHook.dll을 로딩한 프로세스에서 키보드 이벤트가 발생하면 KeyHook.KeyboardProc()이 먼저 호출된다.  
* KeyboardProc() 함수의 내용에는 키보드 입력이 발생했을 때 현재 프로세스 이름과 "notepad.exe" 문자열과 비교하여 만약 같다면 1을 리턴해서 KeyboardProc() 함수를 종료시킨다. -> 이것이 메시지를 가로채서 없애버리는 것이다. -> 결국 키보드 메시지는 notepad.exe의 [application message queue]에 전달되지 않는다.  
* 그 외의 경우에는 return CallNextHookEx(g_hHook, nCode, wParam, lParam); 명령을 실행하면 **메시지는 다른 응용 프로그램 혹은 훅 체인의 또 다른 훅 함수로 전달**된다.  
[^1.]: 인스턴스는 어떤 대상이나 단위의 구체적인 실체를 말하고, 핸들은 운영체제에 의해 생성된 리소스나 오브젝트를 제어하기 위한 32비트 정수값이다.  