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
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084, 8088 이다)
```
cd SirenOrder
mvn spring-boot:run  

cd Payment
mvn spring-boot:run

cd SirenOrderHome
mvn spring-boot:run 

cd Shop
mvn spring-boot:run  

cd gateway
mvn spring-boot:run  
```

## DDD 의 적용
msaez.io 를 통해 구현한 Aggregate 단위로 Entity 를 선언 후, 구현을 진행하였다.

Entity Pattern 과 Repository Pattern 을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

**SirenOrder 서비스의 SirenOrder.java**

```java 
package winterschoolone;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

import winterschoolone.external.Payment;
import winterschoolone.external.PaymentService;

import java.util.List;

@Entity
@Table(name="SirenOrder_table")
public class SirenOrder {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String userId;
    private String menuId;
    private Integer qty;
    private String status;

    @PostPersist
    public void onPostPersist(){
    	Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        Payment payment = new Payment();
        payment.setOrderId(this.getId());
        payment.setMenuId(this.menuId);
        payment.setQty(this.getQty());
        payment.setUserId(this.getUserId());
        // mappings goes here
        SirenOrderApplication.applicationContext.getBean(PaymentService.class)
        .pay(payment);
    }

    @PostUpdate
    public void onPostUpdate(){
        Updated updated = new Updated();
        BeanUtils.copyProperties(this, updated);
        updated.publishAfterCommit();


    }

    @PreRemove
    public void onPreRemove(){
        OrderCancelled orderCancelled = new OrderCancelled();
        BeanUtils.copyProperties(this, orderCancelled);
        orderCancelled.publishAfterCommit();


    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }
    public String getMenuId() {
        return menuId;
    }

    public void setMenuId(String menuId) {
        this.menuId = menuId;
    }
    public Integer getQty() {
        return qty;
    }

    public void setQty(Integer qty) {
        this.qty = qty;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    
}
```

**SirenOrder 서비스의 PolicyHandler.java**
```java
package winterschoolone;

import winterschoolone.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.List;
import java.util.Optional;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }
    
    @Autowired
	SirenOrderRepository sirenOrderRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverAssigned_(@Payload Assigned assigned){

        if(assigned.isMe()){
        	Optional<SirenOrder> optional = sirenOrderRepository.findById(assigned.getOrderId());
        	if(optional != null && optional.isPresent())
        	{
        		SirenOrder sirenOrder = optional.get();
        		
        		sirenOrder.setStatus("Assigned");
                // view 객체에 이벤트의 eventDirectValue 를 set 함
                // view 레파지 토리에 save
            	sirenOrderRepository.save(sirenOrder);
        	}
            
            System.out.println("##### listener  : " + assigned.toJson());
        }
    }

}
```

- DDD 적용 후 REST API의 테스트를 통하여 정상적으로 동작하는 것을 확인할 수 있었다.  
  
- 원격 주문 (SirenOrder 동작 후 결과)

![증빙1](https://user-images.githubusercontent.com/53815271/107907569-64fd5180-6f97-11eb-9f1e-cb1fb97fd4ff.png)

# GateWay 적용
API GateWay를 통하여 마이크로 서비스들의 집입점을 통일할 수 있다.
다음과 같이 GateWay를 적용하였다.

```yaml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: SirenOrder
          uri: http://localhost:8081
          predicates:
            - Path=/sirenOrders/** 
        - id: Payment
          uri: http://localhost:8082
          predicates:
            - Path=/payments/** 
        - id: Shop
          uri: http://localhost:8083
          predicates:
            - Path=/shops/** 
        - id: SirenOrderHome
          uri: http://localhost:8084
          predicates:
            - Path= /sirenOrderHomes/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: SirenOrder
          uri: http://SirenOrder:8080
          predicates:
            - Path=/sirenOrders/** 
        - id: Payment
          uri: http://Payment:8080
          predicates:
            - Path=/payments/** 
        - id: Shop
          uri: http://Shop:8080
          predicates:
            - Path=/shops/** 
        - id: SirenOrderHome
          uri: http://SirenOrderHome:8080
          predicates:
            - Path= /sirenOrderHomes/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080

```

# CQRS
Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능하게 구현해 두었다.
본 프로젝트에서 View 역할은 SirenOrderHomes 서비스가 수행한다.

- 주문(ordered) 실행 후 SirenOrderHomes 화면

![증빙2](https://user-images.githubusercontent.com/53815271/107907619-7e060280-6f97-11eb-89b3-4e3236ff9ddd.png)

- 주문(OrderCancelled) 취소 후 SirenOrderHomes 화면

![증빙3](https://user-images.githubusercontent.com/53815271/107908429-5e6fd980-6f99-11eb-8afc-2a2c070a1663.png)

위와 같이 주문을 하게되면 SirenOrder -> Payment -> Shop -> SirenOrder 로 주문이 Assigend 되고

주문 취소가 되면 Status가 refunded로 Update 되는 것을 볼 수 있다.

또한 Correlation을 key를 활용하여 orderId를 Key값을 하고 원하는 주문하고 서비스간의 공유가 이루어 졌다.

위 결과로 서로 다른 마이크로 서비스 간에 트랜잭션이 묶여 있음을 알 수 있다.

# 폴리글랏

Shop 서비스의 DB와 SirenOrder의 DB를 다른 DB를 사용하여 폴리글랏을 만족시키고 있다.

**Shop의 pom.xml DB 설정 코드**

![증빙5](https://user-images.githubusercontent.com/53815271/107909600-e2c35c00-6f9b-11eb-8ec4-e8ef46c07949.png)

**SirenOrder의 pom.xml DB 설정 코드**

![증빙4](https://user-images.githubusercontent.com/53815271/107909551-d17a4f80-6f9b-11eb-8af2-71b4d0112206.png)

# 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(SirenOrder)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

**SirenOrder 서비스 내 external.PaymentService**
```java
package winterschoolone.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="Payment", url="${api.url.Payment}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```

**동작 확인**
