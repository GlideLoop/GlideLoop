# GlideLoop

GlideLoop는 Kotlin 기반 모듈러 프레임워크입니다.

### 그래서, 대체 이게 뭡니까?

GlideLoop는 Kotlin 기반 기초 라이브러리입니다.

GlideLoop 프로젝트는 기존 프로젝트들의 디펜덴시 관리의 용이함을 위해, 단방향 디펜덴시를 차용해 상위 디펜덴시와 하위 디펜던시를 완전히 분리하여 사용하는 모듈식 라이브러리 구조를 갖고있습니다.

요약하자면, GlideLoop는 버전에 종속되지 않는 중앙 관리식 디펜던시 시스템입니다.<br><br>

### 버전 종속적이지 않은 디펜던시가 어떻게 가능한거죠?
GlideLoop는 [GlideSupport](http://) 플러그인을 통해 컴파일 타임에 외부 접근 레퍼런스를 제외한 모든 코드를 버전명이 포함된 다른 패키지로 분리시킵니다.

그렇기에, GlideLoop를 활용한 라이브러리는 외부에서 API를 사용할 수 있는 API 인스턴스를 제공하는것이 권장됩니다.<br><br>

### 대체 어떻게 작동하는거죠?

GlideLoop 시스템을 이해하려면 먼저 컴파일러 플러그인의 존재를 이해해야 합니다.

GlideLoop는 컴파일시, static과 같은 모든 메서드를 기존 패키지에 프록시 형식으로 탑재하고, 모듈에서 호출을 관장하게 됩니다.

예를 들어, `skywolf46.glideloop.TestClass#testStaticFunction`를 `0.1.0`버전 라이브러리에 사용한다면 최종적으로 해당
메서드는 `skywolf46.glideloop.v0_1_0.TestClass`로 컴파일되게 됩니다.

컴파일러 플러그인은 `skywolf46.glideloop.TestClass`에 다음과 같은 코드를 탑재합니다.

```kotlin
    import skywolf46.glideloop.core.proxy.util.GlidingInvoker
    import skywolf46.glideloop.core.proxy.util.GlidingProtector
    object TestClass {
        private var testStaticFunctionInvoker : (() -> Unit)? = null
        
        fun testStaticFunction(version: String = null) {
            testStaticFunctionInvoker?.invoke() ?: kotlin.run { 
                GlidingProtector.protect(TestClass::class) {
                    if(testStaticFunctionInvoker == null) {
                        testStaticFunctionInvoker = GlideInvoker.findInvoker(TestClass::class, "testStaticFunction", version)
                    }
                }
                testStaticFunctionInvoker?.invoke()
            }
        }
    }
```

`skywolf46.glidingloop.core.proxy` 최초에 설계된 메서드만이 존재하며, 프록시 기능만을 수행합니다.

결론적으로, 컴파일된 모든 파일은 해당 방식으로 다른 클래스 로드 로직을 갖게 되며, 버전에 따른 다른 클래스의 리다이렉션이 가능합니다.

다만, 이 경우 연동된 라이브러리를 사용하는 프로젝트의 경우, 항상 가장 최신의 라이브러리를 갖게 된다는 문제가 발생하게 됩니다.

그렇기 때문에, 버전이 구체적으로 지정된 메서드 콜을 위해서는 라이브러리를 사용하는 프로젝트 또한 [GlideSupporter](https://) 컴파일러 플러그인을 적용하여야 합니다.

[GlideSupporter](https://) 컴파일러 플러그인 적용시, 모든 메서드는 지정된 버전의 파라미터가 자동으로 컴파일시에 추가되게 됩니다.

### 이전 버전과 현재 버전의 메서드가 다른 경우에는 관리가 어떻게 되나요?
GlideLoop는 컴파일러 플러그인, [GlideSupporter](http://)를 통해 추가적인 클래스를 삽입한 후, 클래스를 선택해 가져옵니다.

이 과정에서 클래스는 로드되지 않으며, 클래스 헤더 스캔을 통해 가장 높은 버전의 클래스만 미리 확인 후, 해당 클래스를 JVM에 우선 탑재시킵니다.

JVM은 중복 클래스 로드를 지원하지 않기에 첫번째로 로드된 가장 높은 버전의 스키마만이 남게 됩니다.

해당 로직때문에, static을 통한 라이브러리 코드 작성은 권장되지 않으며, 반드시 사용해야하는 경우 싱글톤을 사용하는것이 권장됩니다.<br><br>

### 동일한 클래스를 하나의 변수에 담기 위해 버전 동일한 캐스팅이 필요합니다.
해당 경우에는 데이터 클래스 관리를 위해 `@Schema` 어노테이션이 지원됩니다.

`@Schema` 어노테이션은 해당 어노테이션이 적용된 클래스를 스키마 클래스로 분리하고, 재작성합니다.

만약 인터페이스에 해당 어노테이션이 적용되었을 경우, GlideLoop 프레임워크는 인터페이스를 스키마 클래스로 분리해 패키지를 변경시키지 않습니다.<br><br>

### 앞의 주의사항을 보았지만 반드시 static을 사용해야 합니다.
피할 수 없는 static 키워드 사용을 위해 GlideLoop에서는 `@Original` 어노테이션을 지원합니다.

`@Original` 어노테이션이 추가된 클래스, 혹은 메서드는 프록시 작성이 되지 않으며, 바이트코드 그대로 클래스에 삽입됩니다.<br><br>

### GlideLoop의 버전 분리 기능 사용을 원하지 않습니다.
GlideLoop의 코어 라이브러리는 모두 버전 분리 기능을 통해 작성된 상태입니다.

원한다면, 프록시가 아닌 버전 패키지에서의 호출을 통해 직접 호출을 하여 사용이 가능하지만 권장되는 사항은 아닙니다.<br><br>

### 스키마 클래스 혹은 스키마 인터페이스의 구조 변경이 필요합니다.
최신 버전에서의 스키마 클래스에의 메서드 추가는 호환성을 해치지 않습니다.

하지만, 스키마 클래스에서 메서드를 제거하거나 메서드의 시그니쳐를 변경하는 경우, 해당 메서드를 사용하는 구버전 라이브러리와의 호환이 완전히 불가능하게 됩니다.

해당 사항은 GlideLoop만이 아닌, 라이브러리를 통한 개발을 통한다면 피할 수 없습니다.

해당 부분은 버전 분리 기능에서 앞으로도 지원될 일이 없습니다.