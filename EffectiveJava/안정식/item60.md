# 정확한 답이 필요하다면 float와 double은 피하라

- float 과 double 타입은 과학 및 공학 계산용으로 설계되어있다.
- float 과 double 타입은 금융 관련 계산과는 맞지 않는다.
- 금융 계산에는 BigDecimal, intm long을 사용해야한다.
  - BingDecimal 은 2가지 단점을 가지고 있다.
    - 느리다
    - 기본타입보다 쓰기가 불편하다
    - 10진수로 표현할 수 있으면 int와 long의 속도가 상대적으로 빠르다.