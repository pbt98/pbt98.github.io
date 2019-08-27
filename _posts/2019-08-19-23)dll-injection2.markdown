---
layout: post
title:  "23. DLL injection 2nd"
date:   2019-08-19 20:40:00 +0900
categories: Reversing
comments: true
---
# 23.4.2) 예제 소스코드 분석  
&nbsp;&nbsp;**Myhack.cpp**  
* DllMain()을 보면 DLL이 로딩(DLL_PROCESS_ATTACH)될 때 디버그 문자열("myhack.dll Injection!!!")을 출력하고 스레드(ThreadProc)를 실행한다.  
* ThreadProc()의 내용은 urlmon!URLDownloadToFile() API를 호출하여 네이버 사이트의 index.html 파일의 다운받는 것이다.  
* 프로세스에 DLL 인젝션이 발생한다. -> 해당 DLL의 DllMain() 함수가 호출된다 -> URLDownloadToFile() API가 호출된다.  

&nbsp;&nbsp;**InjectDll.cpp**  
* main() 함수의 주요기능은 **프로그램의 파라미터를 체크한 후** InjectDll() 함수를 호출하는 것이다.  
* InjectDll() 함수가 바로 DLL 인젝션을 해주는 핵심 함수이다.  
* InjectDll() 함수는 대상 프로세스(notepad.exe)로 하여금 스스로 **LoadLibrary("myhack.dll") API를 호출**하도록 명령하는 기능을 가지고 있다.  

1. 대상 프로세스 핸들 구하기  
**hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID)**  
* OpenProcess() API를 이용하여 PROCESS_ALL_ACCESS 권한의 NOTEPAD.EXE **프로세스 핸들**을 구한다(이때 실행 파라미터로 넘어온 dwPID 값을 사용함.)  

2. 대상 프로세스 메모리에 인젝션할 DLL 경로를 써주기  
**pRemoteBuf = VirtualAllocEx(hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);**  
* 상대방 프로세스(notepad.exe)에게 로딩할 DLL 파일의 경로(문자열)를 알려주는 기능이다.  
* 아무 메모리 공간에나 쓸 수 없으므로 VirtualAllocEx() API를 이용하여 상대방 프로세스(notepad.exe) 메모리 공간에 버퍼를 할당한다.  
* 버퍼 크기는 DLL 파일 경로 문자열 길이(NULL 포함)로 지정하면 된다.  
**WriteProcessMemory(hProcess, pRemoteBuf, (LPVOID)szDllName, dwBufSize, NULL);**  
* 앞에서 할당받은 **버퍼 주소(pRemoteBuf)**에 WriteProcessMemory() API를 이용하여 DLL 경로 문자열("C:\\work\\dummy.dll")을 써준다.  
* WriteProcessMemory() API 역시 hProcess 핸들이 가리키는 상대방 프로세스의 메모리 공간에 쓰는 것이다.  

3. LoadLibraryW() API 주소 구하기  
**hMod = GetModuleHandle("kernel32.dll");**  
**pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(hMod, "LoadLibraryW");**  
* LoadLibrary() API를 호출하기 위해선 그 주소가 필요하다.  
* notepad.exe 프로세스에 로딩된 kernel32.dll의 LoadLibraryW() API의 시작 주소를 알아내야함. -> InjectDll.exe 프로세스에 로딩된 kernel32.dll의 LoadLibraryW() API의 시작 주소를 얻어내고 있음. **notepad.exe에 로딩된 kernel32.dll의 주소와 InjectDll.exe에 로딩된 kernel32.dll의 주소가 동일하다면 위 코드는 문제 없다.** -> Windows 운영체제에서는 프로세스마다 같은 주소에 로딩된다.  


4. 대상 프로세스에 원격 스레드(Remote Thread)를 실행함.  
**hThread = CreateRemoteThread(hProcess, NULL, 0, pThreadProc, pRemoteBuf, 0, NULL);**  
**pThreadProc = notepad.exe 프로세스 메모리 내의 LoadLibraryW() 주소**  
**pRemoteBuf = notepad.exe 프로세스 메모리 내의 "c:\\work\\myhack.dll" 문자열 주소**  
* LoadLibraryW() API를 호출하도록 하는 API는 기본 제공하지 않는다. -> 따라서 CreateRemoteThread()를 이용한다.  

```  
HANDLE WINAPI CreateRemoteThread (
    __in    HANDLE                      hProcess, //프로세스 핸들  
    __in    LPSECURITY                  lpThreadAttributes,  
    __in    SIZE_T                      dwStackSize,  
    __in    LPTHREAD_START_ROUTINE      lpStartAddress, //스레드 함수 주소  
    __in    LPVOID                      lpParameter, //스레드 파라미터 주소  
    __in    DWORD                       lpThreadId  
);  
```  

* hProcess 파라미터가 스레드를 실행할 프로세스의 핸들이다.  
* lpStartAddress, lpParameter 파라미터는 각각 스레드 함수 주소와 스레드 파라미터 주소인데, **이 주소들은 대상 프로세스의 가상 메모리 공간의 주소여야 한다.**  
* CreateRemoteThread()를 호출해서 lpStartAddress에 **LoadLibrary()** 주소를 입력하고 lpParameter에 인젝션을 원하는 'DLL의 경로 문자열 주소'를 주면 된다. -> **CreateRemoteThread()는 실제로 LoadLibraryW()를 호출하도록 만드는 역할이다.**  

# 23.4.3) 디버깅 방법  
