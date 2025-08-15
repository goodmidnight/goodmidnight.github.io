---
title: Android Clean Architecture Error Handling (1) - 클린 아키텍처 예외 구조화 사례
date: 2025-08-15 00:00:00
categories: [ clean-architecture ]
tags: [ android,clean-architecture,development ]
math: true
---

아키텍처란 소프트웨어 설계에 있어 필수적인 영역이며, 어느정도 규모가 있는 Android 프로젝트에서는 클린 아키텍처를 기반으로 확장 개발을 하는 것이 이제 보편적인 선택지가 되었습니다.

`클린 아키텍처`는 여러 개의 계층으로 쌓여있는 아키텍처이며,  **계층의 의존성이 외부에서 내부로 향해야하며** **데이터는 외부 계층에서 내부 계층 그리고 다시 외부 계층으로 흐르는 단방향 흐름을 따라야 한다는 원칙**을 가지고 있습니다.

이러한 구조에서 계층을 넘나들 때마다 데이터는 각 계층에 맞게 레이어의 역할에 맞도록 변환되고 계층의 주체들이 사용하기 수월하도록 가공됩니다. 하지만 **소프트웨어는 데이터뿐만 아니라 예외도 같이 동반합니다. 이러한 예외들을 적절하게 제어하고 처리하는 것은 소프트웨어 개발에 있어 핵심적인 영역입니다.**

단순한 구조에서의 예외 처리는 단순하지만 **클린 아키텍처와 같이 계층적으로 구분된 구조에서는 예외 처리 또한 구조에 맞도록 적절하게 대응하는 방식이 필요합니다.** 이번 포스트에서는 클린 아키텍처 구조에서 예외 처리에 관한 아이디어 및 구현 전략을 제시하겠습니다.

# 베경지식

클린 아키텍처에서 적절하게 예외를 처리하려면 몇 가지의 배경지식들이 요구됩니다. 해당 파트에서 간단하게 클린 아키텍처와 Java/Kotlin에서 사용되는 예외에 대해 다루고 구현을 진행하도록 하겠습니다.

## CleanArchitecture

클린 아키텍처는 로버트 마틴이 제안한 소프트웨어 구조로 **관심사 분리를 통해 유연하고 테스트와 유지보수가 수월한 소프트웨어를 만드는 것을 목표**로 합니다.

일반적인 아키텍처들은 한번 구조가 정해질 경우 추후 새로운 변경사항에 대응하기 어렵다는 단점이 존재합니다.  예를 들어 비즈니스로직이 산재되어 있거나 특정 프레임워크를 중심으로 설계할경우 특정 프레임워크를 교체하거나 데이터의 소스가 바뀔 때 결합을 풀어내고 변경하는 비용이 막대하게 증가하게 됩니다.

클린 아키텍처는 이러한 문제를 해결하도록 관심사 분리와 의존성 규칙이라는 두 가지 개념을 바탕하에 설계되었습니다. **관심사 분리는 코드의 역할에 따라 레이어를 나누는것을 의미하고 의존성 규칙은 레이어의 의존성이 바깥쪽에서 안쪽으로 향해야한다는 것을 의미합니다.**

클린 아키텍처에서는 일반적으로 총 4가지의 역할로 레이어를 분리합니다. 각 레이어 별 역할은 아래와 같습니다.

1. 엔티티: 가장 안쪽에 존재하는 계층으로 애플리케이션의 가장 일반적이고 높은 수준의 비즈니스 규칙들을 담고 있으며, 어떠한 외부 계층이 변경되어도 영향을 받지않는 영역입니다.
2. 유즈케이스: 애플리케이션의 고유 비즈니스 규칙을 포함하는 계층입니다. 엔티티 계층과는 다르게 엔티티를 바탕으로 특정 유즈케이스를 구현하여 시스템에서 사용되어야할 기능들을 정의합니다. 즉 해당 계층은 엔티티 계층을 바탕으로 구성되기 때문에 엔티티 계층이 변경된다면 해당 계층이 변경에 대한 영향을 받을 수 있게됩니다.
3. 인터페이스 어댑터: 앞서 엔티티와 유즈케이스 계층이 비즈니스 규칙을 정의했다면, 인터페이스 어댑터 계층은 내부 계층들이 사용하기 편한 데이터 형식과 외부 프레임워크 및 드라이버 계층이 사용하는 데이터 형식 사이에서 중계하는 역할을 수행합니다. Presenters, Controllers, Gateways등이 해당 계층에 속합니다.
4. 프레임워크 및 드라이버: 가장 바깥쪽에 위치한 계층으로 그 외 나머지 세부 기능들은 해당 계층에서 구현됩니다.

클린 아키텍처의 핵심 규칙과 레이어를 설명하기 위해 아래와 같은 아키텍처 도식도가 흔히 널리 퍼져있으며 해당 사진을 통해 우리가 클린 아키텍처를 어떻게 구성할지에 대한 접근법을 떠올릴 수 있게 됩니다.

<p align="center">
<img src="/assets/post/2025/2025-08-15-architecture-01-01.png" width="80%"/>
</p>

**Android에서는 일반적으로 클린 아키텍처의 모든 레이어를 도입하는 것이 아닌 클린 아키텍처의 원칙을 기반하에 구글이 권장하는 앱 아키텍처를 접목하여 설계합니다.** 이러한 원리를 바탕으로 3-Tier의 아키텍처를 구성하고 추가적인 요구사항에 따라 계층을 확장해 나갑니다. 3-Tier 아키텍처의 각 요소는 아래와 같습니다.

1. UI: UI 관련 로직을 담당하는 계층으로 Activity, Fragment, ViewModel 등이 해당 계층에 속하게 됩니다.
2. Domain: 순수한 비즈니스 로직을 담당하는 계층으로 클린 아키텍처의 Business Rules 계층과 유사합니다.
3. Data: 데이터베이스, 네트워크 등의 데이터 소스를 관리 및 처리하는 계층으로 Repository 구현체 및 API Service, LocalDB등이 해당 계층에 속하게 됩니다.

<p align="center">
<img src="/assets/post/2025/2025-08-15-architecture-01-02.png" width="80%"/>
</p>
## Throwable

Java/Kotlin에서는 소프트웨어에서 발생할 수 있는 예외 상황을 처리하기 위해 `Throwable`이라는 최상위 클래스를 사용합니다.  **Throwable 클래스는 예외 처리에 있어 기본이 되는 클래스로 해당 클래스를 바탕으로 예외를 확장해나갈 수 있으며 try-catch 구문을 처리할 수도 있습니다.**

Throwable에 관한 명세는 [[해당 페이지]](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-throwable/)를 통해 확인하실 수 있습니다.

<p align="center">
<img src="/assets/post/2025/2025-08-15-architecture-01-03.png" width="80%"/>
</p>

`Throwable`은 두 가지의 주요 하위 클래스인 `Exception` 과 `Error`를 나뉘어 상속되는 관계를 유지하고 있습니다. `Exception`은 논리 오류, 외부 요인 또는 개발자 실수에서 발생하는 예외이며 핸들링이 가능하지만,  `Error`의 경우 시스템 자체의 문제로 개발자가 처리하여서는 안되는 예외입니다.

아래에서부터는 해당 두가지의 예외 타입에 대한 내용을 이어나가겠습니다.

## **Exception**

**`Exception`은 앞서 언급하였듯이 개발자가 애플리케이션 코드 상에서 예측할 수 있으면서 복구 가능한 예외입니다.** 구체적으로 작성한 코드의 논리적 결함이나 잘못된 입력 또는 네트워크나 파일 시스템 같이 외부 자원 접근을 통해 발생될 수 있는 것들을 의미합니다. 이러한 유형의 예외들은 개발자가 케이스를 고려하여 적절하게 처리할 수 있습니다.

하나의 예시로 개발을 진행하다보면 흔하게 접하게 되는 NullPointerException, IndexOutOfBoundsException 등은 마찬가지로 Exception을 상속한 클래스이며 이들이 런타임 수행 중간에 나타나 앱의 크래시를 발생시킨 경험이 존재할 것 입니다. 이러한 예외가 런타임 상에 등장하지 않도록 꼼꼼하게 로직을 점검하고 예외가 발생하더라도 적절한 예외처리를 통해 해결하는 것이 중요합니다.

Exception은 다시 Checked Exception과  Unchecked Exception으로 나뉘게 됩니다. Checked Excetpion의 경우 컴파일러가 강제하는 예외로 이에 대한 대응을 처리하지 않을 시, 컴파일 에러를 던지게 됩니다.

**Kotlin은 Java와 다르게 Checked Exception이라는 개념이 존재하지 않습니다. Checked Exception을 사용하는 방식이 오히려 지나친 보일라 플레이트 코드를 유발하고 코드의 복잡성을 높히기 때문입니다.**

물론 일부 Android Framework의 Java코드에서는 여전히 Checked Exception 이 사용되는 것을 아래 사진에서 처럼 확인할 수 있습니다.

<p align="center">
<img src="/assets/post/2025/2025-08-15-architecture-01-04.png" width="80%"/>
</p>

## Error

**`Error`는 애플리케이션 수준에서 복구할 수 없는 시스템 레벨의 심각한 문제를 나타냅니다.** 이는 개발자의 실수로 인해 발생하는 Exception과는 다르게 Error가 발생되었을 때는 더 이상 애플리케이션이 정상적으로 작동할 수 없습니다.

Error를 try-catch를 통해 해결하는 방식은 해당 구문의 코드가 정상적으로 동작할 것이라 보장할 수 없기 때문에 프로그램을 종료시키고 로그를 전달해 근본적인 원인을 해결한 뒤 재배포하는 것이 권장됩니다. Error의 대표적인 예시는 OutOfMemoryError(OOM), StackOverflowError 등이 존재하며, 특히 StackOverflowError는 일반적으로 무한 재귀 호출로 인해 발생합니다.

## 예외처리

`try-catch` 구문은 예외를 처리하는 가장 기본적인 방법입니다. Kotlin에서는 Java와 다르게 **try-catch 구문을 식처럼 사용할 수 있습니다.** 이를 바탕으로 조금 더 실용적이고 간결한 구문을 만들어 낼 수 있습니다.

아래의 코드 예시처럼 try-catch 구문을 직접적으로 함수의 return 값과 연결시킬 수 있다는 것을 확인할 수 있습니다. 추가적으로 눈 여겨볼 것은 when을 통해 예외에 따라 다른 동작을 구현할 수 있다는 점입니다. 지금의 코드는 간단하지만, 추후 여러 유형의 예외를 발생시키는 구문에서는 이를 체계화하여 각각 다른 처리를 할 수 있도록 구현할 수 있습니다.

```kotlin
fun calc(a: Int, b: Int): Int = try {
    a / b
} catch (e: Throwable) {
    when (e) {
        is ArithmeticException -> 0
        else -> 0
    }
}
```

## Result

하지만 이런 try-catch 구문은 함수의 시그니처만 봐서는 해당 함수가 잠재적으로 어떤 예외를 발생시키는지 알 수 없으며, 예외를 핸들링하게 될 경우 try와 catch 블럭을 모두 만들어야하기 때문에 가독성이 떨어진다는 단점이 존재합니다. **Kotlin에서는 이러한 문제점을 해결하기 위해 `Result` 클래스와 그에 대한 익스텐션 함수들을 제공하여 예외를 값 자체로 다루도록 지원하고 있습니다.**

해당 방식은 예외의 발생 가능성을 타입자체에 명시하여 다른 개발자들이 해당 코드를 안전하게 다루도록 유도해주고 예외처리와 그에 따른 체이닝 방식이 편리하기 때문에 선호됩니다.

<p align="center">
<img src="/assets/post/2025/2025-08-15-architecture-01-05.png" width="80%"/>
</p>

예를 들어 아래 코드는 잠재적으로 예외를 던질 수 있는 코드입니다. uri의 포멧이나 위치가 정확하게 일치하지 않을 경우 `FileNotFoundException` 예외를 던질 수 있기 때문입니다. 하지만 이러한 구문을 try-catch로 해결하지 않고 runCatching으로 감싸주어 Result 타입으로 매핑할 수 있게 됩니다.

여기서 사용된 Result 타입은 해당 코드를 사용하는 호출 영역까지 옮겨 그 곳에서 처리하도록 구현할 수 있게 됩니다.

```kotlin
fun convertUriToByteArray(
    context: Context,
    uri: Uri
): Result<ByteArray?> = runCatching {
    context.contentResolver.openInputStream(uri)?.use { it.readBytes() }
}
```

`Result`에 대한 더 자세한 정의 및 익스텐션들은 [[해당 문서]](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-result/)에 존재하니 참고하시길 바랍니다.

## 멀티 모듈

마지막으로 멀티모듈에 대해서 간략하게 알아보겠습니다.

클린 아키텍처의 특성을 가장 잘 적용하기 위해서는 Gradle을 활용한 멀티모듈 기반의 프로젝트로 구현하는것을 권장합니다. **멀티모듈을 사용할 경우 프로젝트의 특정 계층들을 모듈로 분리하여 독립적으로 다룰 수 있고, 의존성을 체계적으로 관리할 수 있기 때문입니다.** 멀티 모듈에 대한 내용은 해당 포스트에서 직접적으로 다루지는 않지만, 중요한 기술이오니 아래 문서들을 참고한다면 도움이 될 것입니다.

- [[Android 앱 모듈화 가이드]](https://developer.android.com/topic/modularization?hl=ko)
- [[일반적인 모듈화 패턴]](https://developer.android.com/topic/modularization/patterns?hl=ko)

# 구현

클린 아키텍처와 예외에 대한 개념들을 다루었으니, 이제 아키텍처에서 예외를 적절한 방식으로 처리할 준비가 되었다는 것입니다. 아래에서부터는 클린 아키텍처에서 수행되는 예외처리 아이디어에 대한 구현 사례를 다루어 보겠습니다.

## 1. 설계

가장 먼저 아키텍처를 설계하여야만 합니다. 프로젝트 아키텍처 구성은 클린 아키텍처와 구글이 권장하는 앱 아키텍처를 결합하여 아키텍처를 크게 data(network), domain, ui로 분류하였고 ui 레이어는 app 모듈에서 처리하도록 구성하였습니다.

멀티 모듈 빌딩을 위해 Gradle 기능을 활용 하였고, 최종적으로 아래 사진과 같이 구성된 모습을 확인하실 수 있습니다.

<p align="center">
<img src="/assets/post/2025/2025-08-15-architecture-01-06.png" width="35%"/>
<img src="/assets/post/2025/2025-08-15-architecture-01-07.png" width="60%"/>
</p>

## 2. 최상위 예외 클래스 구현

여러 계층으로 구성된 아키텍처에서 예외를 적절하게 처리하려면 가장 먼저 고려해야할 부분은 **어떻게 예외를 계층 간 넘겨주어야 하는가** 입니다. 이를 해결하기 위해 **애플리케이션에서 가장 최상위로 정의될 클래스를 정의하고 해당 클래스를 바탕으로 확장해 나가는 방식으로 구현하겠습니다.**

아키텍처에서 다룰 예외는 Error가 아닌 개발자가 처리할 수 있는 Exception을 다룰 것이기 때문에 Exception을 상속한 `ApplicationException` 클래스를 정의해줍니다. 이때 기본적으로 상속되는 생성자를 포함시켜주며, 추가적으로 분류할 `ErrorCode` 인터페이스를 정의해줍니다.

```kotlin
abstract class ApplicationException(
    override val message: String? = null,
    override val cause: Throwable? = null,
    open val errorCode: ErrorCode,
) : Exception(message, cause)
```

여기에 예외의 상세 정보를 담을 `ErrorCode` 인터페이스를 추가합니다. ErrorCode를 인터페이스로 설정한 이유는 `enum`이 해당 인터페이스를 구현하여 다른 유형의 에러 코드 그룹을 다형적으로 처리할 수 있도록 하기 위해서입니다.

```kotlin
interface ErrorCode {
    val code: String
    val message: String?
}
```

ApplicationException에는 단순한 생성자 정의 외에도 유틸리티 기능들을 추가할 수 있습니다. 예를 들어 `toString()` 메서드를 오버라이드하는 기능을 만들어 보겠습니다.

```kotlin
abstract class ApplicationException(
    override val message: String? = null,
    override val cause: Throwable? = null,
    open val errorCode: ErrorCode,
) : Exception(message, cause) {

	 // 예외의 정보를 string으로 출력하는 메서드 
   override fun toString(): String {
		   return "${javaClass.simpleName}(" +
				   "message=${message ?: "null"}, " +
           "cause=${cause?.javaClass?.name ?: "null"}, " +
           "errorCode=${errorCode.code}, " +
           "errorCodeMessage=${errorCode.message}" +
           ")"
    }
}
```

지금까지 최상위 예외 클래스를 구현하였지만, 해당 클래스를 클린 아키텍처의 구조상 과연 어느 위치에 두어야하는가를 고민해야 합니다.

**첫 번째 방식은 최하위 계층인 domain 계층에 두는 것입니다.** 이곳에 정의할 경우 ApplicationException은 애플리케이션 자체의 비즈니스 규칙이 될 것이고, 이를 바탕으로 예외를 확장시켜나갈 수 있기 때문입니다.

**두 번째 방식은 별도의 공통모듈을 두는 방식입니다.** 이는 즉 error라는 이름의 공통 모듈을 두고, 예외 클래스가 요구되는 모듈에서 이를 접근하여 사용하는 것입니다. 이 방식은 다른 공통 모듈에 다른 비즈니스 규칙 부분을 제외하고 순수한 error 관련 의존성만 전달할 수 있지만, 의존성 구조가 복잡해질 수 있다는 단점이 존재합니다.

결국 예외를 비즈니스 규칙의 일부로 볼 것인지 또는 프로젝트의 공통 관심사로 볼 것인지는 프로젝트의 목적에 따라 달라 질 수 있습니다. 이에 따라  실제 프로젝트에서는 설계 구조를 판단하고 더 나은 방법을 채택하는 것이 좋습니다.

물론 두 방식 모두 이번 주제에서 다루는 샘플 구현 사례를 적용하는데에 있어서는 큰 문제가 없기 때문에 Domain 영역에 Exception을 추가하도록하겠습니다.

## 3. 하위 예외 클래스 구현

최상위 예외 클래스를 구현하였다면 이를 바탕으로 계층마다 하위 예외들을 정의할 수 있게 됩니다. 이에 따라 정의한 Data, Domain에 계층에 대하여 `DomainException`, `DataException`을 선언하겠습니다.

가장 먼저 Network 처리를 담당하는 DataException에 대한 예외를 구성해보겠습니다. DataException은 앞서 제작한 ApplicationException을 상속하여 아래와 같은 형태로 구현됩니다.

```kotlin
sealed class DataException(
    override val message: String,
    override val cause: Throwable? = null,
    override val errorCode: ErrorCode
) : ApplicationException(message, cause, errorCode)
```

특이사항은 DataException을 sealed class로 구성하였다는 점입니다. **sealed class를 사용한 이유는 DataException을 바탕으로 상세한 네트워크 예외들을 정의하여 구조화하기 위함입니다.**

이어서 Data에서 사용되는 ErrorCode를 정의하기 위해 `DataErrorCode`를 구현하겠습니다. DataErrorCode는 앞서 구현한 ErrorCode를 구현한 enum class입니다. 각 enum의 요소를 정의할 때 code 필드는 R을 접두어로 사용하도록 하였으며, 각 에러코드에 대한 설명을 message 필드에 담았습니다. 최종적으로 구현된 코드는 아래와 같습니다.

```kotlin
enum class DataErrorCode(
    override val message: String,
    override val code: String,
) : ErrorCode {
    BAD_REQUEST_ERROR(
        message = "[400] 잘못된 요청입니다.",
        code = "R001"
    ),
    NOT_FOUND_ERROR(
        message = "[404] 요청한 리소스를 찾을 수 없습니다.",
        code = "R002"
    ),
    TIMEOUT_ERROR(
        message = "요청시간이 만료되었습니다.",
        code = "R003"
    ),
    
    ...
    
    SERVER_ERROR(
        message = "서버 내부에 오류가 발생하였습니다.",
        code = "R004"
    );
}
```

이제 구현된 DataErrorCode를 바탕으로 DataException을 확장하겠습니다.

```kotlin
sealed class DataException(
    override val message: String,
    override val cause: Throwable? = null,
    override val errorCode: ErrorCode
) : ApplicationException(message, cause, errorCode) {

    data class BadRequestException(
        override val message: String = "Bad request.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DataErrorCode.BAD_REQUEST_ERROR
    ) : DataException(message, cause, errorCode)

    data class NotFoundException(
        override val message: String = "Requested data not found.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DataErrorCode.NOT_FOUND_ERROR
    ) : DataException(message, cause, errorCode)

    data class TimeoutException(
        override val message: String = "Request timed out.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DataErrorCode.TIMEOUT_ERROR
    ) : DataException(message, cause, errorCode)

		...

    data class ServerException(
        val httpCode: Int,
        override val message: String = "Server error.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DataErrorCode.SERVER_ERROR
    ) : DataException(message, cause, errorCode
}
```

이러한 방식으로 요구사항에 따라 추가적인 ErrorCode와 Exception을 정의할 수 있을 것이며 손쉽게 확장이 가능해집니다.

> 현재는 Network 통해 얻어지는 데이터만을 처리하여 단일 data 계층을 사용하였지만, 로컬 DB나 파일 같은 데이터 소스가 추가된다면 그에 따라 계층을 세분화하고 Exception을 추가 정의하는 방식으로 확장할 수 있습니다.
>

---

`DomainExceptio`n의 경우에는 일반적인 비즈니스 규칙에 대한 예외를 정의하여야합니다. 따라서 **구체적인 DataException들을 직접적으로 DomainException에 정의하여 연결하기보다는 조금 더 일반화된 형태로 예외를 나열하겠습니다.**

Domain의 에러코드는 크게 네트워크를 통해 발생하는 코드와 연결시간 초과에 대한 코드로 분리하였으며 이에 따른 Exception을 구현하겠습니다.

```kotlin
enum class DomainErrorCode(
    override val message: String? = null,
    override val code: String,
) : ErrorCode {
		TIMEOUT_ERROR(message = "요청시간이 만료되었습니다.", code = "D001"),
    NETWORK_ERROR(message = "네트워크 에러가 발생하였습니다.", code = "D002"),
    UNKNOWN_ERROR(message = "알 수 없는 에러가 발생하였습니다.", code = "D003");
}
```

```kotlin
sealed class DomainException(
    override val message: String,
    override val cause: Throwable? = null,
    override val errorCode: ErrorCode
) : ApplicationException(message, cause, errorCode) {
    
    //네트워크 예외
    data class NetworkException(
        override val message: String = "Server error.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DomainErrorCode.NETWORK_ERROR
    ): DomainException(message, cause, errorCode)
    
    //타임아웃 예외
    data class TimeoutException( 
        override val message: String = "Request timed out.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DomainErrorCode.TIMEOUT_ERROR
    ) : DomainException(message, cause, errorCode)
    
    //그 외 예외 케이스
    data class UnknownException(
        override val message: String = "An unknown error occurred.",
        override val cause: Throwable? = null,
        override val errorCode: ErrorCode = DomainErrorCode.UNKNOWN_ERROR
    ) : DomainException(message, cause, errorCode)
}
```

이제 DataException와 DomainException을 연결하는 매퍼를 정의해야만 합니다. 매퍼는 아래와 같이 구현할 수 있습니다.

```kotlin
fun DataException.toDomain(): DomainException =
    when (this) {
        is DataException.TimeoutException -> DomainException.TimeoutException(cause = this)
        is DataException.BadRequestException,
        is DataException.NotFoundException,
        is DataException.ServerException -> DomainException.NetworkException(cause = this)
    }
```

---

앞서 DataException 및 DomainException을 구현하였지만, 최종적으로 처리되는 UI 영역에 대한 지침은 정의하지 않았습니다. **UI 계층은 사용자와 가장 가까이 있는 계층으로 에러를 가공하기 보다는 에러 로그를 전송하거나 사용자에게 에러에 대한 안내를 제공하여야만 합니다.**

따라서 UI 계층에서 정의하는 예외 구조는 이전 Exception 구조와는 다르게 구성할 것입니다. 해당 계층의 예외 정의 클래스는 AppError로 이름을 지을 것이며, ApplicationException 과 UI에 표시할 메시지를 담당할 uiMessage 필드를 추가하겠습니다.

```kotlin
data class AppError(
    val exception: ApplicationException,
    val uiMessage: String? = null
)
```

이어서 AppError를 구성하기 위한 메서드를 추가하여야만 합니다. 에러는 ApplicationException을 직접적으로 전달받는 방식과 일반적인 예외(Throwable)을 전달받았을 때에 대한 케이스를 정의하였습니다. **만약 Throwable의 구체적인 타입이 Error일 경우 예외를 던지도록 설정하였습니다.**

```kotlin
companion object {
    fun of(exception: ApplicationException): AppError = AppError(
        exception = exception,
        uiMessage = exception.errorCode.toUiMessage(),
    )

    fun of(exception: ApplicationException, uiMessage: String) = AppError(
        exception = exception,
        uiMessage = uiMessage,
    )

    fun of(throwable: Throwable): AppError =
        when (throwable) {
            is Error -> throw throwable
            is ApplicationException -> of(throwable)
            else -> of(
                throwable = UnknownException(
                    message = throwable.message ?: "An unknown error occurred.",
                    cause = throwable.cause,
                    errorCode = UNKNOWN_ERROR,
                ),
                uiMessage = UNKNOWN_ERROR.toUiMessage()
            )
        }

    fun of(throwable: Throwable, uiMessage: String?): AppError =
        when (throwable) {
            is Error -> throw throwable
            is ApplicationException -> of(throwable, uiMessage)
            else -> of(
                throwable = UnknownException(
                    message = throwable.message ?: "An unknown error occurred.",
                    cause = throwable.cause,
                    errorCode = UNKNOWN_ERROR,
                ),
                uiMessage = uiMessage
            )
        }
}
```

uiMessage는 직접 주입받는 방식과 ErrorCode를 통해서 적절한 메시지를 만들어내는 방식을 고안하였고, ErrorCode를 통해 적절한 메시지를 만들기 위한 매핑 메서드를 아래와 같이 정의하였습니다.

```kotlin
private fun ErrorCode.toUiMessage(): String? {
    return if (this is DomainErrorCode) when (this) {
        DomainErrorCode.TIMEOUT_ERROR -> "연결 시간이 초과되었습니다."
        DomainErrorCode.NETWORK_ERROR -> "네트워크 오류가 발생하였습니다."
        DomainErrorCode.UNKNOWN_ERROR -> null
    }
    else null
}
```

---

이번 포스트에서는 클린 아키텍처 구조에서 계층간 예외를 어떻게 전달할지에 대하여 고민해보고 이를 적절하게 해결할 수 있는 아이디어를 제시하였습니다. 하지만, 아키텍처에서 발생하는 예외 구조만 정의했을 뿐 해당 예외들에 대한 처리 과정은 다루지 않았습니다. 실제 에러들은 로그를 전송하거나 사용자에게 안내와 같은 최종적인 처리 과정이 필요합니다.

이번 포스트에서는 이러한 내용들을 생략하였지만, 다음 시리즈에서는 이러한 예외들을 처리하는 실용적인 방법들을 다루도록 하겠습니다.