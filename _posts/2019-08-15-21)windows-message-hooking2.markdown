---
layout: post
title:  "21. Windows Message Hooking 2nd"
date:   2019-08-15 21:42:00 +0900
categories: Reversing
comments: true
---
## 21.5) 디버깅 실습  
# 21.5.1) HookMain.exe 디버깅  

&nbsp;&nbsp; **핵심 코드 찾기**  
* 한 줄씩 트레이싱  
* 관련 API 검색  
* 관련 문자열 검색   

한 줄씩 트레이신 하는 방법은 프로그램이 잘 동작하지 않거나 예측이 어려운 경우에 사용하므로, "press 'q' to quit!"을 이용해서 찾아보면, (코드 전/후에 관심있는 코드가 있을 것이다.)  
![hookmain](/asset/hookmain.JPG)  
Search for - All referenced text strings를 이용해서 찾아왔다. 찾아오고 보니 여긴 HookMain의 Main 함수 부분이었다.  

&nbsp;&nbsp; **main() 함수 디버깅**  
* 401006 주소에서 LoadLibrary("KeyHook.dll")을 호출한다.  
* 40104B 주소의 CALL EBX 명령에 의해서 KeyHook.HookStart() 함수가 호출된다.  
![hookstart](/asset/hookstart.JPG)  

* 위 사진은 HookStart 함수이다. 100010EF 주소에서 **CALL SetWindowsHookExW()** 명령어를 볼 수 있다.  
* 100010E8, 100010ED 주소의 PUSH 명령어를 보면 SetWindowHookExW() API의 1, 2번째 파라미터를 알 수 있다.  
* API의 첫 번째 파라미터(idHook)의 값은 **WH_KEYBOARD(2)**이다.  
* API의 두 번째 파라미터(lpfn)의 값은 10001020이며, **이 값이 바로 훅 프로시저의 주소**이다.  

# 21.5.2) Notepad.exe 프로세스 내의 KeyHook.dll 디버깅  
* Notepad.exe 실행중에 Attach를 하는 방법이 있고, 실행 중간에 HookMain.exe를 실행시켜서 멈추게 하는 방법이 있다 하는데 notepad.exe의 키보드 입력이 살아있다... 3시간을 고민했지만 잘 모르겠어서 일단은 패스한다.  

## 궁금했던 점  
1. 콜백(CALLBACK) 함수란?  
&nbsp;&nbsp;특정 이벤트가 발생하면 호출해달라고 지정해 놓은 함수다. ex) Windows 프로시저(WindProc)가 대표적이다.  
2. SetWindowsHookEx() API 호출이 왜 KeyHook.dll 내부에 있는건가?  
&nbsp;&nbsp;SetWindowsHookEx() API는 지정한 훅 프로시저를 훅 체인에 등로한다. -> DLL 내부 혹은 외부에서 호출해도 상관 없다.  
