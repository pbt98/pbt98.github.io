---
layout: post
title:  "Starting Bixby Forensic"
date:   2019-06-21 22:30:00 +0900
categories: Graduate School
comments: true
---
# 소개
Library Detection과 관련된 연구를 진행하던 중에 필요한 정보를 뽑아낼 수 있는 테크닉이 부족하다 느껴 공부를 하고있었다.
Bixby는 안드로이드 시스템과도 밀접한 관련을 가진 앱이고, 이 앱이 데이터를 저장하는 메커니즘을 알아낼 수 있다면 다른 앱에서의 Library Detection에 대한 방법론도 조금은 윤곽이 잡히지 않을까 생각한다.

# 우선 과제
* Android permission과 관련된 API에 집중
* Dynamic Analysis로 음성 파일이 저장되는 과정 확인
