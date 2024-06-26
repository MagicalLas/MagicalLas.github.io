---
layout: post
title: "Java의 pointer와 memory allocation"
slug: jvm-pointer
category: jvm
---

```md
> 메모리 해제 누락과 같은 실수를 줄이는 가장 좋은 방법은 뭘까? 오답노트 만들기? 시행착오 거치기? 각자 생각이 다르겠지만, 내 생각은 처음부터 실수할 일이 없는 도구나 환경을 만드는 것이다. 개인의 의지나 역량에 의존하는 것만으로는 실수를 줄이기 어렵다.
>
> 그런 도구 중 가장 잘 알려진 것은 Java의 가비지 콜렉션이 아닐까? Java는 포인터를 없애 버리고, 개발자가 메모리 해제를 신경쓰지 않아도 되게 했다. 그 결과 모든 개체를 힙에 생성/해제하는 비용은 있을지언정, 메모리 누수 같은 짜치는(..) 실수로 다른 기능들을 개발하고 개선할 시간이 낭비될 가능성을 줄였다.

ref: https://je-continue.tistory.com/51
```

꽁띠뉘에님의 글을 읽다가 "Java는 포인터를 없애 버리고"라는 부분이 눈에 띄였습니다. 제가 갖고있던 Java의 모습과는 다소 다른 표현이어서 조금 더 깊게 알아보고 싶었습니다. 따라서 이 부분에 대한 저의 생각과 이해를 공유해보려고 합니다. 위 글의 내용이 틀렸다고 주장하기보다는 조금 부가적인 설명을 하고자하는 목적입니다.

> ps. 개인적으로 꽁띠뉘에님의 글이 매우 잘 작성되었으며, 깊이있고 정확한 내용을 담고있음을 명확히 하고싶습니다. 한두문장으로 트집을 잡는 것 같아 조금 마음이 불편하네요.

(아래 내용은 조금 학술적인 톤을 유지하려고 노력했습니다.)

## 포인터란?

아마 이 글을 읽는 사람이라면 포인터에 대하여 어느정도 지식이 있다고 생각한다. 이 글에서 pointer는 C의 pointer를 기준으로 이야기를 해보고자한다. C의 pointer는 특정한 메모리 주소를 가르키는 값의 타입으로 보통 32비트 혹은 64비트의 unsigned integer의 값을 가진다. 이는 메모리 공간의 주소를 가르키기때문인데, 정확히는 실행되는 프로그램의 메모리 공간이라면 어디든 의미할 수 있다.

ok, 여기까지는 간단하고 명료하다. 하지만 어려워지는 것은 여기부터이다. C나 다른 언어의 경우 대부분 자신이 가르키고있는 특정한 메모리 위치에 어떠한 구조로 위치해있는지 명시하고있다. 예를들어, int* a의 경우에는 a가 가르키는 주소에 int의 메모리 구조로 할당되어있을 것을 나타낸다. 만약 SomeStrcut* b의 경우에는 b가 가르키는 주소에 SomeStruct의 메모리 구조로 할당되어있다고 예측할 수 있다. 하지만 이는 매번 지켜지는 것은 아니고, 컴파일 시점에 체크하거나 체크하지 않으면 프로그래머를 믿고 동작하게 된다. 특히 void*를 사용하거나 이러한 포인터를 형변환하는 경우에 이러한 점들이 잘 나타난다.

대부분의 자료형들은 기본값이 존재한다. int는 0, bool은 false, (C언어는 아니지만) string은 "", 어레이는 []가 대표적이다. 이러한 기본값은 특정 변수를 할당하는 과정에서 바로 할당을 하지 않는다면 설정하는 값으로, 여러가지 실용적인 이유와 이론적인 이유로 동작한다. 포인터도 마찬가지로 특정한 메모리 주소를 가르키는 자료형이기에 기본값이 필요했다. 우리는 이 값을 너무나 잘 알고있는데 바로 **null**이다.

```
ps. C의 경우에 변수를 선언한 경우 자동으로 값이 초기화되지 않는다. 초기화되지 않은 변수에 대한 접근은 Undefined Behavior이다.
```

null pointer exception으로 아주 잘 알려진 이 포인터의 초기값은 포인터가 아무것도 가르키고있지 않다는 것을 의미한다. 아무것도 가르키고있지 않은데, 그곳에 접근하려고하니 에러가 발생하는 것이다.

## Java의 포인터

위에서 이미 힌트가 나왔는데, Java에는 포인터라는 개념이 존재한다. Java의 에러 타입에 이미 `NullPointerExceptions`가 존재하기 때문에 포인터라는 개념이 존재하는 것은 부정할 수 없다. 하지만 좀 더 자세히 살펴보자.

C는 값(value) 기반으로 동작하는 언어로, 포인터도 primitive type 중 하나로 취급된다. Java는 조금 다르게 동작하는데, primitive type(예: int, boolean)들과 객체 참조가 존재한다. 지금은 객체 참조를 C의 포인터와 비슷한 무언가로 이해해도 된다. Java의 경우 대부분의 객체의 변수는 객체의 참조를 나타내는데, 이는 특정 메모리 주소를 가르키는 값이다. 그리고 C와 유사하게도 가르키고있는 메모리 주소가 어떠한 구조로 할당되어있는지에 대한 정보를 담고있다.

Java의 객체 참조와 C의 포인터의 주요 차이점은 Java에서는 저수준(low-level) 연산들을 지원하지 않는다는 것이다. 특정 객체 참조에 대하여 +와 같은 연산을 시도하면 컴파일 과정에서 문제가 발생한다. 따라서 이러한 부분은 실제 포인터를 사용해서 구현되어있더라도 연산을 제한하는 문법을 퉁하여 여러가지 문제점들을 보완하고있다.

하지만 Java도 모든 연산을 엄밀하게 검증하는 것은 아니다. 예를들어 Object와 같은 타입은 모든 객체의 sub type인데, A type <-> Object <-> B type과 같은 변환을 컴파일하는 것이 가능하다. Java의 타입 변환은 업캐스팅(자동 변환)과 다운캐스팅(명시적 변환)으로 나뉘는데, 모든 객체는 Object 타입으로 업캐스팅될 수 있지만, 실제 타입을 모르는 상태에서의 다운캐스팅은 오류를 발생시킬 수 있다.

이를 포인터 관점에서 보면, 다운캐스팅의 잘못된 사용은 잘못된 메모리 영역에 접근하려는 것과 유사하다고 볼 수 있다. C나 C++의 포인터 오류와 유사하게, Java에서는 잘못된 다운캐스팅 시 ClassCastException이 발생하게된다. 이런 관점에서 Java의 타입 변환은 "포인터"의 안전성과 직접적인 연관이 있다고 볼 수 있다. 포인터의 잘못된 사용을 방지하고 메모리 관리를 안전하게 유지하기 위해 타입 시스템을 사용하여 안전한 메모리 관리를 보장한다.

이러한 관점에서 보면 Pointer를 직접 노출하지는 않지만 내부적으로는 사용하고있으며 프로그래머입장에서 내부적인 개념을 드러내는 경우에는 Pointer를 보게된다.

## 포인터와 GC의 관계

우리는 위에서 Java는 포인터가 존재하지만, 내부적으로만 사용하며 직접 low한 operation을 수행할 수 없다는 것을 알게되었다. 그렇다면 이러한 포인터를 직접 사용하지 않는 것이 GC에 필수적일까?

GC의 종류에 대해 살펴보면, 보통 tracing gc와 reference count gc로 나눌 수 있다. tracing GC를 사용하는 경우, mutator root로부터 어떤 객체가 참조 가능한지를 추적하는 방식으로 동작한다. 일단 Java의 경우 tracing gc를 많이 사용하기에 tracing gc를 기준으로 이야기해보겠다.

이 과정에서 가장 중요한 것은 pointer에 대한 +-연산으로 참조하는 메모리 공간을 추적하는 방법이다. 이러한 포인터를 파생 포인터(derived pointer)라고 한다. C/C++과 같은 파생 포인터의 경우 GC가 이를 감지하기가 어려운데, 특정 메모리 영역을 참조하여 도달가능한지가 런타임에 정해지기 때문이다. GC는 최대한 보수적으로 동작해야하며, 사용중인 메모리를 할당하지 않는 safty를 보장해야하기 때문에 이런 파생 포인터는 GC의 구현에 큰 어려움을 준다.

Java의 경우도 이는 마찬가지인데, [Garbage Collection and Local Variable Type-Precision and Liveness in JavaTM Virtual Machines](https://dl.acm.org/doi/pdf/10.1145/277650.277738)에서도 다음과 같은 언급이 등장한다.

```md
> The relatively simple and highly constrained model of references presented by the JVM avoids the optimization-induced problems these other works address, such as interior and derived pointers.
>
> JVM이 제공하는 비교적 단순하고 제약이 많은 참조 모델은 내부 및 파생 포인터와 같은 다른 연구들에서 다루는 최적화 관련 문제를 피할 수 있습니다.
```

이러한 점들을 비교해보았을 때 GC 구현을 쉽게하는데 있어서 자바와 같이 pointer 연산에 제약을 주는 것은 매우 효과적이라고 할 수 있다.

GC와 포인터가 함께 존재하는 언어인 Go를 살펴보는 것도 좋다. Go는 포인터가 명시적으로 존재하며 *int와 같은 타입을 실제로 많이 사용하는 언어이다. Java와 같이 pointer에 대하여 연산을 거의 지원하지 않는다. 따라서 대부분의 경우에는 Java와 비슷하게 tracing하는게 크게 문제가 없겠지만, 특수한 예외로 uintptr이라는 타입을 제공한다. 이는 C의 포인터와 유사한데, uintptr은 단순한 integer로 인식되며 GC는 uintptr가 가르키는 포인터를 할당해제할 수도 있다.

```md
> A uintptr is an integer, not a reference. Converting a Pointer to a uintptr creates an integer value with no pointer semantics. Even if a uintptr holds the address of some object, the garbage collector will not update that uintptr's value if the object moves, nor will that uintptr keep the object from being reclaimed.
>
> uintptr은 참조가 아닌 정수입니다. 포인터를 uintptr로 변환하면 포인터 시맨틱이 없는 정수 값이 생성됩니다. 가비지 컬렉터가 어떤 객체의 주소를 보유하고 있더라도 객체가 이동하면 가비지 컬렉터는 해당 객체의 값을 업데이트하지 않으며, 해당 객체가 회수되지 않도록 유지하지도 않습니다.
>
> ref: https://pkg.go.dev/unsafe#Pointer
```

여기서 핵심적인 포인트는 포인터를 직접적으로 노출하는 것이 아니라, 파생 포인터의 존재 없이 객체의 참조 가능성을 명확하게 만드는 것이 GC 구현에 중요하다는 것이다.

## Conclusion

1. Java에는 객체 참조를 구현하기 위하여 내부적으로는 Pointer가 존재하지만, C와 같은 저수준 연산을 지원하지 않는다.
2. 이러한 객체 참조의 특성 상 Java에서는 파생 포인터와 같은 복잡한 연산이 필요하지 않게 되었다.
3. 파생 포인터는 GC를 구현하는데 매우 힘들게 만드는 매커니즘이다.
4. 포인터라는 용어나 개념을 드러내는 것보다는 low-level의 연산을 막는 것이 GC 구현에 더 중요하다

## ps.

저는 CS 전공을 하지 않았으며 이 글의 내용은 동료로부터 엄격한 리뷰를 거치지 않아 잘못된 내용이 존재할 수도 있습니다. 관련해서 정확하지 않은 내용이나 개선할 점이 있다면 언제든 메일(las@magical.dev)로 연락 부탁드립니다. 감사합니다.
