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
이렇게 간단히 성능 저하 없이 문제를 잘 해결하는게 불가능하다는 생각이 들면서 몇가지 반례를 찾아내었습니다.
블로그 길이의 제약으로 인해 모든걸 설명하지 못한것을 비판하는건 단순히 트집잡기에 불과하여 글을 약 반년간 방치했는데요.
이번에 저도 분산 시스템의 안정성을 검증해야하는 일이 생겨서 좋은 훈련으로 사용해보았습니다.

> 멋진 캐시 적용 아티클을 비난하고싶은 마음은 없습니다. 토스 뱅크는 매우 거대한 시스템을 안정적으로 운영하는 최고의 기술력을 가진 은행입니다. 
> 토스 뱅크의 글을 읽고 추측으로 작성된 글이며, 실제로 내부에서 Lock이나 다른 매커니즘을 추가적으로 사용할 가능성이 높습니다.

## 아티클을 요약하자면

**[이 아티클](https://toss.tech/article/34481)을 먼저 읽고, 저의 블로그를 읽으시길 강력히 권장합니다.**

약관 데이터 서빙을 leader DB(아마 RDBMS겠죠?)에서만 수행하고 있었습니다. 이는 '강한 일관성;'이라고 하는 속성을 지키기 위함이라고 합니다.
요청이 점점 늘어나감에 따라서 단일 DB 인스턴스로는 성능 한계에 도달하게 됩니다.
따라서 빠르게 읽기를 할 수 있는 캐시 레이어를 도입합니다. 이때 redis를 캐시 저장소로 선택하고,
write시에 cache evict, read시에 cache를 채우는 방식으로 구현을 합니다.
일관성이 매우 중요하기 때문에 Kafka 이벤트 발행 순서도 조절하고, redis에 대하여 서킷도 달았습니다.

write시에 cache evict하는 타이밍은 db에 commit이 된 다음 application server에서 redis unlink를 수행하고, 성공적으로 redis unlink가 수행되면 API의 응답을 주게 됩니다.

![토스뱅크 캐시 이미지](https://static.toss.im/ipd-tcs/toss_core/live/cb4390cc-0d43-4d98-998f-ff21a1d7102d/image.png)

## 일관성 모델

원 블로그에서는 처음에 강한 일관성(Strong Consistency, 혹은 Linearizability)을 의도하면서 구현을 이어나가다가 후반부에는 비교적 약한 일관성을 보장하기로 결정합니다.

| "약관 동의 또는 철회 요청 API 처리가 완료된 순간, 바로 다음 요청에 DB에 저장된 값이 응답되어야 한다"

이게 우리의 요구사항입니다. API의 처리가 완료되었다면 그 다음 요청에서 DB의 값이 응답된다. 정확히 어떠한 속성인지 해석의 여지가 있으니 조금 더 너그러운 일관성 모델로 생각해보겠습니다. 바로 Read Your Writes입니다. 

**Read-Your-Writes**: 사용자가 성공적으로 쓰기를 완료한 후, 그 사용자의 다음 읽기 요청은 반드시 그 쓰기 내용을 포함한 최신 값을 반환해야 합니다.

RYW(Read Your Writes)의 경우에 세션의 일관성 모델인데, 세션의 인과관계가 역전되는 일을 방지합니다. 물론 동일 세션이 아닌 다른 세션을 기준으로도 역전이 일어나게 됩니다.

Monotonic Read(No-Flickering)과 같은 속성은 지금은 무시하도록합니다.

## 반례를 발견하는 것의 어려움

분산 시스템의 안전성, 혹은 특정 속성을 보장하지 않는 다는 것은 비교적 쉽게 찾을 수 있습니다. 단 한가지 반례를 찾으면 우리의 모델이 틀렸다는 것을 검증할 수 있습니다. 하지만 반례가 없다는 것을 증명하는 것은 조금 더 까다롭습니다. 모든 조합이 아주 적은 수의 조건에 따라서 지수적으로 증가합니다. 동시에 실행되는 프로세스가 2개로 늘어나면 상황은 4가지가 됩니다. 프로세스가 3개가 되면 더욱 많이 늘어나겠지요. 게다가 끝없이 실행되는 프로그램이라면 어떨까요? 아주 우연히 특정 순서대로 프로그램이 수행된다면, 우리는 그 상황을 검토할 수 있을까요? 아마 불가능할겁니다. 개발자의 두뇌는 그렇게 동작하지 않습니다.

그렇기에 반례를 찾는 상황은 온전히 개발자의 상상력과, 경험에 의존합니다. 이건 문제가 있을 것 같다는 직관, 비슷한 문제를 풀어보았던 경험이 많이 작용하죠.
호기심이 많은 개발자가 이 상황, 저 상황 탐색하면서 문제를 찾을 수도 있습니다. 또, 분산 시스템에 익숙한 사람이라면 특정 정합성을 만족하기 위한 최소 조건을 증명하고, 이 시스템이 충분조건을 만족하고 있다는 것을 증명할 수도 있습니다.

저는 이런 시스템에 관심이 많기에 글을 읽자마다 반례가 바로 떠올랐습니다. 또, 분산 시스템에 익숙한 사람이라면 특정 정합성을 만족하기 위한 필요조건 or 최소 조건이 무엇인지 감으로 알고 있기에, 블로그에 기술된 내용만으로는 시스템이 조건을 충족하지 못한다는 것을 빠르게 파악할 수 있습니다. 물론 아주 익숙한 사람이더라도 시간을 들여 천천히 생각하지 않으면 문제를 놓칠 수 있습니다.

## TLA+ Spec으로 모델 정의하여 검증하기

저는 이번 검증에서 TLA+를 사용하겠습니다. 언급했다시피 단순한 반례 몇가지를 찾는 것은 매우 쉽지만, 개개인의 상상력에 의존합니다.
개발자 한명이 생각할 수 있는 시나리오는 고작해야 한번에 20가지가 전부이고, 수천만건이 있는 상황을 빠짐없이 시뮬레이션하는 것은 불가능하죠. 이는 컴퓨터가 잘 하는 것입니다. 
어떤 컴포넌트의 문제가 있다는 것을 잘 드러낼 수 있는 방법인 Formal Method를 사용하여 모델링을 시작해보겠습니다.

아주 단순한 user -> server -> db & redis 모델입니다. 각 컴포넌트 간의 통신은 network로 통신하며 순서가 바뀔 수 있습니다. 물론 tcp 기반으로 생각하고있기에 커넥션 단위의 메시지 순서가 바뀌지 않습니다. 네트워크는 유실도 일어나지 않습니다. 현실세계에 비하면 매우 너그러운 조건이죠.

단순한 쓰기 & 읽기 시나리오입니다.

![읽기 시나리오](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/read-seq.svg?raw=true)
![쓰기 시나리오](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/write-seq.svg?raw=true)

의사 코드는 아래와 같습니다.
```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)
    redis.set(userID, dbResult)

    return dbResult
}

func write(userID, newValue string) string {
    db.openTransaction()
    db.update(userID, newValue)
    db.commit()

    redis.unlink(userID) // wait for unlink done
    return newValue
}
```

꽤나 간단하죠? Golang으로 작성되었지만 kotlin, spring도 비슷하게 동작할겁니다.

대부분의 어플리케이션에서는 유저가 한번에 단 하나의 호출만 하는 경우는 드뭅니다. 속도와 성능을 위하여 여러 호출이 동시에 진행되죠.
또한 서버가 여러개 존재하는 MSA로 구성되어있다면 요청 하나가 동일한 TermsServer를 여러번 호출하는 경우도 빈번할 것입니다.
내부적으로 요청이 퍼지면서 거의 동일한 시점에 여러 요청이 동시에 처리될 수 있죠.

![요청 스프레드](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/req-spread.svg?raw=true)

강한 일관성이라면 이러한 문제에서 자유로워야합니다. 하지만 우리는 그정도로 원하지 않으니 단 하나의 속성만 체크하겠습니다.
동일한 유저의 요청 스레드라면 자신이 write에 성공한 이후에 새로 시작하는 read에서는 write가 반영되어야한다는 것입니다.

```text
ReadYourWriteConsistency ==
    \A u \in Users:
        clientView[u] # NoneVal =>
            clientView[u].version >= lastWriteVersion[u]
```

TLA+로 위처럼 모델링할 수 있습니다. client는 마지막에 write가 성공한 값이 그 다음 read의 결과로 나와야 한다는 것입니다. 참고로 client는 요청 하나씩만 처리할 수 있는 유저의 커넥션을 의미합니다. 이전 요청이 끝나지 않으면 다음 요청이 시작될 수 없습니다.

여기서 Users는 사실 “세션(session)”을 의미합니다. 하나의 실제 유저는 여러 세션(웹/앱, 여러 탭 등)을 가질 수 있지만, 이 글에서 정의하는 Read-Your-Writes는 “각 세션 기준 RYW”입니다.

## 반례

![반례 시나리오 1](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/1-violence.svg?raw=true)

이렇게 단순한 모델, 그리고 네트워크 지연만 있는 경우에도 문제는 발생합니다. 문제가 되는 시퀀스를 하나 보여드리겠습니다. Set이 지연해서 도착하면서 unlink가 되었지만 기존값으로 캐시가 채워진 모습입니다. 단일 요청 흐름에서 보더라도 동시에 2가지 요청이 수행되는 경우 실제로 write가 성공했지만 이전값이 나온다는 점입니다. 자신의 쓰기가 무시된 상태죠.

이 시나리오에서 토스뱅크가 주장하는 "약관 동의 여부는 약관 동의 또는 철회 요청 API 처리가 완료된 순간, 바로 다음 요청에 DB에 저장된 값이 응답되어야 한다."에 정면으로 위배되는 케이스이죠.  Write 요청이 다시 발생하거나, TTL이 지나지 않는다면 계속해서 이전값을 보게 됩니다. 심지어 Write 요청이 DB에 commit되었고, 성공으로 응답이 내려온 상태죠. DB에 저장된 값이 아닌, 자신이 쓴 값도 내려오지 않는겁니다.

네트워크 지연을 가정하는게 합리적이지 않을 수도 있습니다. 아주 적을 확률일 수 있습니다. 하지만 실제로 발생할 수 있는 경우입니다. network rtt, gc 지연등을 짧게 잡더라도 분산시스템간의 atomic하지 않은 순간에 대하여 끼어들 수 있는 상황은 매일 생깁니다. 아주 조금의 순간이더라도 요청마다 계속해서 수행되다보면 언젠가 발생하는 경우도 있을겁니다.

발생하지 않게 만드려면 지금의 구조로는 부족합니다. 추가적인 매커니즘이 필요해요.

### 서킷브레이커는 도움이 되지 않는다

그리고 그 메커니즘은 분명히 서킷브레이커가 아닙니다. 저는 서킷브레이커를 아예 모델에 넣지도 않았는데요, 이는 redis 호출이 항상 성공하기 때문입니다. 위의 반례의 경우 unlink가 항상 성공하고있습니다.
서킷 브레이커가 열리지 않고도 정합성 문제가 몇분이상 지속될 수 있다는걸 볼 수 있죠. 애초에 서킷브레이커는 문제 발생시의 차단 역할입니다. 근본적인 시스템의 일관성 위반을 막아주지 않습니다.

물론 서킷이 있다는 것은 실용적인 접근이고, 저는 서킷을 좋아합니다. 장애 차단 관점에서 얼마나 중요한지 잘 알고있습니다 ;)

### Unlink를 DB Transaction안에서 수행하는건 도움이 되지 않는다

몇가지 변형을 가하면, 단순한 수정으로 이런 일관성 속성을 개선할 수 있을까요? 지금은 AFTER_COMMIT으로 DB에 커밋한 다음 redis에서 unlink를 수행합니다. 만약 Transaction commit 전에 unlink를 한다면 어떻게 될까요?

여전히 동일한 문제는 남습니다. 아래 시나리오로 확인할 수 있습니다.

![반례 시나리오 2](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/2-violence.svg?raw=true)

**(1)DB Commit -> Unlink와 (2)Read Path에서의 DB Read->Set간의 순서 불일치**가 현상의 원인입니다. 1번과 2번이 각각 Atomic하지 않기에 DB에서의 순서는 존재하지만 Redis에서의 순서는 DB의 순서와 달라지기 때문입니다.

혹시 너무나 마법적인 일이 일어나서 DB에 commit하는 순간 redis도 unlink가 일어난다면 어떻게 될까요? 동일하게 TLA+로 모델링해보겠습니다.

![반례 시나리오 3](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/3-violence.svg?raw=true)

그래도 여전히 동일한 문제가 남습니다. 이는 순서가 바뀌었다는걸 모르기 때문입니다. 1번이 Atomic하더라도 2번이 Atomic하지 않으면, 혹은 2번의 순서가 바뀌어서 수행될 수 있다면 문제가 존재합니다.

### redis SET NX를 사용해도 도움이 되지 않는다

혹시 Read시에 보통 SET NX를 사용하면 값이 이미 있는 경우 무시하는 속성을 사용해서 해결할 수 있을까요? 종종 SET NX를 사용하는게 좋다는 이야기를 어디선가 들었던 기억도 납니다. 하지만 실제로는 도움이 되지 않습니다. 이전 시나리오만 보더라도 SET NX로 바뀐들 해결되지 않습니다. 위에서 나온 반례 시나리오 1, 2, 3을 참고하세요. **결국 DB의 순서와 redis의 순서가 맞춰져야합니다.**

### Write에서도 Cache를 채우도록 해도 도움이 되지 않는다

Read할 때 캐시를 채우는 경우 여러가지 방법을 넣어도 크게 도움이 되지 않네요. 계속해서 읽을 때 캐시를 이전값으로 채우는게 문제라면, 쓰기시점에 제일 최신값으로 업데이트할 수 있을겁니다. Read가 아니라 Write에서 SET을 수행하는겁니다. 이제 조금 더 근본적인 시스템의 재설계가 수행됩니다.

Write시점에 Redis에 값을 넣는 타이밍을 정해야합니다. 이건 단 한가지 방법밖에 없습니다. DB에 commit이 되지 않은 데이터를 set을 할 수는 없으니 commit된 이후에 SET을 수행하도록합시다. commit되지 않은 값을 set해서 최신값이 미리 보이는 것이 RYW 일관성에서는 문제는 아닐 수 있습니다. 하지만 모종의 이유로 Transaction Commit이 실패한다면 DB와 redis의 상태가 깨지게 됩니다. redis는 잘못된 값이 저장되고, 순간의 상황에서 그 잘못된 값이 서빙되겠죠.

Read시점에 Redis에 값을 채우는 매커니즘도 단 한가지만 존재합니다. 바로 SET NX이죠. Write시점에 채운 캐시를 Read시점에 더 이전값으로 채울 수 있기 때문이죠. 따라서 이번 케이스에서는 Read에서 SET NX를 사용하면서 Write Path에서 SET NX를 사용하지 않는 시나리오를 검증하겠습니다.

의사 코드는 아래와 같습니다.
```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)
    redis.setNx(userID, dbResult)

    return dbResult
}

func write(userID, newValue string) string {
    db.openTransaction()
    db.update(userID, newValue)
    db.commit()

    redis.set(userID, newValue)
    return newValue
}
```

| 혹시 글을 읽은 과정에서 순간적으로 "이것도 안되겠네. 이런 경우도 있잖아"하면서 구체적인 시나리오가 생각나셨나요?

그러나 여전히 동일한 문제가 존재합니다. 아래 시나리오를 한번 보시죠.

![반례 시나리오 4](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/4-violence.svg?raw=true)

직전 시나리오를 보니까 Write가 동시에 처리되는 경우 덮어씌우는게 문제로 보입니다. SET NX로 위 케이스는 해소할 수 있을 것 같습니다. Write시에도 SET NX를 사용하면 어떻게 될까요?

```go
func write(userID, newValue string) string {
    db.openTransaction()
    db.update(userID, newValue)
    db.commit()

    redis.setNx(userID, newValue)
    return newValue
}
```

![반례 시나리오 5](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/5-violence.svg?raw=true)

결국 Write요청이 동시에 오게된다면 DB가 Serializable하더라도 Redis에 저장되는 값은 DB와 다른 순서를 가지게 됩니다. 

### Write에서만 Cache를 채우도록 해도 도움이 되지 않는다

Read에서 채우지않고, Write시에만 채우게하면 문제가 없을까요?

```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)
    // redis.setNx(userID, dbResult) cache를 채우지 않는다

    return dbResult
}
```

아마 여러분들이라면 1초만에 답하실 수 있을겁니다. 직전 케이스들 모두 Write 충돌로 인한 문제였습니다. Read는 관여하지 않았죠. **핵심은 여전히 DB의 순서와 Redis의 순서의 불일치입니다.**

지금까지 몇가지 생각나는, 흔히 시도하는 방법들에 대하여 시도해보았지만 문제가 사라지지 않았습니다. 이제는 그만 탐색하고 해결책을 이야기해보겠습니다.

## RYW 일관성을 만족시키는 방법

문제는 DB Commit과 Redis SET이 원자적이지 않다는 것입니다. 이를 원자적으로 만들면 문제가 해결됩니다.

### Locks will save us

Lock은 분산 시스템에서 아주 강력한 도구입니다. Lock이 우리를 구해주는 것은 자명합니다. 강력한만큼 부작용도 있습니다. 꽤나 오버헤드가 크다는 문제가 존재하죠.

최소한으로 Lock을 잡기 위하여 쓰기시점에만 Lock을 잡아서 일관성을 맞출 수 있습니다. 약관의 경우 Read Heavy한 패턴이기에 합리적인 방법입니다. 이 방법의 경우 Read에서 SET NX를 쓰고, Write시에 Lock으로 DB와 Redis의 순서를 엄밀히 보장하면 "Read Your Writes"는 보장합니다.

```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)
    redis.setNx(userID, dbResult)

    return dbResult
}

func write(userID, newValue string) string {
    l := distLock.lock(userID)
    defer distLock.unlock(l) // 이 함수의 결과가 반환될때 release

    db.openTransaction()
    db.update(userID, newValue)
    db.commit()

    redis.set(userID, newValue)
    return newValue
}
```

TLA+로 검증한 결과, 160만가지의 상황을 시뮬레이션하였고 위반하는 경우가 없다는 것을 확인하였습니다. read시점에 이전값이 저장될 수 있지만, write시점에 항상 최신값으로 순서대로 반영되기에 정합성 문제에서 승리하게 되었습니다.

Write에만 Lock을 잡는 것은 Monotonic Read를 만족할 수는 없지만, 중요한 속성은 아니기에 후술하겠습니다.

### Or just use Versioned Conditional Set Mechanism

Lock만 우리를 구원해줄까요? 다른 방법은 없을까요? 우리에게는 다른 방법도 존재합니다. 바로 Versioned Conditional Set(혹은 Last-Write-Wins with Version, LWW-V)라고 불리는 방법이죠. 조금 더 약한 보장만 필요한 경우에 자주 사용되는 방법이죠. DB <-> Redis의 순서를 완전히 동일하게 맞추는 것이 아닌 **각 컴포넌트의 시간의 흐름이 거꾸로 가는 것을 막는** 방법입니다. 각 값에 version을 부여하고, Redis는 오직 더 높은 version의 값만 받아들입니다. 이는 MVCC(Multi-Version Concurrency Control)의 단순화된 형태로도 볼 수 있습니다.

보통의 경우 redis에서 lua/functions를 이용하여 값의 version을 확인해서 이전 버전에 대한 SET (NX)를 무시하는 것입니다. 이것도 Monotonic Read는 만족하지 못하지만, "Read Your Writes"를 만족하게 됩니다.

VCS가 RYW를 만족시키는 이유는 Redis의 버전이 단조 증가하기 때문입니다. 
Write 시점에 Redis의 logical clock을 해당 버전까지 끌어올리고, VCS는 이 clock이 후퇴하지 않음을 보장합니다.
따라서 Write 완료 후의 모든 Read는 최소한 그 버전 이상을 보게 됩니다.

### But redis have a TTL, and it's more complicated

보통의 경우 redis는 모든 데이터를 memory에 올려서 동작합니다. 메모리는 비교적 희소한 자원이기에 DB의 모든 데이터를 항상 메모리에 올려두고있는 것은 바람직하지 않습니다.
또한 redis의 역할을 캐시 레이어이기 때문에 자주 조회되는 데이터에 대하여 저장하면 되죠. 따라서 TTL을 걸어두고 사용하게 됩니다.
특정 시간이 지나면 그 데이터는 메모리에서 내려가고 전체 데이터중에 일부만 저장하고 잇게 됩니다. 캐시는 HitRate이 매우 중요한데 적절하게 TTL이나 캐시 만료 정책을 세워서 운영해야합니다. 오늘은 캐시 Expire 전략을 이야기하는 시간은 아니기에 이정도로 줄입니다.

중요한 것은 redis에 저장된 데이터는 항상 남아있는게 아니라 종종 사라진다는 것입니다. 실제로 발생하는 케이스인데도 우리의 모델에는 그게 없죠.
TTL대신에 redis에 있는 데이터 일부가 random하게 사라지는 경우를 모델링하겠습니다.

```text
RedisTTLExpired ==
    /\ redisValue' = NoneVal
    /\ UNCHANGED << dbValue, serverState, clientView, clientState, clientRequestType, lastWriteVersion, msgs >>

Next ==
    ...
    \/ RedisTTLExpired
```

이 경우에 우리의 마지막 모델인 Write VCS, Read SetNX는 RYW 일관성을 위반하게 됩니다.

![반례 시나리오 6](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/6-violence.svg?raw=true)

redis ttl이 있다고 하더라도 이렇게 순간적으로 없어지는 경우는 드물거에요. 그러나 redis의 메모리가 올라가면서 TTL이 지나지 않은 KEY들을 evict해버릴 수도 있습니다. 혹은 관리자가 캐시를 지우는 상황을 테스트하기 위하여 자신의 redis key를 unlink할 수도 있겠죠. 무슨일이든 redis cache가 만료될 가능성이 있다면 발생할 수 있는 케이스입니다.

### 혹시 Read시에도 VCS를 쓰면 괜찮을까요?

```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)
    redis.VCS(userID, dbResult)

    return dbResult
}

func write(userID, newValue string) string {
    db.openTransaction()
    db.update(userID, newValue)
    db.commit()

    redis.VCS(userID, newValue)

    return newValue
}
```

![반례 시나리오 7](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/7-violence.svg?raw=true)

### 혹시 Write시점에 Lock을 잡은 모델이면 괜찮을까요?

아뇨, 괜찮지 않습니다. 반례 시나리오는 다음과 같습니다.

![반례 시나리오 8](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/8-violence.svg?raw=true)

### Read&Write에 하나의 Lock을 잡으면 해결될까요?

유저별로 강한 일관성을 가져갈 수도 있는 방법이죠. 한번에 Read 혹은 Write 요청 하나만 수행되는겁니다. 아주 강력한 방법이기에 이걸 시도해보죠.

```go
func read(userID string) string {
    l := distLock.lock(userID)
    defer distLock.unlock(l) // 이 함수의 결과가 반환될때 release

    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)
    redis.VCS(userID, dbResult)
    return dbdbResultResule
}
```

이 상황에서 동시성이라는건 없습니다. TLA+로 300만개의 유니크한 경우의 수를 전부 시뮬레이션해보아도 일관성이 깨지는 현상을 발견하지 못합니다.

하지만 모든 Read & Write 요청에서 Lock을 잡는건 비효율적입니다. Read Heavy한 트래픽이기도 하지만, Redis 정합성을 위하여 Redis Cache Hit의 경우에도 Lock을 잡기 때문이죠. 그러면 이렇게 Redis에 값을 넣는 시점에만 Lock을 잡으면 어떨까요? 아래 의사코드를 작성했습니다.

```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    dbResult := db.findByID(userID)

    l := distLock.lock(userID) // cache miss시에 redis에 넣기전 lock
    redis.VCS(userID, dbResult)
    distLock.unlock(l)

    return dbResult
}
```

이 경우는 아쉽게도 커버하지 못하는 경우가 생겼습니다. 하지만 좋은 접근이에요.

![반례 시나리오 9](https://github.com/MagicalLas/MagicalLas.github.io/blob/master/_screenshots/9-violence.svg?raw=true)

그러면 Cache miss된 시점, DB Read 이전에 Lock을 잡으면 어떨까요?

```go
func read(userID string) string {
    result := redis.get(userID)
    if result != nil {
        return result
    }

    l := distLock.lock(userID)

    dbResult := db.findByID(userID)
    redis.VCS(userID, dbResult)

    distLock.unlock(l)

    return dbResult
}
```

이렇게 작성한다면 RYW 일관성을 지킬 수 있습니다. 매우 많은 경우와 구현이 있고 복잡하지 않나요?

| 자신이 쓴걸 그 다음 요청에서 읽을 수 있게 만드는 것은 매우 세심하게 설계되어야합니다.

### 정말 이런 상황이 발생할 수 있는가?

분산 시스템의 edge VCSe를 이야기하면 항상 따라오는 질문입니다.

- "그게 정말 일어나?"
- "확률이 얼마나 되는데?"
- "우리는 안 일어날 것 같은데?"
- 등등 많은 질문들...

아마도 거의 일어나지 않을거에요. Network 지연이 그렇게 크지 않을거고, STW 시간이 몇분씩 걸리지도 않겠죠.
갑자기 ACK, RST도 주지 않고 네트워크가 죽어버리는 경우는 어떤 제품은 한번도 겪지 않을 수도 있습니다.
그러나 발생하지 않는 것은 아닙니다. 모든 것은 확률과 리스크에 달렸죠.

간단한 산술로 대략적인 규모를 살펴볼까요? 잘못되었을 가능성이 매우 높은 가정들이기에 실제와 다릅니다.

가정:
- 읽기 요청: 10,000건/초
- 쓰기 요청: 10건/초 (피크)
- 동시 접속 유저: 5,000명
- 쓰기 duration: 200ms (GC는 10ms + 네트워크 RTT 2ms로 가정시)

**동시 진행 중인 쓰기:**
$$10 \times 0.2 = 2\text{건}$$

**동일 유저에게 요청이 겹칠 확률:**
$$P(\text{충돌}) = \frac{2}{5,000} = 0.04\%$$

**하루 기준 충돌 횟수:**
- 읽기-쓰기 충돌: $10,000 \times 0.0004 \times 86,400 \approx 345,600\text{건}$
- 쓰기-쓰기 충돌: $10 \times 0.0004 \times 86,400 \approx 346\text{건}$

0.04%는 무시해도 될 것 같은 수치입니다. 쓰기-쓰기 충돌도 되게 적게 발생합니다. 아마 실제로는 더 적을 수도 있어요. 혹은 더 많을 수도 있습니다. 위에서 이야기한 요청 하나에서 여러 요청이 파생되는 경우는 거의 동시에 요청이 들어올 수도 있죠. 발생하지 않을 수도 있어요. 하지만 시간이 지나면서 계속 수행하다보면 언젠가 발생할 수 있습니다.

### 우리가 가정하는 것들

저는 이렇게 이야기하고싶습니다. 우리는 어떠한 가정에 기대고 있는지를 아는게 매우 중요합니다. 네트워크는 이렇고, STW는 저렇고, TTL은 어떻고, DB는 어떻게 동작하고 등등... 그리고 그 가정을 어느정도는 알고있지 못하면 가정이 깨졌을 때 문제를 식별하기 힘들어집니다. 혹은 그 가정을 스스로 깨트릴 수도 있습니다.

실제 우리 시스템이나 구현을 전부 모델링해서 검증하는게 아니라면 완벽히 안전하다는걸 검증할 수 없습니다. 하지만 우리의 시스템의 논리적인 버그를 잡는데 도와줍니다.

결국 어디까지 리스크를 견딜것인지가 중요합니다. 그 관점에서 트레이드오프를 하는것이죠. 질문은 이겁니다. **"이 가정이 깨졌을 때 우리 시스템은 어떻게 동작하는가?"** 대답이 "모르겠다"라면, 그건 리스크를 관리하는 게 아니라 운에 맡기는 겁니다.

## 일관성 모델을 정의하고 지켜나가는 방법

일관성을 보장하는 건 공짜가 아닙니다. 분산 락은 latency를 추가하고, VCS는 구현 복잡도를 높이며, TTL을 제거하면 메모리 비용이 증가합니다.
어떤 선택이 맞는지는 비즈니스 요구사항에 달려있습니다.

중요한 건 어떤 일관성 모델을 선택했는지 명확히 알고, 그에 맞는 정확한 구현을 하는 것입니다. 일관성 모델을 잘 지키기는게 중요하다면 그걸 중요하게 모델링하세요.

| Hope is not a strategy.

제가 좋아하는 말을 하나 인용해볼게요. 희망은 전략이 아닙니다. 어떤 일관성 모델을 만족할거라고 희망, 기대하지 말고 전략을 세워서 접근해야합니다.

## 마무리

분산시스템은 직관으로 이해하기 어렵습니다. 개발자의 상상력과 사고력은 한계가 있고, 유명한 오픈소스들도 오류를 범하기도합니다.
자신있게 안전하다고 생각한 속성도 실제로 지켜지지 못할 수도 있습니다.

추상적인 모델을 검증할 수 있는 Formal Method를 사용해서 캐시 시스템의 일관성을 모델링하고 검증해보았습니다. 
작은 노력으로 사람이 상상할 수 없는 조합을 검증하면서 우리의 시스템을 더 잘 이해할 수 있습니다.
단순 Look-aside 방식에서 SET, SET NX, Lock등의 방법들을 쉽게 전환하면서 검증하며 시스템을 견고하게 디자인해나갈 수 있습니다.

앞으로도 안전한 시스템 디자인하시길 바랍니다. 혹시 안전한, 정확한 시스템을 구현하기 위한 이야기를 나누고싶으신 분이 있다면 편히 메일주세요!
