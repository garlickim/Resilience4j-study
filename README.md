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
    
