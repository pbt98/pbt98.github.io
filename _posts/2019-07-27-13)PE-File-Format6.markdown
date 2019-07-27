---
layout: post
title:  "13. PE File-Format 6th"
date:   2019-07-27 20:00:00 +0900
categories: Reversing
comments: true
---
# 13.6) EAT  
* 라이브러리: Windows 운영체제에선 **다른 프로그램에서 불러 쓸 수 있도록** 관련 함수들을 모아놓은 파일(DLL/SYS)이다. 예를 들어, kernel32.dll, Win32 API가 있다.  
* EAT(Export Address Table): 라이브러리 파일에서 제공하는 함수를 다른 프로그램에서 가져다 사용할 수 있도록 해주는 매커니즘이다.  
* **EAT를 통해서만 해당 라이브러리에서 Export하는 함수의 시작 주소를 정확히 구할 수 있다.**  
* IAT와 같은 방법으로 IMAGE_EXPORT_DIRECTORY에 Export 정보를 저장하고 있다.  
* 그러나 IAT와는 다르게 라이브러리의 EAT를 설명하는 이 구조체는 PE 파일에 **하나만** 존재한다.  
* PE 파일에서 이 구조체의 위치는 **PE 헤더**에서 찾을 수 있다. -> *IMAGE_OPTIONAL_HEADER32.DataDirectory[0].VirtualAddress* 값이 *IMAGE_EXPORT_DIRECTORY* 구조체 배열의 시작 주소이다.(역시 RVA 값이다.)  

![ioh](/asset/ioh.JPG)  

|offset|value|description|  
|:-----|:----|:----------|  
|00000160 | 0000261C|RVA of EXPORT Directory|  
|00000164 | 00006C7B|size of EXPORT Directory |  

* RVA = 261C 이므로 RAW = 1A1C이다.  

## 13.6.1) IMAGE_EXPORT_DIRECTORY  

```C  
typedef struct _IMAGE_EXPORT_DIRECTORY {
    DWORD Characteristics;
    DWORD TimeDateStamp;
    WORD MajorVersion;
    WORD MinorVersion;
    DWORD Name;
    DWORD Base;
    DWORD NumberOfFunctions;
    DWORD NumberOfNames;
    DWORD AddressOfFunctions;
    DWORD AddressOfNames;
    DWORD AddressNameOrdinals;
} IMAGE_EXPORT_DIRECTORY, *PIMAGE_EXPORT_DIRECTORY;
```  

|항목 | 의미|
|:---|:----|  
|NumberOfFuctions|실제 Export 함수 개수|  
|NumberOfNames | Export 함수 중에서 이름을 가지는 함수 개수|  
|AddressOfFunctions| Export 함수 주소 배열|  
|AddressOfNames|함수 이름 주소 배열|  
|AddressOfNameOrdinals| Ordinal 배열 |  

* 라이브러리에서 함수 주소를 얻는 API는 GetProcAddress()이다. 이 API가 EAT를 참조해서 원하는 API의 주소를 구해낸다.

```
참고: GetProcAddress() 동작 원리  
1. AddressOfNames 맴버를 이용해 '함수 이름 배열'로 간다.  
2. '함수 이름 배열'은 문자열 주소가 저장되어 있다. 문자열 비교(strcmp)를 통하여 원하는 함수 이름을 찾는다.(이때 배열의 인덱스를 name_index라 하자.)  
3. AddressOfNameOrdinals 맴버를 이용해 'ordinal 배열'로 간다.  
4. 'ordinal 배열'에서 name_index로 해당 ordinal 값을 찾는다.  
5. AddressOfFunctions 맴버를 이용해 '함수 주소 배열(EAT)'로 간다.  
6. '함수 주소 배열(EAT)'에서 아까 구한 ordinal을 배열 인덱스로 하여 원하는 함수의 시작 주소를 얻는다.  
```  

## 13.6.2) kernel32.dll을 이용한 실습  
* RVA = 261C 이므로 RAW = 1A1C 이다.  

![gpa](/asset/GPA.JPG)  

|File Offset | Memeber | Value | RAW |  
|:-----------|:--------|:------|:----|  
|1A1C|Characteristics |00000000| - |  
|1A20|TimeDateStamp | 44AB80AC | - |
|1A24|MajorVersion | 0000 | - |  
|1A26|MinorVersion | 0000 | - |  
|1A28|Name| 00004B56| 3F56 |  
|1A2C|Base | 00000001| - |  
|1A30| NumberOfFunctions | 000003B5 | - |  
|1A34 | NumberOfNames | 000003B5 | - |  
|1A38 | AddressOfFunctions | 00002644 | 1A44 |  
|1A3C | AddressOfNames | 00003518 | 2918 |  
|1A40 | AddressOfNameOrdinals | 000043EC| 37EC |  

1. 함수 이름 배열로 간다.  
&nbsp;&nbsp;AddressOfNames 맴버의 값은 RAW = 2918이고, NumberOfFunctions에 의해 3B5 * 4 + 2918 = 37EC이므로, 2918 ~ 37EC까지 모두 함수 이름 배열의 원소이다.  

![aon](/asset/AON.JPG)  

2. 함수 이름 찾기  
&nbsp;&nbsp;세 번째 원소 RVA = 00004B7B 를 이용해서 알아내보면, RAW = 3F7B 이므로, 'AddAtomW'를 찾아낼 수 있다.

![add](/asset/AddAtom.JPG)  

3, 4. Ordinal 배열  
&nbsp;&nbsp;AddressOfNameOrdinals 맴버의 값은 RAW = 37EC 이므로, index = 2에 맞춰서 찾아보면 Ordinal = 2 임을 찾아낼 수 있다.(kernel32.dll의 특성이다.)  

![ord](/asset/Ordinals.JPG)  

5. 함수 주소 배열 - EAT  
&nbsp;&nbsp;'AddAtomW'의 실제 함수 주소를 아까 구한 Ordinal을 이용해서 찾아갈 수 있다. AddressOfFunctions 맴버의 값은 RVA 주소 값의 배열이고 RAW = 1A44 이므로, 찾아보면 RVA = 00032619임을 알 수 있다.  

![real](/asset/real.JPG)  

6. AddAtomW 함수 주소  
&nbsp;&nbsp;RVA = 00032619이므로 'AddAtomW' 함수의 실제 주소(VA)는 kernel32.dll의 ImageBase = 7C800000 이므로 7C800000 + 32619 = 7C832619 이다. ollydbg로 확인해보면, 실제 함수임을 알 수 있다.  

![real_](/asset/real_atom.JPG)  

# 13.7) Advanced PE  
* 책에선 Hex Editor와 연필, 종이만 가지고 IAT/EAT의 주소를 하나하나 계산해서 파일/메모리에서 실제 주소를 찾는 훈련이 중요하다고 했다. 즉, 실습의 반복이 중요하다 강조했다.  
## 13.7.1) PEView.exe  
* PE 헤더를 각 구조체별로 보기 쉽게 표현해준다.  
* RVA <-> File Offset 변환을 간단히 수행해준다.  
* 책에선 이걸 콘솔 버젼이나 GUI로 만들어보라 추천한다.  
## 13.7.2) Patched PE  
* 일반적인 PE 규격을 벗어난 것들 이지만, Windows 운영체제에서 실행이 됨을 확인할 수 있다.  

# 13.8) 마무리
* PE 스펙은 그저 스펙일 뿐, 만들어 놓고 사용되지 않는 내용이 많다.  
* 내가 지금 배운 PE 헤더에 대한 지식도 잘못된 부분이 있을 수 있다.(tiny PE 외에도 PE 헤더를 조작하는 여러 창의적인 기법들이 있다.)  
* 항상 모르는 부분을 체크하자.  
* 책에서 강조한 연습은 꼭 다시 연습하자.  
