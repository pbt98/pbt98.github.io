---
layout: post
title:  "Fixing the site"
date:   2020-09-12 19:55:00 +0900
categories: Story
comments: true
---
# 사이트가 먹통이 되었다?
어느 날 부터인가 사이트에 포스트를 올려도 메인에 업데이트가 되지 않음을 발견했다.  
Github에서는 Kramdown의 버전이 낮다는 경고를 보내어 아마 그것이 원인이지 않을까 하고 포스팅을 그냥 쉬었었다.  

# 고쳐보자
파일을 결국 로컬에 끌고 와서 *bundle*을 이용해 로컬에서 디버깅을 시작했다.  
그 결과, **Graduate School** 카테고리가 따옴표로 묶여있지 않아서 **Graduate**로 표시되고 사이트에서 착오를 일으키고 있었던 것이다...  
아직 대학원생도 아닌데 뭣하러 Graduate School이라 쓰나 하는 생각이 들어, 그냥 Laboratory로 고쳤다. 그러고 나니 잘 되더라.  