---
layout: post
title:  "18. Analyzing UPack PE header 1st"
date:   2019-08-06 19:20:00 +0900
categories: Reversing
comments: true
---
## 18.1) UPack 설명  
* PE 헤더를 기형적으로 변형 시키는 Packer라 한다. -> 이런 특이한 PE 헤더 때문에 각종 PE 유틸리티들(debugger, PE viewer 등)이 정상적으로 작동하지 않았다.  
* 이러한 특징 때문에 많은 악성코드가 UPack으로 실행 압축되어 배포되었고, 이에 많은 Antivirus 제품들이 UPack으로 실행 압축된 파일들은 모두 바이러스로 진단하는 지경에 이르렀다.  
* 이러한 특징 때문에 이번 장에서 UPack을 분석하거나 UPack으로 실행 압축된 PE 파일을 분석하기 위해서는 안티바이러스를 끄고 분석해야만 했다.  

## 18.2) UPack으로 notepad.exe 실행 압축하기  
* UPack으로 실행 압축 시 PE 파일의 백업은 필수다.  
* 디폴트 옵션으로만해도 충분하다.  
![upackc](/asset/upackc.JPG)  
![ilpe](/asset/ilpe.JPG)  

* PE view로 보면 제대로 읽어들이지 못하는 모습을 볼 수 있다. (IMAGE_OPTIONAL_HEADER, IMAGE_SECTION_HEADER 등의 정보가 없다.)  

## 18.3) Stud_PE 이용  
* http://www.cgsoftlabs.ro/zip/Stud_PE.zip
* 내가 받은 버젼은 2.6.1.0이다.  

## 18.4) PE 헤더 비교  
# 18.4.1) 원본 PE 헤더  

![notepad](/asset/notepad.jpg)  

* IMAGE_DOS_HEADER, DOS Stub, IMAGE_NT_HEADERS, IMAGE_SECTION_HEADER 순으로 정상적인 PE 헤더를 보여주고 있다.  

# 18.4.2) notepad_upack.exe(실행 압축)의 PE 헤더  

![upack_note](/asset/upack_note.JPG)  

* MZ, PE 시그니처가 매우 가깝게 붙어있다.  
* DOS Stub는 아예 사라졌다.  

## 18.5) UPack의 PE 헤더 분석  
# 18.5.1) 헤더 겹쳐쓰기  
* MZ 헤더(IMAGE_DOS_HEADER)와 PE 헤더(IMAGE_NT_HEADERS)를 교묘하게 겹쳐쓰는 기법이다.  

![stud_basic](/asset/stud_basic.JPG)  

* MZ 헤더에서 아래 2가지 맴버가 **가장 중요**하다  

```  
(offset  0) e_magic  : Magic number = 4D5A('MZ')  
(offset 3C) e_lfanew : File address of new exe header  
```  

* 저 두개 외의 나머지는 프로그램 실행에 중요하지 않다.  
* 문제는 PE File Format의 스펙에 따라서 IMAGE_NT_HEADERS의 시작 위치가 '가변적'이라는 것이다. -> e_lfanew의 값에 따라서 IMAGE_NT_HEADERS의 시작 위치가 결정된다.  

```
e_lfanew = MZ 헤더 크기(40) + DOS Stub 크기(가변 : VC++의 경우 보통 A0) = E0  
```   

* UPack에서는 e_lfanew 값이 10이다. PE 스펙에 어긋나진 않다.  

# 18.5.2) IMAGE_FILE_HEADER.SizeOfOptionalHeader  
* IMAGE_FILE_HEADER.SizeOfOptionalHeader의 값을 변경해서 헤더 안에 디코딩 코드를 삽입할 수 있게 만든다.  
* 값의 의미는 IMAGE_OPTIONAL_HEADER 구조체의 크기이다. UPack은 이 값을 148로 변경한다.  
![coffh](/asset/coffh.JPG)  

* IMAGE_OPTIONAL_HEADER는 PE32 파일 포맷에서 크기는 이미 E0로 결정되어 있지만, 설계자들은 PE 파일의 형태에 따라서 각각 다른 IMAGE_OPTIONAL_HEADER 형태의 구조체를 **바꿔 낄 수 있도록** 설계한 것이다. -> IMAGE_OPTIONAL_HEADER64 같은 경우를 생각해보면 된다.  
* SizeOfOptionalHeader의 또 다른 의미는 섹션 헤더(IMAGE_SECTION_HEADER)의 시작 옵셋을 결정한다는 것이다.  
* **IMAGE_OPTIONAL_HEADER 시작 옵셋에 SizeOfOptionalHeader 값을 더한 위치(옵셋)부터 IMAGE_SECTION_HEADER가 나타난다.**  
* 따라서 UPack 에서는 SizeOfOptionalHeader 값이 148 + IMAGE_OPTIONAL_HEADER 시작 옵셋(28) = 170(16진수 덧셈은 헷갈리지 말자.) 에서 IMAGE_SECTIONAL_HEADER가 시작된다.  
* UPack의 의도는 SizeOfOptionalHeader의 값을 조절해서 IMAGE_OPTIONAL_HEADER와 IMAGE_SECTIONAL_HEADER의 사이에 공간을 확보해 **디코딩에 필요한 코드**를 적절히 끼워 넣는 것이다.  
![decoding](/asset/decoding.JPG)  

# 18.5.3) IMAGE_OPTIONAL_HEADER.NumberOfRvaAndSizes  

|Index|Contents|  
|:---------|:--------------|  
|0|EXPORT Directory|  
|1|***IMPORT Directory***|  
|2|***RESOURCE Directory***|  
|3|***EXCEPTION Directory***|  
|4|SECURITY Directory |  
|5|BASERELOC Directory |  
|6|***DEBUG Directory***|  
|7|COPYRIGHT Directory |  
|8|GLOBALPTR Directory |  
|9|***TLS Directory***|  
|A|***LOAD_CONFIG Directory***|  
|B|***BOUND_IMPORT Directory***|  
|C|***IAT Directory***|  
|D|DELAY_IMPORT Directory|  
|E|***COM_DESCRIPTOR Directory***|  
|F|Reversed Directory|  

* 진하게 표시된 내용은 잘못 변경했을 때 실행 에러가 발생하는 항목들이다.  
* 이 값의 의미는 바로 뒤에 이어지는 IMAGE_DAATA_DIRECTORY 구조체 배열의 원소 개수를 나타낸다.  
* 정상 PE 파일 : 10, UPack PE 파일: A  
![rvamo](/asset/rvamo.JPG)  

* IMAGE_DATA_DIRECTORY 구조체 배열의 원소 개수는 10로 정해져 있지만, PE 스펙에 따르면 NumberOfRvaAndSizes 값을 **원소 개수**로 인정하게 되어있다. -> UPack에서는 이를 이용해서 A로 바꾸었다.  
* 변경하면 LOAD_CONFIG(파일 옵셋 D8 이후)부터는 사용하지 않는다. 그리고 무시된 영역에 **자신의 코드**를 덮어썼다.  
![iddmo](/asset/iddmo.JPG)

* 흐리게 표시된 부분이 무시되지 않는 IMAGE_DATA_DIRECTORY 부분이고, 노란색으로 표시된 부분이 무시되는 부분이다. -> UPack 자체의 디코딩 코드다.