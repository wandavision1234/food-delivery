## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 

```
package yanolza;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="PaymentHistory_table")
public class PaymentHistory {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private Long cardNo;
    private String status;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }
    public Long getCardNo() {
        return cardNo;
    }

    public void setCardNo(Long cardNo) {
        this.cardNo = cardNo;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package yanolza;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="paymentHistories", path="paymentHistories")
public interface PaymentHistoryRepository extends PagingAndSortingRepository<PaymentHistory, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# order 서비스의 주문처리
http localhost:8081/orders name="KimGangMin" cardNo=0 status="order started"

# payment 서비스의 결제처리
http localhost:8088/paymentHistories orderId=1 cardNo=0000

# reservation 서비스의 예약처리
http localhost:8088/reservations orderId=3 status="confirmed"

# 주문 상태 확인
http localhost:8081/orders/3

```


## Polyglot Persistence

Polyglot Persistence를 위해 h2datase를 hsqldb로 변경

```
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>
<!--
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
-->

# 변경/재기동 후 예약 주문
http localhost:8081/orders name="lee" cardNo=1 status="order started"

HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Wed, 18 Aug 2021 09:41:30 GMT
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "cardNo": 1,
    "name": "lee",
    "status": "order started"
}


# 저장이 잘 되었는지 조회
http localhost:8081/orders/1

HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8    
Date: Wed, 18 Aug 2021 09:42:25 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "cardNo": 1,
    "name": "lee",
    "status": "order started"
}
```

## CQRS

CQRS 구현을 위해 고객의 예약 상황을 확인할 수 있는 Mypage를 구성.

```
# mypage 호출 
http localhost:8081/mypages/12

HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 18 Aug 2021 09:46:13 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/2"
        },
        "self": {
            "href": "http://localhost:8084/mypages/2"
        }
    },
    "cancellationId": null,
    "name": "kim",
    "orderId": 2,
    "reservationId": 2,
    "status": "Reservation Complete"
}
```


## 동기식 호출 과 Fallback 처리

주문(Order)->결제(Payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
#PaymentHistoryService.java

package yanolza.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="${api.payment.url}")
public interface PaymentHistoryService {
    @RequestMapping(method= RequestMethod.POST, path="/paymentHistories")
    public void pay(@RequestBody PaymentHistory paymentHistory);

}
```

- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        yanolza.external.PaymentHistory paymentHistory = new yanolza.external.PaymentHistory();
        // mappings goes here
        //PaymentHistory payment = new PaymentHistory();
        System.out.println("this.id() : " + this.id);
        paymentHistory.setOrderId(this.id);
        paymentHistory.setStatus("Reservation Good");
        paymentHistory.setCardNo(this.cardNo);      
        
        
        OrderApplication.applicationContext.getBean(yanolza.external.PaymentHistoryService.class)
            .pay(paymentHistory);
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 결제 (payment) 서비스를 잠시 내려놓음 (ctrl+c)

# 주문요청
http http://localhost:8088/orders name="me" cardNo=123 status="Order Start"
 
HTTP/1.1 500 Internal Server Error
Content-Type: application/json;charset=UTF-8
Date: Wed, 18 Aug 2021 09:52:24 GMT
transfer-encoding: chunked

{
    "error": "Internal Server Error",
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction",
    "path": "/orders",
    "status": 500,
    "timestamp": "2021-08-18T09:52:24.229+0000"
}

# 결제 (payment) 재기동
mvn spring-boot:run

#주문처리
http http://localhost:8088/orders name="me" cardNo=123 status="Order Start"

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Wed, 18 Aug 2021 09:54:24 GMT
Location: http://localhost:8081/orders/3    
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/3"
        },
        "self": {
            "href": "http://localhost:8081/orders/3"
        }
    },
    "cardNo": 123,
    "name": "me",
    "status": "Order Start"
}
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제가 이루어진 후에 상점시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 상점 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package yanolza;

 ...
    @PostPersist
    public void onPostPersist(){
        PaymentApproved paymentApproved = new PaymentApproved();
        
	paymentApproved.setStatus("Pay OK");
        
	BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();

    }
```
- 상점 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package yanolza;

...

@Service
public class PolicyHandler{
    @Autowired ReservationRepository reservationRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_AcceptReserve(@Payload PaymentApproved paymentApproved){

        if(!paymentApproved.validate()) return;

        System.out.println("\n\n##### listener AcceptReserve : " + paymentApproved.toJson() + "\n\n");
	
        Reservation reservation = new Reservation();
        reservation.setStatus("Reservation Complete");
        reservation.setOrderId(paymentApproved.getOrderId());
        reservation.setId(paymentApproved.getOrderId());
        reservationRepository.save(reservation);
       

    }
```

예약 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 예약시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# 예약 서비스 (reservation) 를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders name="ChooChoo" cardNo=27 status="order started"

#주문상태 확인
http localhost:8081/orders/3      # 주문정상

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/3"
        },
        "self": {
            "href": "http://localhost:8081/orders/3"
        }
    },
    "cardNo": 27,
    "name": "ChooChoo",
    "status": "order started"
}
	    
#예약 서비스 기동
cd reservation
mvn spring-boot:run

#주문상태 확인
http localhost:8084/mypages     # 예약 상태가 "Reservation Complete"으로 확인

 {
                "_links": {
                    "mypage": {
                        "href": "http://localhost:8084/mypages/2"
                    },
                    "self": {
                        "href": "http://localhost:8084/mypages/2"
                    }
                },
                "cancellationId": null,
                "name": "ChoiJung",
                "orderId": 2,
                "reservationId": 1,
                "status": "Reservation Complete"
            }
```

