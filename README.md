# winterone
# 서비스 시나리오
- 기능적 요구사항
1. 고객이 음료를 주문한다.
2. 고객이 결제를 한다.
3. 결제가 완료되면 주문내역을 매장으로 보낸다.
4. 매장에서 주문을 할당한다. 
5. 고객이 주문을 취소할 수 있다.
6. 고객이 중간중간 주문상태를 조회한다.


- 비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지않으면 주문이 진행되지 않는다 → Sync 호출
1. 장애격리
    1. 결제시스템에서 장애가 발생해도 주문취소는 24시간 받을 수 있어야한다 → Async (event-driven), Eventual Consistency
    1. 주문량이 많아 매장시스템이 과중되면 잠시 주문할당을 하지 않고 잠시후에 하도록 유도한다 → Circuit breaker, fallback
1. 성능
    1. 고객이 주문상태를 SirenOrderHome에서 확인 할 수 있어야 한다. → CQRS 

# Event Storming 결과

![EventStormingV1](https://user-images.githubusercontent.com/53815271/107929718-66417500-6fbd-11eb-9754-5dd92a06afe5.png)

# 헥사고날 아키텍처 다이어그램 도출

![폴리글랏 아키텍처](https://user-images.githubusercontent.com/53815271/107929578-2f6b5f00-6fbd-11eb-9176-ac3455d5c7be.png)


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
- 잠시 Payment 서비스 중시

![증빙6](https://user-images.githubusercontent.com/53815271/107910391-a85abe80-6f9d-11eb-8dd5-6b7a4d1cdc01.png)

- 주문 요청시 에러 발생

![증빙7](https://user-images.githubusercontent.com/53815271/107910392-a8f35500-6f9d-11eb-98e4-2cf9fa2fbd46.png)

- Payment 서비스 재기동 후 정상동작 확인

![증빙8](https://user-images.githubusercontent.com/53815271/107910393-a98beb80-6f9d-11eb-833f-150d11f51067.png)
![증빙9](https://user-images.githubusercontent.com/53815271/107910394-a98beb80-6f9d-11eb-841c-aa6ab38cf99b.png)

# 운영
.
