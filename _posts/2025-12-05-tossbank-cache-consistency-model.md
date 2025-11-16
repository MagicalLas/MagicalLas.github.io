---
layout: post
title: "That's not strong consistency cache architecture"
slug: tossbank-strong-cache
category: essay
---

## urge to write this article

회사에서 이야기를 나누다가 어떤 아티클을 공유받았습니다. 토스뱅크에서 작성한 캐시에 대한 글이었고,
꽤나 어려워보이는 개념들을 이야기하면서 멋진 시스템 개선 방식을 소개하고있어서 즐겁게 읽었습니다. 
한편으로는 분산 시스템에서의 strong consistency를 적용하는 방식이 꽤나 특이하다는 느낌도 받았습니다. 
읽다보니 점점 엄밀한 잣대로 글을 읽게 되더군요. 제 안의 분산시스템 악마가 깨어나고 있었습니다.
이렇게 간단히 성능 저하 없이 문제글 잘 해결하는게 불가능하다는 생각이 들면서 몇가지 반례를 찾아내었습니다.
이미 저 글의 표현이 잘못되었다는건 확인했지만, 단순히 트집잡기에 불과하여 글을 약 반년간 방치했는데요.
이번에 저도 분산 시스템의 안정성을 검증해야하는 일이 생겨서 좋은 훈련으로 사용해보았습니다.

> 멋진 캐시 적용 아티클을 비난하고싶은 마음은 없습니다. 토스 뱅크의 기술력은 뛰어나다고 생각하며,
> 매우 거대한 시스템을 안정적으로 운영하는 몇 안되는 은행입니다. 저의 전세 대출도 여기서 받았습니다 ;>

## 아티클을 요약하자면

## 일관성 모델

## 반례

## 서킷브레이커는 도움이 되지 않는다

## DB Transaction으로는 도움이 되지 않는다

## 대안 1 - lock

## 대안 2 - lamport timestamp

## 마무리
/\  clientRequestType = [u1 |-> "Write", u2 |-> "Read", u3 |-> "Read"]
/\  clientRequestVal = [u1 |-> "v1", u2 |-> "none", u3 |-> "none"]
/\  clientState = [u1 |-> "IDLE", u2 |-> "IDLE", u3 |-> "IDLE"]
/\  clientView = "v1"
/\  dbValue = "v1"
/\  msgs = {}
/\  redisValue = "none"
/\  serverState = [s1 |-> "IDLE", s2 |-> "IDLE", s3 |-> "IDLE", s4 |-> "IDLE"]
