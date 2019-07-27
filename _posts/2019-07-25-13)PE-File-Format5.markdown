---
layout: post
title:  "13. PE File-Format 5th"
date:   2019-07-24 10:07:00 +0900
categories: Reversing
comments: true
---
# 13.5) IAT
* IAT: Import Address Table의 약자로, 프로그램이 어떤 라이브러리에서 어떤 함수를 사용하고 있는지를 기술한 테이블이다.  
## 13.5.1) DLL
* DLL 도입 전: 16비트 DOS 시절에는 라이브러리를 프로그램에 포함시켜서 같이 컴파일했었다. 
* 문제점: Windonws OS로 넘어오면서, 멀티테스킹을 지원하게 되었는데 전과 같은 방식으로 프로그래밍을 할 경우 같은 함수를 쓰는 프로그램이 동시에 메모리에 올라가면서 메모리 낭비가 심해졌다.  
* DLL 도입:  
&nbsp;&nbsp;1. 프로그램에 라이브러리를 포함시키지 말고 별도의 파일(DLL)에 구성하여 필요할 때마다 불러 쓸 것.  
&nbsp;&nbsp;2. 일단 한 번 로딩된 DLL의 코드, 리소스는 Memory Mapping 기술로 여러 Process에서 공유해 쓰자.  
* DLL 로딩 방식:  
&nbsp;&nbsp;1. Explicit Linking: 프로그램에서 사용되는 순간에 로딩하고 사용이 끝나면 메모리에서 해제되는 방법  
&nbsp;&nbsp;2. Implicit Linking: 프로그램을 시작할 때 같이 로딩되어 프로그램 종료할 때 메모리에서 해제되는 방법(IAT가 이 방법을 따른다.)  
## 13.5.2) IMAGE_IMPORT_DESCRIPTOR
* PE파일은 자신이 어떤 라이브러리를 Import하고 있는지 이 구조체에 명시하고 있다.  

```C
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {
        DWORD Characteristics;
        DWORD OriginalFirstThunk;
    };
    DWORD TimeDateStamp;
    DWORD ForwarderChain;
    DWORD Name;
    DWORD FirstThunk;
} IMAGE_IMPORT_DESCRIPTOR;

typedef struct _IMAGE_IMPORT_BY_NAME {
    WORD Hint;
    BYTE Name[1];
} IMAGE_IMPORT_BY_NAME, *PIMAGE_IMPORT_BY_NAME;
```  

* 일반적인 프로그램에선 보통 여러 개의 라이브러리를 Import하기 때문에 **라이브러리의 개수**만큼 위 구조체의 배열 형식으로 존재하며, 구조체 배열의 마지막은 NULL 구조체로 끝나게 된다. 구조체의 중요 맴버는 모두 *RVA*값을 가진다.  

| 항목 | 의미 |  
|:-------|:---------|  
| OriginalFirstChunk |INT(Import Name Table)의 주소(RVA)  |  
| Name | 라이브러리 이름 문자열의 주소(RVA) |  
| FirstChunk| IAT(Import Address Table)의 주소(RVA) |   

--------------------------  
참고:
&nbsp;&nbsp;1. PE 헤더에서 'Table' 이라고 하면 **배열**을 뜻한다.  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2. INT와 IAT는 long type(4바이트 자료형) 배열이고 NULL로 끝난다.(크기가 따로 명시되어 있지 않다.)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3. INT에서 각 원소의 값은 IMAGE_IMPORT_BY_NAME 구조체 포인터다.(IAT도 같은 값을 가지는 경우가 있다.)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4. INT와 IAT의 크기는 같아야 한다.  

--------------------------  

```
PE 로더가 Import 함수 주소를 IAT 입력하는 순서  
1. IID의 Name 맴버를 읽은 뒤 라이브러리의 이름 문자열("kernel32.dll")을 얻는다.  
2. 해당 라이브러리를 로딩힌다.  
3. IID의 OriginalFirstThunk 맴버를 읽어서 INT 주소를 얻는다.  
4. INT에서 배열의 값을 하나씩 읽어 해당 IMAGE_IMPORT_BY_NAME 주소(RVA)를 얻는다.  
5. IMAGE_IMPORT_BY_NAME의 Hint(ordinal) 또는 Name 항목을 이용하여 해당 함수의 시작 주소를 얻는다.
6. IID의 FirstThunk(IAT) 맴버를 읽어서 IAT 주소를 얻는다.  
7. 해당 IAT 배열 값에 위해서 구한 함수 주소를 입력한다.  
8. INT가 끝날 때까지(NULL을 만날 때까지) 위 4 ~ 7 과정을 반복한다.
```  

## 13.5.3) notepad.exe를 이용한 실습
* IMAGE_IMPORT_DESCRIPTOR 구조체 배열은 **PE 바디**에 존재한다.  
* 이 배열을 찾아가기 위한 정보는 PE 헤더의 IMAGE_OPTIONAL_HEADER32(64)  .DataDirectory[1].VirtualAddress 값이 실제 IID 구조체 배열의 시작 주소다.(RVA 값)  
* 여담으로 IID의 다른 용어는 **IMPORT Directory Table**이라고 한다.  
* 3번째 포스팅에서 IMAGE_OPTIONAL_HEADER64.DataDirectory[1]의 값을 파악해두길 잘했다. -> F448(RVA) 따라서 File Offset = C848  
* 윈도우 XP 가상머신 상에서의 실습으로 변경하면서 책을 그대로 따라가게 되었다.  
* RVA가 7604 이므로 File Offset은 6A04이다.  

| File Offset | Member | RVA | RAW |  
|:--------- |:---------|:-----|:----|  
| 6A04 | OriginalFirstThunk(INT) | 00007990 | 00006D90 |  
| 6A08 | TimeDateStamp | FFFFFFFF | - |  
| 6A0C | ForwarderChain | FFFFFFFF | - |  
| 6A10 | Name | 00007AAC | 00006EAC |  
| 6A14 | FirstThunk(IAT) | 000012C4 | 000006C4 |  

1. 라이브러리 이름(Name)  
&nbsp;&nbsp;Name 항목은 Import 함수가 소속된 라이브러리 파일의 이름 문자열 포인터다. 
![Name](/asset/Name.JPG)  
여기서 "comdlg32.dll"를 발견할 수 있다.
2. OriginalFirstThunk - INT(Import Name Table)  
&nbsp;&nbsp;INT는 Import 하는 함수의 정보(Ordinal, Name)가 담긴 구조체 포인터 배열이다. 이 정보를 얻어야 **프로세스 메모리에 로딩된 라이브러리에서 해당 함수의 시작 주소**를 정확히 구할 수 있다.  
![INT](/asset/INT.JPG)  
INT는 **주소 배열**의 형태로 되어있다.(배열의 끝은 NULL로 되어 있다. 이 주소들은 모두 RVA 값으로 따라가면 Import 하는 **API 함수 이름 문자열**이 나타난다.  
3. IMAGE_IMPORT_BY_NAME  
&nbsp;&nbsp;RVA 7A7A는 RAW 6E7A 이므로 따라가보면,  
![pgset](/asset/PageSet.JPG)

| Hint(ordinal) | Name |  
|:-------- | :----|   
| 000F | PageSetupDlgW |  
| 0006 | FindTextW |  
| 0012 | PrintDlgExW |  
| 0003 | ChooseFontW |  
| 0008 | GetFileTitleW |  
| 000A | GetCpenFileNameW |  
| 0015 | ReplaceTextW |  
| 0004 | CommDlgExtendedError |  
| 000C | GetSaveFileNameW |  

4. FirstThunk - IAT(Import Address Table)  
&nbsp;&nbsp;IAT의 RVA는 12C4 이므로 RAW는 6C4이다.
![iat](/asset/IAT.JPG)  

IAT의 첫 번째 원소 값은 이미 763248D6로 하드코딩되어 있다. 그러나 쓸모 없는 값으로 이 프로그램이 실제 로딩되었을땐 이 값은 정확한 주소 값으로 대체된다.(아마 내 가상 시스템이 Windows XP SP2여서 그런 것 같다.)  

* 175p의 ollydbg로 notepad.exe를 열어서 IAT를 확인하는 예시는 잘 모르겠다. 다시 읽어볼 것 요망.