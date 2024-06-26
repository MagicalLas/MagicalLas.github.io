---
layout: post
title: "CS 다시 돌아보기"
slug: interview-cs
category: cs
---

## Network

### TCP/IP

TCP/IP는 7계층에 영향을 받았지만, 7계층으로 이루어지지는 않았다. 7계층은 의미론적인 개념으로만 남았고, 실제로 TCP/IP와 1:1로 매칭되지는 않는다.

TCP/IP는 TCP와 IP뿐만 아니라 다른 프로토콜들도 포함하고 있지만, 이 두가지 프로토콜이 주요하기에 이렇게 이름붙여졌다.

### TCP

Def: 전송 제어 프로토콜.

#### Connection

TCP는 하나의 IP 소프트웨어를 사용해서 여러 프로세스들이 TCP 통신을 할 수 있도록 다중 연결을 지원한다.



#### Handshake


#### Guarantee

- 데이터가 온전히 전달된다.
- 하지만 데이터가 언제 전달되는지에 대한 시간을 보장하지 않는다.


### Subnet

서브넷은 전체 네트워크가 아닌 더 작은 (흔히 내부) 네트워크를 의미한다. 외부 네트워크(흔히 인터넷)과 서브넷을 구분하여 관리하게 된다.

만약 서브넷이 없다면 특정 IPv4주소를 보았을 때 모든 호스트는 유니크한 값을 가져야 할 수도 있다.

#### IP 관리(CIDR)

서브넷 안의 IP의 범위를 관리하기 위하여 이전에는 class를 통하여 관리했다. 하지만 class를 통한 관리는 상위 비트 마스킹을 통하여 진행했고 class가 제한적이었기에 단점이 많았다. 따라서 class를 통한 IP 범위 관리에서 CIDR(Classless Inter Domain Routing)으로 발전하였다.

CIDR은 10.0.0.0/16과 같이 몇비트를 사용할 것인지에 따라서 표기하게 된다. CIDR은 또한 계층적이기도 하여, 10.0.0.0/16의 범위 안에 10.0.0.0/24가 속한다.

이러한 IP 관리는 서브넷에서 어떻게 IP 범위를 관리할 것인지에 대한 방법이다.

#### Subnet Mask

서브넷 마스크는 특정 IPv4를 보고 이 IP는 서브넷에 있는지 아닌지를 판단하는 마스킹 방법이다. CIDR은 이러한 마스킹의 일종이라고 할 수 있다.

----

### HTTP

Def: HTTP는 Hyper Text Transfer Protocol의 약자로, 말그대로 Hyper Text 전달하는데 사용되는 프로토콜입니다. 웹의 많은 기능들이 이 HTTP 레이어 위에서 구현되어있습니다. HTTP 클라이언트로는 크롬과 같은 브라우저가 대표적입니다.

#### Specification

HTTP는 HTTP 대표적으로 1.0과 2.0, 3.0이 있으며 HTTPS도 2.0부터 포함됩니다.

HTTP 1.0은 기본적인 기능들의 구현이 들어갔습니다. Get, Post, Put과 같은 Method나 Header들에서 정하는 대표적인 기능이 소개되었습니다.

HTTP 2.0은 여러가지 성능을 위한 기능들과 암호화 기법들이 추가되었습니다.

#### Connection

----

## DB

## OS, System Programming

##
