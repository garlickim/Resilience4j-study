# Resilience4j-study

## Resilience4J란?  
Hystrix에서 영감을 받아 생성된 가벼운 라이브러리로, Hystrix와 달리 Vavr외의 다른 라이브러리의 의존성이 없음.

<br>

## 등장배경  
Netflix Hystrix가 maintenance로 바뀌면서, Resilience4j를 사용하도록 권장

<br>

## Core Modules

### CircuitBreaker
<details>
<summary>더보기</summary>
<div markdown="1">
 
  * 3가지의 정상 상태(CLOSED, OPEN, HALF_OPEN)와 특수 상태(DISABLED, FORCED_OPEN)로 구성된 유한 상태 머신(finite state machine)로 구현 됨  
  * sliding window를 사용하여 호출 결과를 집계 및 저장
  * count-based 과 time-based, 두 종류의 sliding window가 존재 <br><br>

     <b> sliding window란? </b>  
     * count-based sliding window  
        - 마지막 N번 호출 결과를 집계  
        - N개의 원형 배열로 구현 됨  
        - size=10 이면 measurements도 항상 10  
        - N개의 요청을 저장하고 aggregation 함  
        - total aggregation은 새로운 결과가 기록되면 가장 오래된 measurement이 제거되며, 업데이트 됨  
        - 스냅샷 검색시간 O(1)  
        - 메모리 소비는 O(N) 
        
    * time-based sliding window
        - 마지막 N초 호출 결과를 집계  
        - N개의 부분 집합(버킷) 원형 배열로 구현 됨  
        - size-10 이면 partial aggregation(bucket)도 항상 10  
        - head bucket 은 현재 초의 호출 결과에 대한 aggregation 저장  
        - 나머지 bucket은 나머지 초의 호출 결과에 대한 aggregation 저장  
        - total aggregation은 새로운 결과가 기록되면 가장 오래된 bucket이 제거되며, 업데이트 됨  
        - 스냅샷 검색시간 O(1)
        - 메모리 소비는 O(N)
    <br><br>
    
     <b> Failure rate and slow call rate threshold (실패율과 느린 호출률의 임계치)</b>  
     * CircuitBreaker의 상태는 실패율 또는 느린 호출률이 임계치와 같거나 그 이상일 때, OPEN 으로 변할 것
     * 기본적으로 모든 Exception은 실패로 집계
     * 실패율에 집계할 Exception을 정의할 수 있음
     * 실패/성공으로 집계하지 않는 Exception도 정의할 수 있음
     * 최소 호출수를 정의해야 실패율과 느린 호출률이 계산 됨
     * Circuit이 OPEN되면 CallNotPermittedException 발생하며, 호출을 거부
     * DISABLED, FORCED_OPEN 싱태에서는 Circuit Breaker events 가 발생하지 않으며, Metrics이 기록되지 않음  <br><br>
     
 * Circuit Breaker는 Thread Safe 함
     * Circuit Breaker 상태는 AtomicReference에 저장 됨  
     * atomic operations을 사용하여 상태를 업데이트 함
     * Sliding Window로 부터 snapshot을 읽거나 호출을 기록하는 것은 동기화 됨   
        --> 함수 호출을 동기화 하는 것은 아님
     * 오직 하나의 thread만 상태 및 sliding window를 update 할 수 있음
     * sliding window size = 15 라 해서, 15개의 호출만 동시 실행되는 것은 아님  
     * concurrent threads 수를 제한하고자할 시, Bulkhead 사용
     
 * CircuitBreakerRegistry  
     * Thread safe와 atomicity guarantee를 제공하는 ConcurrentHashMap의 기반 in memory CircuitBreakerRegistry가 제공 됨  
     * CircuitBreaker instance를 관리함(create and retrieve)
     
 * CircuitBreakerConfig  
     * CircuitBreakerConfig builder를 사용하여 custom한 global configration을 할 수 있음
     * | property | default | description |  
       |----------|---------------|-------------------|
       | failureRateThreshold | 50 | 실패율 임계치(%) |
       | slowCallRateThreshold | 100 | 느린 호출률 (%) <br> 100보다 크면 circuit open |
       | slowCallDurationThreshold | 60000 | 느린 호출로 판단하는 초 [ms] <br> slowCallRate를 증가 시킴 |
       | permittedNumberOfCallsInHalfOpenState | 10 | HalfOpenState 상태에서 허용되는 호출 수 |
       | slidingWindowType | COUNT_BASED | sliding window type <br> COUNT_BASED or TIME_BASED |
       | slidingWindowSize | 100 | 에러율을 계산하기 위한 최소 호출 수 | 
       | waitDurationInOpenState | 60000 | OPEN 에서 HALF OPEN으로 가기 전 CircuitBreaker의 대기 시간 |
       | automaticTransitionFromOpenToHalfOpenEnabled | false | true일 경우, waitDurationInOpenState 시간 이후 자동으로 OPEN에서 HALF OPEN으로 transition 됨 |
       | recordExceptions | empty | 실패로 기록할 Exception list |
       | ignoreExceptions | empty | 실패/성공으로 기록하지 않을 Exception list <br> recordExceptions에 있는 Exception일지라도 기록하지 않음 |


</div>
</details> 

### Bulkhead
<details>
<summary>더보기</summary>
<div markdown="1">  
  
  * 동시 실행(concurrent execution) 수를 제한할 수 있는 bulkhead pattern의 2가지 구현체를 제공  
       * SemaphoreBulkhead  
          - Semaphores 기반  
          - 다양한 Threading과 I/O model에서 잘 작동함  
          - "shadow" thread pool 옵션을 제공하지 않음  
          - correct thread pool size를 보장하는 것은 client의 책임 
            
       * FixedThreadPoolBulkhead 
          - bounded queue와 fixed thread pool 사용
  
  * BulkheadRegistry
      * in memory BulkheadRegistry 와 ThreadPoolBulkheadRegistry 제공
      * ThreadPoolBulkheadRegistry은 Bulkhead instance를 관리(create and retrieve)하는데 사용할 수 있음
 
  * BulkheadConfig
      * BulkheadConfig builder를 사용하여 custom한 global configration을 할 수 있음
      * | property | default | description |  
        |----------|---------------|-------------------|
        | maxConcurrentCalls | 25 | bulkhead에 의해 허용되는 최대 병렬 실행(parallel executions)량 |
        | maxWaitDuration | 0 | bulkhead가 가득찼을 때, 진입하고자 하는 Thread를 차단해야하는 최대 시간 |
        
  * ThreadPoolBulkheadConfig
      * ThreadPoolBulkheadConfig builder를 사용하여 custom한 global configration을 할 수 있음
      * | property | default | description |  
        |----------|---------------|-------------------|
        | maxThreadPoolSize | Runtime.getRuntime().availableProcessors() | Max thread pool size |
        | coreThreadPoolSize | Runtime.getRuntime().availableProcessors() - 1 | Core thread pool size | 
        | queueCapacity | 100 | Capacity of the queue |
        | keepAliveDuration | 20 | Thread가 Core수 보다 많을 때, idle thread가 종료되기 전 새 task를 기다리는 최대 시간[ms] |
                    
</div>
</details>

### RateLimiter  
<details>  
<summary>더보기</summary>      
<div markdown="1">  
  
  * 서비스의 고가용성 및 안정성을 수립하고 확장 가능한 API를 준비하기 위한 필수 기술  
  
  * 초과 요청에 대해서 거절하거나 나중에 실행하기 위한 Queue 생성, 또는 앞의 두가지 방식을 결합하여 사용 가능  
  
  * RateLimiter의 구현체
      * AtomicRateLimiter (AtomicReference 통해 state를 관리, Default)
      * SemaphoreBasedRateLimiter <br><br>
      * limitRefreshPeriod 후에 permissions refresh 하는 스케쥴러 포함
      * AtomicRateLimiter 구현체의 경우 RateLimiter가 사용되지 않는 경우 refresh를 skip 할 수 있도록 최적화 되어 있음
  
  * AtomicRateLimiter.State는 완전히 변경할 수 없음  
      * activeCycle - 마지막 호출에서 사용된 Cycle number  
      * activePermissions - 마지막 호출 후 사용 가능한 permission 수 (Can be negative if some permissions were reserved)  
      * nanosToWait - 마지막 호출에 대한 permission을 기다리는 nanoseconds 수
      
  * SemaphoreBasedRateLimiter
      * Semaphores 사용
    
  * RateLimiterRegistry
      * in memory RateLimiterRegistry는 RateLimite instance를 관리(create and retrieve)하는데 사용할 수 있음
      
  * RateLimiterConfig 
      * RateLimiterConfig builder를 사용하여 custom한 global configration을 할 수 있음
      * | property | default | description |  
        |----------|---------------|-------------------| 
        | timeoutDuration | 5 | thread가 permission을 기다리는 대기 시간[s] |
        | limitRefreshPeriod | 500 | limit refresh 기간이 지나면 rate limiter는 permission count를 limitForPeriod 값으로 재설정[ns] |
        | limitForPeriod | 50 | 한번의 limit refresh 기간 동안 사용 가능 한 permission 수 |
        
  * Runtime 시점에 rate limiter params을 변경하기 위해, changeTimeoutDuration와 changeLimitForPeriod를 사용 가능함
      * 현재 대기하고 있는 Thread 또는 현재 period permissions에 영향을 미치지 않음
      

</div>
</details>

### Retry

<details>  
<summary>더보기</summary>      
<div markdown="1">        
 
   * RetryRegistry
      * in memory RetryRegistry는 Retry instance를 관리(create and retrieve)하는데 사용할 수 있음

   * RetryConfig
      * RetryConfig builder를 사용하여 custom한 global configration을 할 수 있음
      * | property | default | description |  
        |----------|---------------|-------------------| 
        | maxAttempts | 3 | 최대 Retry 횟수 |
        | waitDuration | 500 | Retry 사이의 fixed 대기 시간[ms] |
        | intervalFunction | numOfAttempts -> waitDuration | Retry 실패 후, 대기 interval을 수정하는 function <br> 기본적으로, wait duration은 일정하게 유지 |
        | retryOnResultPredicate | result -> false | result를 Retry 해야하는지 판단하는 predicate <br> true : 재시도 |
        | retryOnExceptionPredicate | throwable -> true | Exception을 Retry 해야하는지 판단하는 predicate <br> true : 재시도 |
        | retryExceptions | empty | 실패로 기록되고, Retry 할 Exception list | 
        | ignoreExceptions | empty | ignore하고 Retry 하지 않는 Exception list |
          
   * IntervalFunction
      * Retry 사이에 fixed 대기 시간을 사용하지 않고, 모든 Retry에 대한 대기 시간을 계산하는데 IntervalFunction을 사용
      * IntervalFunction 생성을 간단하게 할 수 있는 factory method를 제공
         ~~~ java
         IntervalFunction defaultWaitInterval = IntervalFunction.ofDefaults();

         // This interval function is used internally when you only configure waitDuration  
         IntervalFunction fixedWaitInterval = IntervalFunction.of(Duration.ofSeconds(5));

         IntervalFunction intervalWithExponentialBackoff = IntervalFunction.ofExponentialBackoff();

         IntervalFunction intervalWithCustomExponentialBackoff 
                              = IntervalFunction.ofExponentialBackoff(IntervalFunction.DEFAULT_INITIAL_INTERVAL, 2d);

         IntervalFunction randomWaitInterval = IntervalFunction.ofRandomized();

         // Overwrite the default intervalFunction with your custom one
         RetryConfig retryConfig = RetryConfig.custom()
                                              .intervalFunction(intervalWithExponentialBackoff)
                                              .build();
         ~~~

</div>
</details>
