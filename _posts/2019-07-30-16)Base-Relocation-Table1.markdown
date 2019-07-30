---
layout: post
title:  "16. Base Relocation Table 1st"
date:   2019-07-30 22:40:00 +0900
categories: Reversing
comments: true
---
## 16.1) PE 재배치  
* 이번 장에선 Windows 7 환경이 필요하다.
* Review) PE 파일이 프로세스 가상 메모리에 로딩될 때 PE 헤더의 ImageBase 주소에 로딩된다.  
* DLL(SYS) 파일의 경우엔 ImageBase 위치에 이미 다른 DLL(SYS) 파일이 로딩되어 있다면 다른 비어 있는 주소 공간에 로딩된다. -> 이것을 PE 재배치라 한다.  
* 즉, PE 재배치란 PE 파일이 ImageBase에 로딩되지 못하고 다른 주소에 로딩될 때 수행되는 일련의 작업들을 의미한다.  

# 16.1.1) DLL/SYS  
* ImageBase 위치에 다른 DLL(SYS)이 로딩되어있을땐 빈 주소에 로드한다.  

# 16.1.2) EXE  
* 프로세스가 생성될 때 EXE 파일이 가장 먼저 메모리에 로딩되기 때문에 재배치를 고려할 필요 없었다. -> Windows Vista 이전  
* ASLR(Address Space Layout Randomization) 기능이 추가되어서 고려해야한다. -> Windows Vista 이후  

## 16.3) PE 재배치 동작 원리  
1. 프로그램에서 하드코딩된 주소 위치를 찾는다.  
2. 값을 읽은 후 ImageBase 만큼 뺀다.(VA -> RVA)  
3. 실제 로딩 주소를 더한다.(RVA -> VA)  

* 여기서 핵심은 **하드코딩**된 주소 위치를 찾는 것이다.  
* 이를 위해서 PE 파일 내부에 Relocation Table이라고 하는 하드코딩 주소들의 옵셋(위치)을 모아 놓은 목록이 존재한다.(Relocation Table은 PE 파일 빌드(컴파일/링크) 과정에서 제공된다.) Relocation Table로 찾아가는 방법은 PE 헤더의 Base Relocation Table 항목을 따라가는 것이다.  

# 16.3.1) Base Relocation Table  
* Base Relocation Table 주소는 PE 헤더에서 DataDirectory 배열의 여섯 번째 항목에 들어있다.(배열의 Index = 5)  
* *IMAGE_NT_HEADERS\IMAGE_OPTIONAL_HEADER\IMAGE_DATA_DIRECTORY[5]  
