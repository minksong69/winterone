# winterone

역할분담

1
Rental Book 기준으로 서비스 시나리오 헥사고날 아키텍쳐 다이어그램 
-> 소민아 수석

2
ckeck points
-saga : pub/sub 구현
-cqrs : view 스티커 Mypage (그린스트커)
-correlation (서로관련있는 키)
req/resp Sync 호출
gateway : ingress gateway , Spring cloud gateway를 설치하던
polyglot : 주문서비스는 h2, 배송서비스는 mysql
-> 김기웅 선임 , 손원석 수석

3
deploy/pipeline : yaml 기반 이미지만든다음 yaml로 해도됨
-> 신은경 수석

4
Circuit Breaker :
autoscale(hpa) : scale out되는 것 확인
-> 성성기 수석

5
zero-downtime deploy(readness probe) : avail한것을 부하테스트로 보여줘야함
self-healing(Liveness Probe)
-> 손원석 수석

6
config map/persistence volume :마이크로 서비스내에서 활용해야함 
-> 김금 선임

참조 git 주소

https://github.com/HorangApple/rentalbook

https://github.com/phone82

# 구현
ㅁㄴㅇ
