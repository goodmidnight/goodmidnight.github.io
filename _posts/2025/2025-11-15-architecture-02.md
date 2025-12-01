---
title: Android Clean Architecture Error Handling (2) - 클린 아키텍처 예외 처리 사례
date: 2025-11-15 00:00:00
categories: [ android, clean-architecture ]
tags: [ android,clean-architecture,development ]
math: true
---


해당 포스트는 이전 포스트와 연결되므로 이전 포스트를 먼저 보고 오시기를 추천드립니다.

- [Android Clean Architecture Error Handling (1) - 클린 아키텍처 예외 구조화 사례](https://goodmidnight.github.io/android/clean-architecture/architecture-01/)

이전 포스트(1부)에서는 클린 아키텍처의 각 레이어에서 예외를 구조화하고, 예외를 UI 레이어까지 전달하는 아이디어 다루었습니다.

하지만, 실제 서비스에서 발생되는 예외 처리는 구조화에서 멈추는 것이 아니라, 사용자에게 피드백을 제공하거나 시스템에 기록이 되는 과정까지 이어져야만 합니다.

이러한 처리 과정은 사용자에게 명확한 대응 방안을 안내하고, 개발팀에게는 서비스를 안정적으로 개선할 수 있는 기반을 제공합니다.

이번 포스트에서는 1부에 이어 예외 흐름의 종착지인 UI 레이어를 중심으로 후속 처리를 어떠한 방식으로 수행할지에 대한 내용을 다루어보겠습니다.

# 베경지식

예외를 처리하는 방식을 이해하기 위해서는 이와 관련한 배경 지식들이 요구됩니다. 해당 파트에서는 관련 배경 지식들을 간략하게 소개하겠습니다.

## Log

Log는 개발 과정에서 애플리케이션의 동작을 추적하거나 문제를 해결하기 위해 사용하는 기록으로 Log를 통해 실행 과정 중 특정 시점의 값이나 함수 호출부 또는 이벤트와 같은 다양한 상태를 확인할 수 있습니다.

Log를 활용할 경우 프로그램의 실행 및 제어 흐름을 파악하는데 도움이 되며, 실제 프로그램이 구동되었을 때 특정 포인트에서 데이터가 어떠한 형태를 나타나는지를 확인할 수 있습니다. 또한 일부 실행 흐름에서 발생하는
중요한 정보들을 저장하거나 전송하는 방식으로 연계하여 개발자가 안정적인 시스템을 만드는데 기여합니다.

**Android에서 Log를 간단하게 출력하기 위해서는 android.util.Log의 정적 메서드를 호출해야하며, 개발자는 `Logcat`이라는 도구를 통해 해당 기록을 실시간으로 확인할 수 있습니다.**
Log를 호출하는 방법은 아래와 같습니다.

```kotlin
import android.util.Log

...

val TAG = "MainActivity" 
Log.d(TAG, "onCreate 메서드가 호출되었습니다.")
```

코드에서 사용된 Log.d는 디버그 목적에서 사용되는 Log이며 이외에도 Log는 중요도와 사용목적에 따라 여러가지 메서드가 존재하니 이에 따라 적절하게 사용하는 것이 중요합니다. 종류는 아래와 같습니다.

| 메서드                                 | 설명                                                          |
|-------------------------------------|-------------------------------------------------------------|
| Log.v() (Verbose)                   | 자세한 정보를 출력할 때 사용됩니다. 주로 개발 과정에서의 변수값이나 흐름을 캡처할 때 사용합니다.     |
| Log.d() (Debug)                     | 디버깅 과정에서 필요한 정보를 출력할 때 사용합니다.                               |
| Log.i() (Info)                      | 일반적인 정보성 메시지를 나타내며, 앱의 일반적인 실행 흐름과 같은 정보성 메시지를 나타낼 때 사용합니다. |
| Log.w() (Warning)                   | 심각한 문제는 아니지만, 잠재적인 위험이나 비정상적인 동작이 감지되었을 때 사용합니다.            |
| Log.e() (Error)                     | 앱의 실행에 직접적인 영향을 미치는 심각한 오류가 발생했을 때 사용합니다.                   |
| Log.wtf() (What a Terrible Failure) | Error 보다 시스템에 심각한 영향을 미치는 오류를 발생하였을 때 사용합니다.                |


추가적으로 `Log.d()`의 첫 번째 파라미터로 사용된 `TAG`는 로그를 필터링하는 데 사용되는 식별자 입니다. 일반적으로 클래스 이름을 `TAG`로 사용하고 `companion object`에 상수로 선언하여
일관성을 유지하는 것이 일반적인 관행입니다.


```kotlin

class HelloProvider {소
    operator fun invoke() {
        println("Hello!")
        Log.i(TAG, "invoke: Hello!")
    }

    companion object {
        const val TAG: String = "HelloProvider"
    }
}
```

# 구현

이어서 [[이전 포스트]](https://goodmidnight.github.io/android/clean-architecture/architecture-01/)에서 구성한 최종 구조를 바탕으로 예외를 처리하는 몇 가지의
아이디어를 소개하겠습니다. 이전 포스트의 최종적인 결과인 `UiException` 는 아래와 같이 구성되어있습니다.

```kotlin
data class UiException(
    val cause: ApplicationException,
    val uiMessage: String? = null,
) {
    companion object {
        fun of(exception: ApplicationException): UiException = UiException(
            cause = exception,
            uiMessage = exception.errorCode.toUiMessage(),
        )

        fun of(exception: ApplicationException, uiMessage: String) = UiException(
            cause = exception,
            uiMessage = uiMessage,
        )

        fun of(throwable: Throwable): UiException =
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

        fun of(throwable: Throwable, uiMessage: String?): UiException =
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

        fun ErrorCode.toUiMessage(): String? {
            return if (this is DomainErrorCode) when (this) {
                DomainErrorCode.TIMEOUT_ERROR -> "연결 시간이 초과되었습니다."
                DomainErrorCode.NETWORK_ERROR -> "네트워크 오류가 발생하였습니다."
                DomainErrorCode.UNKNOWN_ERROR -> null
            }
            else null
        }
    }
}
```

## UI 피드백

가장 먼저 uiMessage를 활용하여 사용자에게 피드백을 전달하는 방식에 대해 다루어보겠습니다.

예를 들어 아래와 같이 데이터를 불러오는 클래스가 있다고 가정해봅시다. 해당 클래스에는 아이템 목록을 불러오는 fetchItems() 라는 메서드가 있지만, 이 메서드는 네트워크를 통해 아이템을 불러오지 못해 예외를
발생시키고 있습니다.

```kotlin
class DataFetcher {
    fun fetchItems(): Result<List<Item>> {
        
        //...
        // 네트워크 요청 실패로 아이템을 받아오지 못하고 있다.
        return Result.failure(DomainException.NetworkException())
    }

    data class Item(
        val id: Long,
        val title: String,
    )
}
```

UI 레이어의 ViewModel에서는 DataFetcher 클래스를 통해 아이템 목록을 불러와 UI 업데이트를 하기 원하였다 가정하고 이를 구현하기 위해 아래와 같은 로직을 설계하였습니다.

```kotlin
class MainViewModel(
    private val dataFetcher: DataFetcher
) : ViewModel() {
    
    private fun loadItems() {
        val fetchResult: Result<List<DataFetcher.Item>> = dataFetcher.fetchItems()
        fetchResult.fold(
            onSuccess = {
                // TODO Handle success
            },
            onFailure = { e -> 
               // TODO Handle failure
            }
        )
    }
}

```

성공하였을 때는 UI를 업데이트하면 되지만, 만약 실패를 하였을 경우에는 사용자에게 UI를 통한 피드백을 전달하여야 합니다. **피드백은 스낵바나 토스트 같은 알림을 노출시키거나 실패 전용 화면을 로드하는 방식 등
다양한 방법으로 처리할 수 있습니다**. 해당 예시에서는 2가지를 모두 사용하겠습니다.

```kotlin
import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.flow.StateFlow

class MainViewModel(
    private val dataFetcher: DataFetcher,
) : ViewModel() {

		// UI 화면 상태 플로우
    private val _screenState = MutableStateFlow<ScreenState>(ScreenState.Loading)
    val screenState: StateFlow<ScreenState> = _screenState

		// 스낵바 메시지 플로우
    private val _snackbar = MutableSharedFlow<String>()
    val snackbar: SharedFlow<String> = _snackbar

    init {
        loadItems()
    }

    private fun loadItems() {
        val fetchResult: Result<List<DataFetcher.Item>> = dataFetcher.fetchItems()
        fetchResult.fold(
            onSuccess = {
		            // 성공시에는 UI 화면 상태를 업데이트 한다.
                _screenState.value = ScreenState.Success(it)
            },
            onFailure = { e ->
                // TODO Handle failure
            }
        )
    }

    sealed interface ScreenState {
        data object Loading : ScreenState
        data class Success(val items: List<DataFetcher.Item>) : ScreenState
        data class Failure(val uiException: UiException) : ScreenState
    }
}
```

예시 코드에서는 전반적인 화면의 상태를 관리하는 ScreenState와 Snackbar의 메시지를 관리할 플로우를 도입하였습니다. 데이터 패치가 성공적으로 이루어질경우 screenState 플로우에 Success
타입이 발행됩니다.

그렇다면 실패의 경우는 어떻게 처리해야할까요? 앞서 UiException에서는 `UiException.of()` 팩터리 메서드를 구현해 두었기 때문에 해당 메서드를 활용한다면, 예외를 처리하기 유리한 타입으로 변환할
수 있게 됩니다.

```kotlin
private fun loadItems() {
    val fetchResult: Result<List<DataFetcher.Item>> = dataFetcher.fetchItems()
    fetchResult.fold(
        onSuccess = {
            _screenState.value = ScreenState.Success(it)
        },
        onFailure = { e ->
            val exception = UiException.of(e)
        }
    )
}
```

잠깐 다시 짚고 넘어가자면, 기존에 구현한 `UiException`은 아래처럼 uiMessage와 cause필드로 구성되어있습니다. **사용자는 모든 예외에 대한 정보를 알 필요가 없으며, UI에 예외 메시지
피드백을 전달할 필요도 필요도 없습니다.** 이를 대응하기 위해서 UiException 내부 uiMessage 필드 타입을 nullable로 설정하였습니다. 개발자는 사용자가 알아야 할 예외에 대해서만
uiMessage를 전달하도록 구성하면 됩니다.

```kotlin
data class UiException(
    override val cause: ApplicationException,
    val uiMessage: String? = null,
)
```

이제 loadItems() 로직을 실패하였을 때의 처리방식을 구현할 수 있게됩니다. uiMessage 필드가 존재할 경우에만 스낵바 플로우에 데이터를 발행하기만 하면됩니다. 아래와 같이 옵셔널 체이닝을 한다면
간단하게 표현할 수 있습니다.

```kotlin
private fun loadItems() {
    val fetchResult: Result<List<DataFetcher.Item>> = dataFetcher.fetchItems()
    fetchResult.fold(
        onSuccess = {
            _screenState.value = ScreenState.Success(it)
        },
        onFailure = { e ->
            val exception = UiException.of(e)
            exception.uiMessage?.also { _snackbar.tryEmit(it) }
        }
    )
}
```

앞서 DataFetcher가 fetchItems()을 할 때 `DomainException.NetworkException()` 을 반환하도록 코드를 작성하였기 때문에, 그에 따른 uiMessage가 스낵바 플로우로
전달되게 될 것 입니다. UI 로직에서는 해당 플로우를 감지하여 적절한 타이밍에 스낵바 컴포넌트를 노출시키기만 하면 됩니다.

## 로그 출력

두번째 처리 아이디어는 로그 출력입니다. 아래와 같이 UiException 클래스 내에 로그를 출력해주는 메서드를 정의합니다.

```kotlin
data class UiException(
    val cause: ApplicationException,
    val uiMessage: String? = null,
) {
		override fun toString(): String {
		    return "$TAG(cause=$cause, uiMessage=$uiMessage)"
		}

   fun printLog() = Log.e(TAG, this.toString())
	 
	 companion object {
      private val TAG: String = UiException::class.java.simpleName
      
      // ...
   }
}    
```

1부에서 ApplicationException에 대해 `toString()` 메서드를 오버라이드하였기 때문에, UiException에 대하여 더욱 상세한 로그정보를 노출시킬 수 있습니다.

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

<p align="center">
<img src="/assets/post/2025/2025-11-15-architecture-02-01.png" width="80%"/>
</p>

Throwable 객체는 예외 스택을 출력할 수 있는 `printStackTrace()`  메서드를 제공합니다. 디버그 상태에서는 예외의 자세한 정보를 아는것이 중요하기 때문에 해당 메서드를 활용해보겠습니다.
메서드의 이름은 `handleError()`로 짓고, 디버그 상태에서만 호출되도록 하겠습니다. **`BuildConfig.DEBUG`를 활용한다면 손쉽게 디버깅 모드만의 동작을 정의할 수 있습니다.**

```kotlin
data class UiException(
    override val cause: ApplicationException,
    val uiMessage: String? = null,
) : BaseError {

		// 디버그 상태일 경우, 스택과 로그를 동시에 출력한다.
    fun handleError() {
        if (BuildConfig.DEBUG) {
            printLog()
            cause.printStackTrace()
        }
    }

    override fun toString(): String {
        return "$TAG(cause=$cause, uiMessage=$uiMessage)"
    }

    fun printLog() = Log.e(TAG, this.toString())

    companion object {
        private val TAG: String = UiException::class.java.simpleName
        
        // ...
    }   
}
```

이제 다시 ViewModel로 넘어와 아래와 같이 `handleError()` 를 적용해줍니다.

```kotlin
private fun loadItems() {
    val fetchResult: Result<List<DataFetcher.Item>> = dataFetcher.fetchItems()
    fetchResult.fold(
        onSuccess = {
            _screenState.value = ScreenState.Success(it)
        },
        onFailure = { e ->
            val exception = UiException.of(e)
            exception.uiMessage?.also { _snackbar.tryEmit(it) }
            //handleError 추가
            exception.handleError()
        }
    )
}
```

## 로그 전송

마지막 아이디어는 로그를 전송하는 것입니다. **지금까지의 방식은 실제 운영환경에서 사용자마다 어떠한 에러가 발생하였는지에 대한 정보를 수집할 수 없습니다. 개발자가 안정적인 시스템을 만들기 위해서는 이러한 예외
정보를 수집하여야만 합니다.**

좋은 방법은 별도의 로그 관리 서버를 구축하고, 해당 서버에 지속적으로 로그를 전송하는 것입니다. 이때 로그는 발생할 때 마다 보내지 않고 로컬 DB 또는 파일에 적재 후 특정 주기마다 보내야 합니다.
WorkManager를 활용한다면 아이디어를 쉽게 적용할 수 있을 것입니다.

하지만, 별도 서버를 구축하지 않고 모니터링할 수 있는 좋은 솔루션들이 존재합니다. 아래에서 부터는 앱에서 필수적으로 사용되는 Firebase를 활용하여 로그 전송 아이디어를 적용해보겠습니다.

가장 먼저 Firebase 프로젝트를 생성한 후, `FirebaseCrashlytics` 플러그인 및 의존성을 적용해야합니다. 적용하는 방식을 다루기에는 분량이 길어지고, 이 부분에 대한 여러 레퍼런스들이 많이
존재하기 때문에 별도로 다루지 않겠습니다.

```kotlin
firebaseBom = "34.2.0"
firebaseCrashlyticsPlugin = "3.0.6"

...

[plugins]
firebase-crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "firebaseCrashlyticsPlugin" }

...

[libraries]
firebase-bom = { group = "com.google.firebase", name = "firebase-bom", version.ref = "firebaseBom" }
firebase-crashlytics = { group = "com.google.firebase", name = "firebase-crashlytics" }
```

FirebaseCrashlytics는 앱 크래시 기록만 모니터링할 수 있다고 생각하실 수 있겠지만, 실제로는 크래시가 발생하지 않아도 예외를 리포트하고 모니터링할 수 있는 방법이 존재합니다. 아래에서부터는 예외를
기록하는 방법에 대한 내용을 이어가도록 하겠습니다.

Firebase 설정을 프로젝트에 적용하였다면, ‘특정 진입점’에서 아래 선언을 수행해줍니다. 필수적이지는 않지만, 모니터링 시 예외 기록을 필터링 하거나 식별할 수 있기 때문에 강력하게 권장합니다. ‘특정 진입점은
앱을 시작하는 Application의 onCreate() 나 로그인 시점이 될 수 있습니다.

```kotlin
import com.google.firebase.crashlytics.FirebaseCrashlytics

...

// 사용자를 고유하게 식별할 수 있는 아이디 정의 
// 서버로부터 전달받거나 서버가 없을 경우 디바이스 기기 정보를 조합하여 정의할 수 도 있음
val userID: String 

FirebaseCrashlytics.getInstance().setUserId(userID)

```

이후 UiException으로 다시 돌아가 Crashlytics에 예외를 기록하는 로직을 추가합니다. 예외를 기록하기 위해서는 `FirebaseCrashlytics` 의 `recordException()` 메서드를
사용합니다.

```kotlin
data class UiException(
    override val cause: ApplicationException,
    val uiMessage: String? = null,
) : BaseError {

    fun handleError() {
		    // Crashlytics에 예외기록 파라미터로 Throwable객체를 포함한다.
        crashlytics.recordException(cause)
        if (BuildConfig.DEBUG) {
            printLog()
            cause.printStackTrace()
        }
    }

    override fun toString(): String {
        return "$TAG(cause=$cause, uiMessage=$uiMessage)"
    }

    fun printLog() = Log.e(TAG, this.toString())

    companion object {
        private val TAG: String = UiException::class.java.simpleName
        private val crashlytics = Firebase.crashlytics
    }
}
```

실제로 handleError()가 동작되도록 예외를 발생시키면 아래와 같이 Firebase Crashlytics 시스템의 `심각하지 않음`  유형으로 이벤트가 기록된 것을 확인하실 수 있습니다.

<img src="/assets/post/2025/2025-11-15-architecture-02-02.png" />
<img src="/assets/post/2025/2025-11-15-architecture-02-03.png" />
<img src="/assets/post/2025/2025-11-15-architecture-02-04.png" />

---

지금까지 1부에서 제작한 클린 아키텍처 기반의 예외 구조에서 최종적으로 전달되는 예외를 3가지의 아이디어를 통해 처리하는 방법에 다루었습니다.

이러한 아이디어를 통해 사용자에게 적절한 피드백을 제공해주고 개발자는 더욱 안정적인 시스템을 만들기위한 정보들을 획득할 수 있게 되었습니다.

**포스트에서 제시된 내용들은 하나의 아이디어일 뿐 체계적인 시스템 관리를 위해서는 이를 넘어 더욱 다양하고 정교한 프로세스를 구현할 수 있어야만 합니다**. 또한 Firebase Crashlytics 외에도 로그를
수집하고 모니터링할 수 있는 솔루션들이 다양하게 있으니, 여러가지 솔루션를 비교분석하여 적절한 툴을 도입하는 것 또한 중요합니다.