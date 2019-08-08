---
layout: post
title:  "19. Debugging UPack - Finding OEP"
date:   2019-08-07 17:13:00 +0900
categories: Reversing
comments: true
---
## 19.1) OllyDbg 실행 에러  
* UPack은 IMAGE_OPTIONAL_HEADER에서 NumberOfRvaAndSizes 값을 A로 변경하기 때문에 OllyDbg 초기 검증 과정에서 에러가 발생한다.  
* 이 에러때문에 EP로 가지 못하고 ntdll.dll 영역에서 멈춰버리는데, 강제로 EP를 설정해줘야한다.  
* Stud PE 에서 확인해보면, ImageBase = 1000000, EntryPoint(RVA) = 1018 임을 확인할 수 있다.  
![st](/asset/st.JPG)  
![rva](/asset/rva.JPG)  

## 19.2) 디코딩 루프  
* 디코딩 루프를 디버깅할 때는 **조건 분기를 적절히 건너 뛰어 루프를 탈출해야**하고, **레지스터를 잘 보면서 어떤 주소에 값을 쓰고 있는지 잘 살펴야**한다.  
* UPack은 두 번째 섹션에 압축된 원본 데이터가 존재하고, 이 데이터를 디코딩 루프를 돌면서 첫 번째 섹션에 압축해제한다.  

![oep](/asset/oep.JPG)  

* 처음 두 명령은 010011B0 주소에서 4바이트를 읽어서 EAX에 저장하는 명령이다. EAX는 0100739D 값을 가지는데, 이는 **원본 notepad의 OEP(Original Entry Point)다.**  
* LODS DWORD PTR DS:[ESI] 는 ESI가 가리키는 주소에서 4바이트 값을 읽어서 EAX 레지스터에 저장하라는 의미이다.  

![deco](/asset/deco.JPG)  

* 여기가 decode() 함수의 주소이다. 이 부분을 따라가다 보면,  
![trac](/asset/trac.JPG)  

* 0101FE57과 0101FE5D 주소에는 EDI 값이 가리키는 곳에 뭔가 쓰는 명령어(MOVS, STOS) 명령어가 있음을 확인할 수 있다. 이때 EDI 값은 **첫 번째 섹션 내의 주소**를 가리킨다. -> **압축을 해제한 후 실제 메모리에 쓰는 명령어들**이다.  
* 0101FE5E와 0101FE61 주소에는 CMP/JB 명령어를 통해 EDI 값이 01014B5A가 될 때까지 계속 루프를 돌게된다.([ESI + 34] = 01014B5A).  
* 0101FE61 주소가 디코딩 루프의 끝부분이다. 이 루프를 트레이싱하면 EDI가 가리키는 주소에 어떤 값들이 쓰여지는 것을 볼 수 있다.  

## 19.3) IAT 세팅  
![LB](/asset/LB.JPG)  
![GP](/asset/GP.JPG)  

* UPack이 임포트하는 2개의 함수 LoadLibraryA와 GetProAddress를 이용하여 루프를 돌면서 원본 notepad의 IAT를 구성한다.(notepad에서 임포트 하는 함수들의 실제 메모리 주소를 얻어 원본 IAT 영역에 쓰는 것이다.)  

![op](/asset/op.JPG)  

* 우리가 많이 보던 원본 notepad의 EP 코드다!  

## 19.4) 마무리  
* UPack은 PE 헤더는 분석이 어렵지만, 디버깅은 쉬운 편이다. 익숙해질 때까지 반복해보자.  
* PE 스펙은 단지 스펙일 뿐이고, 실제 구현은 PE 로더 개발자에 의해 좌우되기 때문에(예를 들어, FileAlignment의 배수에 맞지 않으면 강제로 0으로 처리한다던지), OS별로 실제 테스트를 해봐야한다.  
* PE 헤더에는 실제로는 사용되지 않는 항목이 많음을 깨우쳐주는 단원이었다.  

