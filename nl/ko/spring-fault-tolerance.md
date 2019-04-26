---

copyright:
  years: 2018, 2019
lastupdated: "2019-03-15"

keywords: fault tolerance spring, hystrix spring, netflix spring, hystrixcommand spring, bulkhead spring, circuit breaker spring

subcollection: java

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}
{:tip: .tip}

# Spring의 결함 허용
{: #spring-tolerance}

복원력 있는 시스템을 작성하면 해당 시스템 내의 모든 서비스에 대한 요구사항이 적용됩니다. 클라우드 환경의 동적 네이처로 인해 서비스가 예기치 못한 상황을 예상하고 정상적으로 응답하도록 디자인되어야 합니다.

Spring은 [Netflix Hystrix](https://github.com/Netflix/Hystrix/wiki){: new_window} ![외부 링크 아이콘](../icons/launch-glyph.svg "외부 링크 아이콘") 결함 허용 라이브러리를 사용하여 애플리케이션 레벨 결함 허용 내용을 지원합니다. Hystrix는 폴백, 회로 차단기, 벌크헤드 및 연관된 메트릭에 대한 지원을 제공합니다. 

이 정보는 [클라우드 고유 개발: 결함 허용](/docs/java?topic=cloud-native-fault-tolerance#fault-tolerance)에 설명된 결함 허용 사례를 기반으로 합니다.
{: note}

## Spring 애플리케이션에서 Hystrix 사용
{: #spring-hystrix-starter}

Hystrix를 Spring 애플리케이션 내에서 쉽게 사용하기 위해 애플리케이션 내에서 Hystrix를 통합하는 어노테이션이 있는 방식을 사용하는 Spring Boot Starter가 있습니다.

애플리케이션에 이 단일 종속성을 추가하는 방식이 Hystrix에 도입됩니다. 

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```
{: codeblock}

다른 많은 Spring Starters와 마찬가지로 애플리케이션에서 기능을 사용하려 한다고 Spring에 표시하려면 기본 애플리케이션 클래스에 어노테이션을 추가합니다. 회로 차단을 위해 Hystrix 어노테이션의 Spring 처리를 사용하려면 `@EnableCircuitBreaker`를 추가하십시오.

```java
@SpringBootApplication
@EnableCircuitBreaker
public class MyApplication {

	public static void main(String[] args) {
		SpringApplication.run(MyApplication.class, args);
	}
```
{: codeblock}

### HystrixCommand를 사용하여 폴백 정의
{: #spring-fallback}

메소드에 회로 차단 동작을 추가하려면 `@HystrixCommand`으로 어노테이션을 작성합니다. Spring은 `@EnableCircuitBreaker`라는 어노테이션이 있는 애플리케이션 내에서 이러한 메소드를 찾고 Hystrix 기능으로 랩핑하여 오류/제한시간 초과를 모니터하고 필요에 따라 적절한 대체 동작을 호출합니다. 

`@HystrixCommand`는 `@Component` 또는 `@Service`의 메소드에서만 지원됩니다.
{: note}

다음 `@HystrixCommand` 어노테이션은 `service()` 호출을 랩핑하여 회로 차단 동작을 제공합니다. `service()` 메소드가 실패하거나 회로가 열려 있는 경우 프록시가 `fallback()` 메소드를 호출합니다.

```java
@Autowired
private MicroService myService;

@GetMapping("/endpoint")
public String value() throws Exception {
    return "Service returned: " + myService.service();
}

@Service
class MicroService {
    @HystrixCommand(fallbackMethod = "fallback")
    public String service() {
        // Do some processing. If an exception is thrown
        // the fallback method will be called
        return "Success";
    }

    public String fallback(Throwable e) {
        return "Fallback! Reason: " + e.getMessage() + "\n";
    }
}
```
{: codeblock}

Spring 기반 [Hystrix Circuit Breaker](https://spring.io/guides/gs/circuit-breaker/){: new_window} ![외부 링크 아이콘](../icons/launch-glyph.svg "외부 링크 아이콘") 예를 참조하여 자세히 보십시오.
{: tip}

### 제한시간 사용
{: #spring-timeout}

원격 서비스를 호출하는 경우 바로 리턴되지 않거나 시간이 조금 걸리거나 아주 오래 걸릴 수도 있습니다. Hystrix는 애플리케이션에 허용 가능한 내용을
정의하는 기능을 제공합니다. 단순히 `HystrixCommand` 어노테이션에 추가하기만 하면 기본값 1초라는 제한시간 값을 변경할 수 있습니다.

```java
@HystrixCommand(
    fallbackMethod = "fallback",
    commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "30000"),
    }
)
```
{: codeblock}

### 벌크헤드 사용
{: #spring-bulkhead}

Hystrix는 세마포어와 큐 기반 벌크헤드 둘 다 지원합니다. 다음 스니펫은 4개의 스레드를 할당하고 미해결 요청 수를 10으로 제한하는 큐 기반 벌크헤드를 구성하는 방법을 보여줍니다.

```java
@HystrixCommand(
    fallbackMethod = "fallback",
    threadPoolProperties = {
        @HystrixProperty(name = "coreSize", value = "4"),
        @HystrixProperty(name = "maxQueueSize", value = "10")
    }
)
```
{: codeblock}

### 회로 차단기 상태
{: #spring-breaker-status}

Hystrix Spring Starter에는 애플리케이션에 대한 기본 `/health` 엔드포인트를 개선하는 묘안이 하나 더 있습니다(Spring Actuator를 통해 제공됨, 세부사항은 [메트릭 주제](/docs/java?topic=java-spring-metrics#spring-metrics) 참조).

상태 엔드포인트가 [추가 세부사항을 포함하도록 구성](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html#production-ready-health){: new_window} ![외부 링크 아이콘](../icons/launch-glyph.svg "외부 링크 아이콘")되는 경우, 회로 차단기 상태가 상태 검사 정보에 포함됩니다. (이 동작은 기본적으로 사용 불가능합니다.)

```
{
    "hystrix": {
        "openCircuitBreakers": [
            "MicroService::service"
        ],
        "status": "CIRCUIT_OPEN"
    },
    "status": "UP"
}
```
{: screen}

## 다음 단계
{: #spring-tolerance-next-steps notoc}

Hystrix 구성에 대한 자세한 정보는 [Hystrix Configuration Wiki](https://github.com/Netflix/Hystrix/wiki/Configuration){: new_window} ![외부 링크 아이콘](../icons/launch-glyph.svg "외부 링크 아이콘")를 참조하십시오.

Spring을 사용하는 Hystrix에 대한 자세한 정보는 다음을 참조하십시오.

* [A Guide to Spring Cloud Netflix](https://www.baeldung.com/spring-cloud-netflix-hystrix){: new_window} ![외부 링크 아이콘](../icons/launch-glyph.svg "외부 링크 아이콘")
* [Spring Circuit Breaker Guide](https://spring.io/guides/gs/circuit-breaker/){: new_window} ![외부 링크 아이콘](../icons/launch-glyph.svg "외부 링크 아이콘")