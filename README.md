# IOC (Inversion of Control)

- **IOC (제어의 역전)**: 개발자가 객체의 생성, 생명주기, 메서드 호출 등을 직접 관리하지 않고, **스프링 컨테이너**에게 제어권을 넘기는 것을 말한다

---

# DI (Dependency Injection)

- **DI (의존성 주입)**: 객체 내부에서 필요한 다른 객체를 직접 생성하는 게 아닌, **외부(IOC 컨테이너)**에서 생성한 후 주입시켜주는 방식이다
- 의존성 주입 방식으로는 총 3가지 방법이 존재한다
    - **생성자 주입 (Contstructor Injection)**
    - **수정자 주입 (Setter Injection)**
    - **필드 주입 (Field Injection)**

## 생성자 주입 (Constructor Injection)

- **생성자 주입**: **생성자**를 통해 의존관계를 주입받는 방식
- 해당 주입 방법은 생성자를 호출 시에 딱 한 번만 호출되는 것을 보장한다
- 스프링 4.3 버전 이후라면 클래스에 생성자가 **단 1개**만 있다면 @Autowired를 생략해도 주입이 되며, 주입받는 필드에 `final` 키워드를 사용함으로써 주입되어야 하는 것을 보장한다

```java
@Controller
public class TestController {
	// final 키워드 사용
	private final TestService testService;
    
    // 생성자가 1개이므로 @Autowired 생략 가능
    public TestController(TestService testService){
    	this.testService = testService;
    }
}
```

<aside>

### 장점

- **불변성(Immutability) 보장**: `final` 키워드를 사용함으로써, 객체 생성 시점에 의존성이 한 번 세팅되면 절대 변경되지 않음을 보장한다
- **순수 자바 코드 단위 테스트 용이**: 스프링 컨테이너 없이도 테스트 코드에서 객체를 생성하여 테스트하기 용이하다
- **NPE 방지**: 객체를 생성하는 시점에 반드시 의존성이 필요하므로, 의존성 주입이 누락되면 아예 컴파일 에러가 발생하여 컴파일 시점에서 에러를 잡을 수 있다
</aside>

<aside>

### 단점

- 의존성이 많아질수록 생성자 코드가 길어지고 번거롭다 → 이는 **Lombok**의 @RequiredArgsConstructor를 사용하면 해결된다

```java
@Controller
@RequiredArgsConstructor // final이 붙은 필드의 생성자를 자동으로 만들어준다
public class TestController {
	private final TestService testService;
}
```

</aside>

## 수정자 주입 (Setter Injection)

- **수정자 주입**: 클래스의 필드를 변경하는 **Setter(수정자)** 메서드에 `@Autowired`를 붙여서 주입하는 방식

```java
@Controller
public class TestController {
	private TestService testService;
    
    @Autowired 
    public void setTestService(TestService testService){
    	this.testService = testService;
    }
}
```

<aside>

### 장점

- 주입받는 객체가 변경될 가능성이 있거나, 반드시 주입받지 않아도 되는 경우에는 유용하다
    - 하지만 실제 백엔드 애플리케이션에서 런타임에 의존성을 변경하는 일은 극히 드물다
</aside>

<aside>

### 단점

- **NPE 발생 위험** : 주입이 완료되지 않은 상태에서도 객체가 생성되고 메서드가 호출될 수 있어, 실행 도중 에러 발생할 수 있다
- **의존성 노출**: 의존성을 주입하기 위해 Setter 메서드를 public으로 열어두어야 하므로, 누군가 변경할 수 있는 여지를 남긴다
</aside>

## 필드 주입 (Field Injection)

- 필드 주입: 클래스의 필드에 `@Autowired`를 붙여서 주입하는 방식

```java
@Controller
public class TestController {
	@Autowired
	private TestService testService;
}
```

<aside>

### 장점

- **코드의 간결성**: 클래스 내부에 생성자나 수정자 메서드를 작성하지 않아, 가장 읽기 쉽고 쓰기 편하다
</aside>

<aside>

### 단점

- **단위 테스트의 어려움:** 스프링 컨테이너가 없으면 외부에서 의존성을 주입할 방법이 없어, 순수 자바 코드로 테스트를 작성할 때 의존성을 Mock 객체로 교체하기가 어렵다
- **불변성을 보장 X**: final 키워드를 선언할 수 없어, 객체가 생성된 이후에 의존성이 변경될 위험이 있다
</aside>

---

# AOP (Aspect-Oriented Programming)

- AOP (관점 지향 프로그래밍): 흩어진 **공통 관심사를 하나의 모듈(Aspect)로 싹 모아서 분리**해 버리는 프로그래밍 기법이다.

## Why AOP?

<aside>

1. 웹 애플리케이션을 만들다 보면 다양한 **핵심 비즈니스 로직**을 작성하게 된다
2. 그런데 서비스를 운영하기 위해선 핵실 로직 외에도 **부가적인 기능**들이 필수적이다
    1. **로깅(Logging)**: 메서드 호출 시간을 측정
    2. **보안(Security)**: 사용자가 권환이 있는지 확인
    3. **트랜잭션(Transaction)**: 데이터베이스 작업 단위로 묶어주기
3. 문제는, 이러한 부가적인 기능들이 다른 수많은 클래스와 메서드에 **동일하게 반복되어 작성**된다

   ⇒ 이렇게 여러 곳에 걸쳐서 나타나는 공통된 부가 기능을 **Cross-Cutting Concerns(공통 관심사)**라고 부른다


### ⚠️ 만약 로그를 남기는 코드를 수백 개의 메서드에 다 작성을 완료했는데, 나중에 로그 형식을 바꿔야 해야한다면?

- 하는 수 없이 수백 개의 클래스를 전부 열어서 일일이 수정해야 한다…

⇒ 이를 해결하기 위해서 **AOP 기법**을 사용하여 **공통 관심사를 하나의 모듈(Aspect)**로 모아서 분리해버린다

- 이를 통해 개발자는 핵심 비즈니스 로직만 작성해 두고, 해당 패키지에 있는 모든 서비스 메서드가 실행 될 때, 메서드 호출 시간 측정 로그만 남겨달라고 설정만 해두면, 프레임워크가 알아서 적용해준다

</aside>

## AOP 핵심 용어

| **용어** | **상세내용** |
| --- | --- |
| **JoinPoint** | • Method를 호출하는 '시점', 예외가 발생하는 '시점'과 같이 Application 실행할 때, 특정 작업이 실행되는 '시점' 의미<br>• Advice가 적용될 위치, 끼어들 수 있는 지점<br>• Method 진입 지점, 생성자 호출 시점, Field에서 값을 꺼내 올 때 등 다양한 시점에 적용 |
| **Advice** | • JoinPoint에서 실행되어야 하는 코드<br>• 부가적 관점에 해당 (트랜잭션, 로그, 보안, 인증 등..)<br>• 실제 어떤 일을 해야할지에 대한 내용 |
| **Target** | • 실질적인 비즈니스 로직을 구현하고 있는 코드<br>• 핵심관점에 해당<br>• Aspect를 적용하는 곳 (Class 혹은 Method) |
| **PointCut** | • Target Class와 Advice가 결합(Weaving)될 때, 둘 사이의 결합 규칙 정의<br>- Advice가 실행된 Target 특정 Method 지정<br>• JoinPoint의 상세 스펙 정의 한 것으로 A란 **Method의 진입 시점에 호출할 것**과 같이 더욱 구체적으로 Advice 실행 지점 명시 |
| **Aspect** | • Advice와 PointCut을 합쳐 하나의 Aspect으로 지칭<br>- 일정패턴을 가지는 Class에 Advice를 적용하도록 지원할 수 있는 것을 Asepct라고 지칭<br>• 흩어진 관심사를 모듈화 한 것으로 주로 부가 기능 모듈화 |
| **Weaving** | • AOP에서 JoinPoint들을 Advice로 감싸는 과정 지칭<br>• Weaving 하는 작업을 도와주는 것이 AOP 툴이 하는 역할 |

## 주요 어노테이션

| 종류 | **상세내용** |
| --- | --- |
| `@Around` | Target Method를 감싸 특정 Advice 실행하기 위해 사용 |
| `@Before` | Advice Tartget Method 호출 되기 전 Advice 기능 수행을 위해 사용 |
| `@After` | Target Method 결과에 관계없이  |
| `@AfterReturning` | Target Method가 성공적으로 결과값 반환 뒤 Advice 기능 수행을 위해 사용 |
| `@AfterThrowing` | Target Method가 예외 발생 시 Advice 기능 수행을 위해 사용 |
| `@Around` | Advice Target Method를 감싸 Target Method 호출 전과 후에 Adivce 기능 수행을 위해 사용 |

```java
@Aspect // 해당 클래스가 Aspect 선언
@Component
public class PerformanceAspect {

    // @MeasureTime 어노테이션이 붙은 메서드에만 적용 (PointCut)
    @Around("@annotation(MeasureTime)")
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        // 타겟 메서드 실행 (Target)
        Object result = joinPoint.proceed();

        stopWatch.stop();

        // 어떤 메서드가 실행되었는지 이름과 걸린 시간을 로그로 출력
        System.out.println("[성능 측정] " + joinPoint.getSignature().getName() + " 메서드 실행 시간: " + stopWatch.getTotalTimeMillis() + "ms");

        return result;
    }
}
```

---

# PSA (Portable Service Abstraction)

- PSA(일관된 서비스 추상화): 추상화를 통해 스프링의 내부 구현이 변경되거나 동일 기능에 대한 다른 기술을 사용하더라도 동일한 포멧으로 개발자가 사용 가능할 수 있도록 하는 스프링의 핵심 원칙
- 이 방법을 통해 특정 기술에 대한 종속성을 줄이고 유연하고 확장성있는 개발이 가능하며 모든 코드 구조를 알지 않아도 되므로 코드가 간결해진다

```java
@Service
public class OrderService {

    // 개발자는 하위 기술이 JDBC인지 JPA인지 알 필요가 없다
    // @Transactional을 선언함으로 트랜잭션 적용 완료
    @Transactional
    public void createOrder() {
        orderRepository.save(new Order());
        inventoryRepository.decrease(1L);
    }
}
```

---

# Spring Bean

## Spring Bean이란?

- Spring Bean: 스프링 **IoC 컨테이너**가 생성하고 관리하는 **자바 객체**

## Bean의 Lifecycle

<aside>

1. **스프링 컨테이너 생성:** 스프링 애플리케이션이 실행
2. **스프링 빈 생성:** 등록된 빈들의 객체를 인스턴스화
3. **의존성 주입 (DI):** 생성자 주입, 필드 주입 등을 통해 필요한 의존성을 서로 엮어준다
4. **초기화 콜백:** 빈이 생성되고 의존성 주입까지 완료된 후, 초기화 작업을 할 수 있도록 특정 메서드 (`@PostConstruct`)를 호출
5. **빈 사용:** 애플리케이션이 돌아가면서 빈들이 비즈니스 로직 수행
6. **소멸 전 콜백:** 스프링 컨테이너가 종료되기 직전에, 안전하게 자원을 반납할 수 있도록 특정 메서드 (`@PreDestroy`)를 호출
7. **스프링 컨테이너 종료:** 애플리케이션이 종료
</aside>

## Bean Scope

1. **Singleton (싱글톤 스코프)** - **스프링의 기본값**
    - 스프링 컨테이너의 **시작부터 종료까지 유지**되는 가장 넓은 범위의 스코프
    - 컨테이너 안에 **오직 1개의 인스턴스만** 생성되고, 이 빈을 요청하는 모든 곳에서 **같은 객체를 공유**해서 사용한다
    - 객체를 매번 생성하지 않아서 메모리 낭비가 적고, 성능이 좋다
2. **Prototype (프로토타입 스코프)**
    - 스프링 컨테이너에 빈을 요청할 때마다 **매번 새로운 인스턴스**를 생성해서 반환한다
    - 스프링이 프로토타입 빈을 생성하고 의존성 주입하는 것까지만 책임지며, 그 이후로는 관리하지 않는다.

      → `@Predestroy` (종료 콜백)이 호출되지 않는다

3. **웹 관련 스코프**
    1. **request:** HTTP 요청 하나가 들어오고 나갈 때까지 유지되는 스코프 (요청마다 별도의 빈 생성)
    2. **session:** HTTP 세션과 동일한 생명주기를 가지는 스코프
    3. **application:** 서블릿 컨텍스트와 생명주기를 같이 하는 스코프

## Annotation

- **Annotation(@)**: 코드에 특별한 의미를 부여하고 추가적인 정보를 제공하는 **메타데이터의 한 종류**로 컴파일러나 프레임워크에게 유용한 정보를 제공하는 역할을 수행

### Annotation 역할

1. 컴파일러에게 코드 작성 문법 에러를 체크하도록 정보를 제공
2. 소프트웨어 개발 툴이 빌드나 배치 시 코드를 자동으로 생성할 수 있도록 정보를 제공
3. 실행할 때 특정 기능을 실행하도록 정보를 제공

## 스프링에서 어노테이션으로 Bean을 등록하는 과정

<aside>

1. **애플리케이션 시작 및 컴포넌트 스캔 시작**
    - 스프링 부트 애플리케이션을 실행한다
    - 가장 먼저 `@SpringBootApplication` 어노테이션 내부에 있는 `@ComponentScan`이 작동한다.
    - 이 스캐너는 개발자가 지정한 패키지와 그 하위 패키지들을 스캔하기 시작한다
2. **빈 정의서 생성**
    - 어노테이션이 붙은 클래스를 발견하면, `BeanDefinition`이라는 객체를 만든다
        - `BeanDefinition`: 빈 생성 레시피
        - 해당 객체에는 다음과 같은 정보가 담겨진다
            - 어떤 클래스인가? (Class Name)
            - 싱글톤? 프로토타입? (Scope)
            - 해당 객체를 만들 때 다른 객체가 필요한가? (생성자 정보)
    - 객체를 생성한 후 `BeanDefinitionRegistry`에 저장한다
3. **빈 생성 및 의존성 주입**
    - `BeanDefinitionRegistry` 에 저장된 모든 `BeanDefinition`을 읽어들여서 의존성이 얽혀있는 순서를 파악한다
    - 그 후 객체를 실제로 메모리에 생성한다
    - 마지막으로 생성된 객체들 사이에 필요한 의존성을 주입한다
4. **빈 컨테이너에 저장**
    - 완성된 객체는 스프링 컨테이너 내부에 저장된다
    - 애플리케이션에서 객체를 요청하면, 컨테이너가 레지스트리에서 꺼내어 동일한 객체를 반환해 준다
</aside>

### `@ComponentScan`

- 해당 어노테이션은 `@Component` 어노테이션이 붙은 클래스들을 자동으로 빈으로 등록해주는 역할을 한다
    - `@Controller`, `@Repository`, `@Service`, `@Configuration` 어노테이션 내부에 `@Component`가 존재하여 다 빈 등록 대상들이다.

    ```java
    //@Controller
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Component
    public @interface Controller {
       ...
    }
    ```

- `@ComponentScan`이 붙은 클래스의 패키지 경로를 기반으로 탐색을 시작하여, `@Component`가 붙은 클래스를 찾으면 빈으로 등록해준다
    - `@ComponentScan`은 `SpringBootApplication` 어노테이션 내부에 존재한다

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
// SpringBootApplication 안에 있는것을 확인할 수 있다
@ComponentScan( 
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
   ...
}
```

- 결국에는 **프로젝트 root**에 존재하는 `SpringBootApplication`가 최상위 루트로 설정되어, r**oot에서 하위 패키지까지 다 컴포넌트 스캔을 진행**할 수 있다.

## (선택) 하나의 interface를 구현한 service가 여러개 있을때 어떻게 주입 해야할까요?

<aside>

</aside>

---

# MVC & Spring MVC

## MVC

- MVC 패턴은 애플리케이션을 세 가지 핵심 역할로 나누어 개발하는 아키텍처 패턴

### Model

- 애플리케이션의 **데이터와 비즈니스 로직을 담당**한다
- 데이터베이스와 소통하며 데이터를 저장, 수정, 조회하는 모든 작업이 일어난다

### View

- 사용자에게 보여지는 **화면을 담당**한다
- 모델로부터 데이터를 전달받아 화면을 그려주는 역할만 한다

### Controller

- 모델과 뷰 사이의 **중개자 역할**을 한다
- 사용자의 요청을 받아 모델에 데이터 처리를 지시하고, 그 결과를 바탕으로 어떤 뷰를 보여줄지 결정한다

⇒ 이렇게 함으로써, 화면을 바꾸더라도 모델과 컨트롤러에는 **아무런 영향을 끼치지 않아 협업과 유지보수가 편리**해진다

## Spring MVC

- 웹 애플리케이션에 특화된 구체적인 프레임워크 구현체
- `DispatcherServlet`이라는 단일 진입점이 모든 요청을 통제한다

⇒ 즉, **MVC**는 역할을 3개로 나누는 **아키텍처 패턴**이며, **Spring** **MVC**는 MVC의 아이디어를 바탕으로 웹서버를 더욱 튼튼하게 돌리기 위해 **DispaatcherServlet이라는 진입점과 여러 부품들을 추가**하여 만든 구조이다

## Servlet

- **Servlet**: 웹 브라우저의 요청을 받아 **처리**하고, **동적인 결과**를 만들어 반환하는 자바 웹 프로그래밍 기술

## 웹 요청 처리 과정

<aside>

1. **HTTP 요청 수신**
    1. 클라이언트가 웹 서버로 HTTP 요청을 보낸다
    2. 웹 서버는 **정적인 파일들은 직접 처리**하고, **동적인 로직**이 필요한 요청은 WAS 내부에 있는 **서블릿 컨테이너**로 넘긴다
2. **Request / Response 객체 생성**
    1. 서블릿 컨테이너는 클라이언트의 HTTP 요청 메시지를 파싱하여 `HttpServletRequest` 객체를 만든다
        1. 여기에는 URL, 파라미터, 헤더 정보와 같은 내용들이 담긴다
    2. 응답을 담아 보낼 빈 박스인 `HttpServletResponse` 객체도 생성한다
3. **Servlet 호출 및 로직 실행**
    1. 컨테이너는 **요청된 URL에 매핑된 서블릿**을 찾는다
        1. 만약 객체가 없다면 생성
    2. 서블릿의 `service` 메서드를 호출하여 앞서 만든 Request와 Response 객체를 **파라미터**로 넘겨준다
    3. `service` 메서드에서 개발자가 작성한 **비즈니스 로직이 실행**된다
4. **Response 반환**
    1. 로직 처리가 끝나고 서블릿은 결과를 `HttpServletResponse` 객체에 담는다
    2. 컨테이너는 해당 Response 객체의 내용을 다시 HTTP 응답 메시지 형태로 변환하여 웹 서버를 거쳐 클라이언트에게 보낸다
5. **자원 정리**
    1. 응답이 완료되면, Request와 Response 객체는 **메모리에서 소멸**된다
</aside>

## WAS (Web Application Server)

- **WAS**: 웹서버가 처리하지 못하는 복잡한 로직과 동적인 데이터를 처리(애플리케이션 로직 처리)하는 애플리케이션 서버
- 근데 보통 WAS = 웹 서버 + 웹 컨테이너(서블릿 컨테이너)의 형태를 가진다
- 따라서, 정적인 요청도 처리할 수 있으며, 내부적으로 서블릿을 실행할 수 있는 환경도 갖추고 있다

## Tomcat

- **Tomcat**: 수많은 WAS 제품 중 하나이자 가장 유명한 자바 기반의 WAS
- 자바 서블릿과 자바 서버 페이지를 실행할 수 있는 웹 서버이자 서블릿 컨테이너이다

## DispatcherServlet

- **DispatcherServlet**: HTTP 프로토콜로 들어오는 모든 요청을 가장 먼저 받아 적합한 컨트롤러에 위임해주는 프론트 컨트롤러
- 모든 요청을 처리하다보니 정적 파일에 대한 요청마저 모두 가로채는 문제가 있어서 다음과 같이 해결하고자 했다
    1. 정적 자원 요청과 애플리케이션 요청을 분리
        1. `/apps` 의 URL로 접근하면 **Dispatcher Servlet**이 **담당한다**
        2. `/resources` 의 URL로 접근하면 Dispatcher Servlet이 컨트롤할 수 없으므로 **담당하지 않는다**
    2. 애플리케이션 요청을 탐색하고 없으면 정적 자원 요청으로 처리
        1. 요청을 처리할 컨트롤러를 먼저 찾고, 요청에 대한 컨트롤러를 찾을 수 없는 경우에, 2차적으로 설정 된 자원 경로를 탐색하여 자원을 탐색한다

### 디스패처 서블릿이 요청을 받아서 컨트롤러로 위임하는 과정

1. **서블릿 요청/응답을 HTTP 서블릿 요청 응답으로 변환**
    1. HTTP 요청이 등록된 필터들을 거쳐 디스페처 서블릿이 처리하는데, 가장 먼저 요청 받는 부분은 `service` 메소드이다

        ```java
        public abstract class HttpServlet extends GenericServlet {
        
            ...
        
            @Override
            public void service(ServletRequest req, ServletResponse res)
                throws ServletException, IOException {
        
                HttpServletRequest  request;
                HttpServletResponse response;
        
                try {
                    request = (HttpServletRequest) req; // Servlet 관련 Request 객체를 Http 관련 Request로 캐스팅
                    response = (HttpServletResponse) res; // Servlet 관련 Response 객체를 Http 관련 Response로 캐스팅
                } catch (ClassCastException e) {
                    throw new ServletException(lStrings.getString("http.non_http")); // 캐스팅시 HTTP 요청이 아니므로 에러를 던진다
                }
                service(request, response);
            }
            
            ...
        }
        ```

2. **Http Method에 따른 처리 작업 진행**
    1. 캐스팅 후 `HttpServletRequest` 객체를 파라미터로 갖는 `service` 메소드를 호출한다
    2. 과거 HTTP 표준에는 `PATCH` 메소드가 존재하지 않았다 (???) → javax에는 `doPatch` 메소드가 존재하지 않아서 스프링 개발팀이 자체적으로 개발하여 추가했다
    3. 그 외의 모든 메소드들은 자바 표준 기술인 부모 클래스 `HttpServlet`이 처리한다

        ```java
        public abstract class HttpServlet extends GenericServlet {
        
            ...
        
            protected void service(HttpServletRequest req, HttpServletResponse resp)
                throws ServletException, IOException {
        
                String method = req.getMethod();
        
                if (method.equals(METHOD_GET)) {
                    ...
                    doGet(req, resp);
        
                } else if (method.equals(METHOD_HEAD)) {
                    long lastModified = getLastModified(req);
                    maybeSetLastModified(resp, lastModified);
                    doHead(req, resp);
        
                } else if (method.equals(METHOD_POST)) {
                    doPost(req, resp);
        				...
        
                } else {
                    ... // 에러 처리
                }
            }
        
            ...
        }
        ```

    4. `HttpServlet`에서 요청 메소드에 따라 필요한 처리와 `doX` 메소드를 호출한다
    5. 이제는 `doX` 메소드를 오버라이딩하고 있는 자식 클래스인 `FrameworkServlet`로 다시 요청이 이어진다

        ```java
        public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
        
            ... 
        
            @Override
            protected final void doGet(HttpServletRequest request, HttpServletResponse response)
                    throws ServletException, IOException {
        
                processRequest(request, response); // 모든 doX 메소드는 processRequest라는 메소드를 호출한다
            }
            
            @Override
            protected final void doPost(HttpServletRequest request, HttpServletResponse response)
                    throws ServletException, IOException {
        
                processRequest(request, response);
            }
        
            ...
        }
        ```

3. **요청에 대한 공통 처리 작업 진행**
    1. `doX` 메소드 내부를 보면 결국에 전부 `processRequest` 메소드를 호출하고 있다
    2. `processRequest`에서는 `request`에 대한 공통 처리를 다 하고 `doService`를 호출한다
4. **컨트롤러로 요청을 위임 (이 부분은 도저히 이해가 되지 않습니다… 맞는 내용인지 모르겠습니다…)**
    1. 요청에 패핑되는 `HandlerMapping` (`HandlerExecutionChain`) 조회

       → `HandlerMapping`를 조회하여 알맞는 `Intercept`를 `chain`으로 묶어서 **반환**

    2. 요청을 처리할 `HandlerAdapter` 조회

       → 해당 `Controller`의 **종류를 찾기 위해** `HandlerAdaptor`를 조회

    3. `HandlerAdapter`를 통해 컨트롤러 메소드 호출(`HandlerExecutionChain` 처리)

       → `HandlerAdaptor`를 통해 컨트롤러 메소드 호출하고 그 결과를 다시 `HandlerAdaptor`한테 반환

       → 이걸 또 `dispatcherServlet`에게 반환