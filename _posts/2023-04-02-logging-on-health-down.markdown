
# Health 응답결과를 로그로 출력하기

쿠버네티스 환경에서 Health 결과 OUT_OF_SERVICE 응답을 받은 경우가 종종 발생했다. 그 시점 무슨 문제점이 있었는지 로그를 찾을 수 없어, "/actuator/healt" 결과가 비정상이면 Health Details를 로그로 남기도록 한다.

## 분석

"/actuator/health" 응답을 준비하는 과정에서 logging 하는 부분이 있다면, log level을 설정하면 되겠지만 없었다.
그래서, 다른 방법을 찾아보기로 했다.

health 체크는 "org.springframework.boot.actuate.health.HealthIndicator" 를 상속받아 구현한 빈을 수집해서 이를 모두 호출한다. 이 지점을 spring aop로 애스팩트 했다.

### 1. HealthIndicator Interface를 타켓으로 포인트컷하기 위해 `@within`과 `+`를 사용한다.

  ```java
    @Around(value = "within(org.springframework.boot.actuate.health.HealthIndicator+) && execution(* getHealth(..))")
    public Health writeSuccessLog(ProceedingJoinPoint pjp) throws Throwable {
  ```

### 2. getHealth() 내부에서 Heath.withoutDetails() 으로 상세 사유가 지워지는 문제 해결한다.

HealthIndicator 내부를 보면 `default Health getHealth(boolean includeDetails)` 에서 `health()` 를 호출하는데 health() 결과에서는 상세사유가 지워지지 않았으니 이 메쏘드를 포인트컷 해 보았으나, 두가지 문제점이 있었다.

1. default 구현하지 않으면 사용한다는 뜻으로, 구현체에서 오버라이딩했다면 `health()`가 호출된다는 보장이 없다.

2. `getHealth(..)`에서 `health()`가 호출되었더라도, 동일 클래쓰 내에서 호출되어 어드바이스가 수행되지 못한다. 스프링 aop 은 빈을 프록싱하므로, HealthIndicator 클래쓰로 처음 진입하는 getHealth() 직전에는 어드바이스를 수행하지만, 이미 클래쓰 내부로 들어와서 `health()`로 호출될떄는 프록싱이 없어 어드바이스를 수행하지 못한다.

결국은 몌소드 `getHealth()`를 조인포인트로 around 어드바디이스를 수행한다. around를 사용하면 입력매개변수와 결과값을 모두 조작이 가능하므로, `includeDetails`를 무조건 true로 전달하여 health details를 포함하여 로그를 출력하고, 원래의 파라미터에 따라 필요시 상세제거해 응답을 전달해주면 된다.

참고로, Health에는 무엇에 대한 Health인지가 알수없어 aop target class 로 확인 해야한다.

  ```java
    @Around(value = "within(org.springframework.boot.actuate.health.HealthIndicator+) && execution(* getHealth(..))")
    public Health writeSuccessLog(ProceedingJoinPoint pjp) throws Throwable {

        Object[] args = pjp.getArgs();
        boolean includeDetails = (boolean) args[0];

        Object[] newArgs = { true };
        Health health = (Health) pjp.proceed(newArgs);

        Status status = health.getStatus();
        if(Status.DOWN.equals(status) || Status.OUT_OF_SERVICE.equals(status)) {
            log.error("{} {}", pjp.getTarget().getClass(), health);
        }

        return includeDetails ? health : new HealthAccessor(health).withoutDetails();
    }
  ````
