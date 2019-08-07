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
