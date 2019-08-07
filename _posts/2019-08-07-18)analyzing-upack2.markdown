---
layout: post
title:  "18. Analyzing UPack PE header 2nd"
date:   2019-08-07 13:02:00 +0900
categories: Reversing
comments: true
---
# 18.5.4) IMAGE_SECTION_HEADER  
* IMAGE_SECTION_HEADER 구조체에서 프로그램 실행에 사용되지 않는 항목들에는 UPack 자신의 데이터를 덮어쓴다.  
* 앞에서 섹션의 개수는 3개, 시작 옵셋은 170임을 알고 있으므로 밑의 그림과 같이 확인할 수 있다.  
![ishmo](/asset/ishmo.JPG)  

|name|
|:-----|
|offset to relocations|  
|offset to line numbers|  
|number of relocations |  
|number of line numbers|  

* 위의 맴버들은 프로그램 실행에 아무런 의미 없는 맴버들이다.  

# 18.5.5) 섹션 겹쳐쓰기  
![secmo](/asset/secmo.JPG)  

* 첫번째 섹션과 세번째 섹션의 파일 시작 옵셋(RawOffset)이 10으로 같고, 정상적인 PE 파일에선 헤더 영역인데 **섹션**이 시작된다.  
* 첫번째 섹션과 세번째 섹션의 파일에서의 크기가 같다.  
* 여기서 주목해야할 사실은 **같은 파일 이미지를 가지고 각각 다른 위치와 다른 크기의 메모리 이미지를 만들 수 있다는 사실**에 주목해야한다.  
* 파일의 헤더 영역의 크기는 200이고, 두 번째 섹션 영역의 크기(AF28)는 파일의 대부분을 차지할 정도로 크다. -> 이곳에 원본 파일(notepad.exe)이 압축되어 있다.  
* **메모리에서의 첫 번째 섹션 영역** 또한 주목해야한다. 섹션의 메모리 크기는 14000인데, 이는 원본 파일의 Size of Image와 같은 값이다. -> **두 번째 섹션에 압축되어있는 파일 이미지를 첫 번째 섹션에 그대로 압축해제하는 것이다.**  
* 정리하자면, 메모리 두 번째 섹션 영역에 압축된 notepad가 들어 있고, 압축이 풀리면서 첫 번째 섹션 영역에 기록된다. 중요한 것은 notepad.exe의 메모리 이미지가 **통째로 풀리기** 때문에 프로그램이 정상적으로 실행될 수 있다.(주소가 정확히 일치하게 된다.)  

# 18.5.6) RVA to RAW  

```  
일반적으로 우리가 알고 있는 RVA to RAW  
RAW = RVA - VirtualAddress + PointerToRawData  
```  

* 위 식대로 EP의 파일 옵셋(RAW)를 계산하면 UPack의 EP는 RVA 1018이므로(밑 그림 참고) RAW = 1018 - 1000 + 10 = 28    
![addr](/asset/addr.JPG)  
![wrong_rva](/asset/wrong_rva.JPG)  

* 그러나, 여긴 코드가 아니라 문자열 영역임을 알 수 있다.  
* 일반적으로 섹션 시작의 파일 옵셋을 가리키는 **PointerToRawData값은 FileAlignment의 배수**가 되어야 한다. UPack의 FileAlignment는 200이므로 PointerToRawData 값은 0, 200, 400, 600 등이 되어야한다. -> **PE 로더는 첫 번째 섹션의 PointerToRawData(10)가 FileAlignment(200)의 배수가 아니므로 강제로 배수에 맞춰서 인식한다.(이 경우는 0)**  

```  
따라서 맞게 고치면,  
RAW = 1018 - 1000 + 0 = 18  
```  

# 18.5.7) Import Table(IMAGE_IMPORT_DESCRIPTOR array)  
* 먼저 Directory Table에서 Import Directory Table(IMAGE_IMPORT_DESCRIPTOR 구조체 배열) 주소를 얻어야 한다.  
![IMT](/asset/IMT.JPG)  

* 위의 그림에서 앞의 4바이트가 Import Table의 주소(RVA), 뒤의 4바이트가 Import Table의 크기를 나타낸다. 따라서 RVA = 000271EE 이다.  
* 271EE는 세 번째 섹션 영역이므로(18.5.5 참조) RAW = 271EE - 27000 + **0**(강제변환) = 1EE  

![1ee](/asset/1ee.JPG)  

* PE 스펙에 따르면 **Import Table은 IMAGE_IMPORT_DESCRIPTOR 구조체 배열로 이루어지고 마지막은 NULL 구조체로 끝나야 한다.**  

```C  
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {
        DWORD Characteristics;  
        DWORD OriginalFirstThunk;  //INT 
    };
    DWORD TimeDateStamp;  
    DWORD ForwarderChain;  
    DWORD Name;  
    DWORD FirstThunk;  //IAT  
} IMAGE_IMPORT_DESCRIPTOR;  
```  

* 선택된 영역이 Import Table을 나타내는 IMAGE_IMPORT_DESCRIPTOR 구조체 배열이다. 1EE ~ 201 옵셋까지 첫 번째 구조체이며, 그 뒤는 두 번째 구조체도 아니고, 그렇다고 끝을 나타내는 NULL 구조체도 아니다.  
* 그러나, 세 번째 섹션은 0 ~ 1FF 이므로 200 옵셋 이하는 **세 번째 섹션 메모리에 매핑되지 않는다.**  
* 세 번째 섹션이 메모리에 로딩될 때 파일 옵셋 0 ~ 1FF는 27100 ~ 271FF에 매핑 되고 27200 ~ 28000 영역은 NULL로 채워진다!(IMAGE_IMPORT_DESCRIPTOR 구조체 배열이 어떻게 끝나는지 상기해보자.) -> 따라서 PE 스펙에 저촉되지 않는다.  

# 18.5.8) IAT(Import Address Table)  
* 위의 IMAGE_IMPORT_DESCRIPTOR 구조체를 살펴보면  

|offset|member|RVA|  
|:-------|:-----|:-------|  
|1EE|OriginalFirstThunk(INT)| 0|  
|1FA|Name| 2|  
|1FE| FirstThunk(IAT)| 11E8|  

* Name의 RVA 값은 2이고, 이는 Header 영역에 속한다.(첫 번째 섹션이 RVA 1000에서 시작하므로)  
* 헤더 영역에선 **그냥 RVA와 RAW 값이 같다.**  
![dkfEmf](/asset/dkfEmf.JPG)  

* 보통은 OriginalFirstThunk(INT)를 쫓악면 API 이름 문자열이 나오지만, UPack과 같이 OriginalFirstThunk(INT)가 0일 때는 **FirstThunk(IAT) 값을 따라가도 상관없다.**(INT, IAT나 API 이름 문자열만 나오면 된다.)  

```  
RAW = 11E8 - 1000 + 0 = 1E8  
```  

![iatmo](/asset/iatmo.JPG)  

* 위 그림에서 IAT와 INT의 역할을 동시에 하고 있는데, 이곳은 Name Pointer(RVA) 배열이고 배열의 끝은 NULL이다. 또한 2개의 API를 import하는 것을 알 수 있다. -> 모두 헤더 영역이므로 RVA = RAW다.  

## 18.6) 마무리  
* PE 스펙의 허술함을 이용해서 이렇게 까지 용량을 줄일 수 있는 Packer가 있을줄은 몰랐다. 다른 Packer들도 분석해보면서 나만의 PE View를 만들 계획을 세워야 겠다.