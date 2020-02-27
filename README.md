# Resilience4j-study

## Resilience4J란?  
Hystrix에서 영감을 받아 생성된 가벼운 라이브러리로, Hystrix와 달리 Vavr외의 다른 라이브러리의 의존성이 없음.

<br>

## 등장배경  
Netflix Hystrix가 maintenance로 바뀌면서, Resilience4j를 사용하도록 권장

<br>

## Core Modules

### CircuitBreaker
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
        - 메모리 소비는 O(N) <br><br>
        
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
     * CircuitBreaker의 상태는 실패율 또는 느린 호출률이 임계치와 같거나 그 이상일 때, OPEN 으로 변할 것.
     * 기본적으로 모든 Exception은 실패로 집계
     * 실패율에 집계할 Exception을 정의할 수 있음
     * 실패/성공으로 집계하지 않는 Exception도 정의할 수 있음
     * 최소 호출수를 정의해야 실패율과 느린 호출률이 계산 됨
     * Circuit이 OPEN되면 CallNotPermittedException 발생하며, 호출을 거부
     * DISABLED, FORCED_OPEN 싱태에서는 Circuit Breaker events 가 발생하지 않으며, Metrics이 기록되지 않음
