---
layout: post
title:  "16. Base Relocation Table 2nd"
date:   2019-08-06 14:40:00 +0900
categories: Reversing
comments: true
---
# 16.3.1) Base Relocation Table  
* Base Relocation Table 주소는 PE 헤더에서 DataDirectory 배열의 여섯 번째 항목에 들어있다.(배열의 Index = 5)  
* *IMAGE_NT_HEADERS\IMAGE_OPTIONAL_HEADER\IMAGE_DATA_DIRECTORY[5]*  
![brl](/asset/brl.JPG)  
![2f](/asset/2f.JPG)  
* BASE RELOCATION Table의 RVA가 2F000이므로 따라가보면 2번째 사진과 같이 확인할 수 있다.  

# 16.3.2) IMAGE_BASE_RELOCATION 구조체  
* 위의 그림에선 Base Relocation Table에 하드코딩 주소들의 옵셋(위치)들이 나열되어 있다. -> 이 테이블만 읽어 내면 하드코딩 주소 옵셋을 정확히 알아낼 수 있다.  
* Base Relocation Table은 *IMAGE_BASE_RELOCATION* 구조체 배열이다.  
```C
typedef struct _IMAGE_BASE_RELOCATION {
    DWORD VirtualAddress;
    DWORD SizeOfBlock;
//    WORD TypeOffset[1];
} IMAGE_BASE_RELOCATION;
typedef IMAGE_BASE_RELOCATION UNALIGNED * PIMAGE_BASE_RELOCATION;

#define IMAGE_REL_BASED_ABSOLUTE       0
#define IMAGE_REL_BASED_HIGH           1
#define IMAGE_REL_BASED_LOW            2
#define IMAGE_REL_BASED_HIGHLOW        3
#define IMAGE_REL_BASED_HIGHADJ        4
#define IMAGE_REL_BASED_MIPS_JMPADDR   5
#define IMAGE_REL_BASED_MIPS_JMPADDR16 9
#define IMAGE_REL_BASED_IA64_IMM64     9
#define IMAGE_REL_BASED_DIR64         10
```  

| 맴버 이름 | 의미 |  
|:---------|:------------|  
|VirtualAddress| 기준 주소(Base Address)이며, 실제로는 RVA 값임|  
|SizeOfBlock| 각 단위 블록의 크기 의미|  
|TypeOffset| 이 구조체 밑으로 WORD 타입의 배열이 따라온다는 뜻 -> 이 배열이 프로그램에 하드코딩된 주소들의 옵셋|  

# 16.3.3) Base Relocation Table의 해석 방법  
| RVA| Data| Comment|  
|:-----|:---------|:---------|  
|0002F000| 00001000| VirtualAddress|  
|0002F004| 00000150| SizeOfBlock|  
|0002F008| 3420   | TypeOffset|  
|0002F00A| 342D  | TypeOffset|  
|0002F00C| 3436 | TypeOffset|  
|.... |....| ....|  

* 위의 그림에서 일부를 표현한 표이다.  
* (잘 이해되지 않는 내용) IMAGE_BASE_RELOCATION 구조체 정의에 따르면 VirtualAddress 맴버의 값은 1000이고, SizeOfBlock 맴버의 값은 150이다.  
* 즉, TypeOffset 배열의 기준 주소는 RVA 1000 이며, 블록의 전체 크기는 150이다.(기준 주소별로 이러한 블록이 배열 형태로 존재한다.)  
* 블록의 끝은 0으로 표시한다.  
* TypeOffset 값은 2바이트(16비트) 크기를 가지며 Type(4비트)과 Offset(12비트)이 합쳐진 형태이다. ex) TypeOffset 값이 3420이면, 밑의 표처럼 해석된다.  

|Type(4비트)|Offset(12비트)|  
|:--------|:------|  
|3|420|   

* 최상위 4비트는 Type으로 사용된다. PE 파일에서 일반적인 값은 3(IMAGE_REL_BASED_HIGHLOW)이고, 64비트용 PE+ 파일에서는 A(IMAGE_REL_BASED_DIR64)이다.  
* TypeOffset의 하위 12비트가 진짜 Offset을 의미한다. 이 값은 VirtualAddress 기준의 옵셋이다. 따라서 프로그램에서 하드코딩 주소가 있는 옵셋은 다음과 같이 계산된다.  
```
VirtualAddress(1000) + Offset(420) = 1420(RVA)  
```  
![ext](/asset/ext.JPG)  
* 위 그림에서 볼수 있듯이, 현재 내 머신에선 notepad.exe가 000F0000에 로드 되었는데, RVA 1420에 적힌 하드코딩 된 주소 값을 찾아보면,  
![r](/asset/rva_1.JPG)  

* 010010C4임을 알 수 있다.  
* 값을 읽은 후엔 ImageBase 만큼 빼준다.(VA -> RVA) 010010C4 - 01000000 = 000010C4  
* 실제 로딩 주소를 더한다.(RVA -> VA)  000010C4 + 000F0000 = 000F10C4  
![c0](/asset/10c4.JPG)  

* PE 로더는 내가 찾아낸 과정을 통해서 프로그램 내에 하드코딩된 주소(010010C4)가 아닌 실제 로딩된 메모리 주소(000F0000)에 맞게 보정을 한 값(000F10C4)을 같은 위치에 덮어쓴다.  
* 이 과정을 하나의 IMAGE_BASE_RELOCATION 구조체의 모든 TypeOffset에 대해 반복하면 RVA 1000 ~ 2000 주소 영역에 해당되는 모든 하드코딩 주소에 대해 PE 재배치 작업이 수행된 것이다.  
