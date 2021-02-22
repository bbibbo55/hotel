# 호텔 예약

> Winter School 2팀 Workspace입니다.

## 시나리오

호텔 예약 시스템에서 요구하는 기능/비기능 요구사항은 다음과 같습니다. 사용자가 예약과 함께 결제를 진행하고 나면 객실 관리자가 객실을 배정하는 시스템입니다. 이 과정에 대해서 고객은 진행 상황을 확인할 수 있고, 카카오톡으로 알림을 받을 수 있습니다. 

#### 기능적 요구사항

1. 고객이 원하는 객실을 선택 하여 예약한다.
2. 고객이 결제 한다.
3. 예약이 신청 되면 예약 신청 내역이 호텔에 전달 된다.
4. 호텔이 확인 하여 예약을 확정 한다.
5. 고객이 예약 신청을 취소할 수 있다.
6. 예약이 취소 되면 호텔 예약이 취소 된다.
7. 고객이 예약 진행 상황을 중간 중간 조회 한다.
8. 예약 상태가 바뀔 때 마다 카카오톡으로 알림을 보낸다.
9. 고객이 예약 취소를 하면 예약 정보는 삭제되나, 객실 관리팀에서는 이력 관리를 위해 취소 장부를 별도 저장한다.

#### 비 기능적 요구사항

1. 트랜잭션
   - 결제가 되지 않은 예약건은 아예 호텔 예약 신청이 되지 않아야 한다. `Sync 호출`

2. 장애격리
   - 객실관리 기능이 수행 되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. `Pub/Sub`
   - 결제 시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도 한다.
     (장애처리)

3. 성능
   - 고객이 예약 확인 상태를 마이페이지에서 확인할 수 있어야 한다. `CQRS`
   - 예약 상태가 바뀔 때 마다 카톡 등으로 알림을 줄 수 있어야 한다.
   
# 분석/설계

## Event Storming

#### 초기 버전

1. 잘못된 이벤트 제거 : "호텔선택됨", "룸타입선택됨" 등 UI 이벤트 제거 
pp의 Order, store 의 주문처리, 결제의 결제이력은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌
2. order의 주문, reservation의 예약과 취소, payment의 결제 이력, 고객관리의 카카오톡 알림 등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌(바운디드 컨텍스트)
3. 도메인 서열 분리 
   - Core Domain: Order, Reservation(없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.7% 목표, 배포주기는 1주일 1회 미만)
   - Supporting Domain: customermanagement(경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율)
   - General Domain: payment(결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높을 것으로 판단. 향후 External로 전환 필요)

![screenshot-miro com-2020 12 18-09_30_35](https://user-images.githubusercontent.com/76149887/102559719-e6f60900-4113-11eb-96c6-54829db72270.png)

#### 2020.12.21

- 1차 완성본

![screenshot-miro com-2020 12 21-13_50_26](https://user-images.githubusercontent.com/76149887/102837515-453a2900-443f-11eb-8293-e7f46bc94732.png)


#### 2020.12.22

- 예약 취소 건에 대한 추가 의견으로, 예약 취소는 이력 정보만 남기고 실제 예약 정보는 삭제하는 것으로 합의
- 예약 취소 시, 예약 취소에 대한 이력만을 저장하기 때문에 이 이력의 저장에 대한 동기 호출로 결제 취소를 하지 않고, 비동기 호출로 전환.

![screenshot-miro com-2020 12 22-10_19_41](https://user-images.githubusercontent.com/76149887/102837538-51be8180-443f-11eb-9fbc-00bec667f053.png)

#### 2021.01.22

- 오더 속성 변경(호텔 서비스인 것을 좀 더 명확하게 드러내려고 호텔아이디와 룸타입으로 변경)
- 호텔이 확인 하여 예약을 확정한다는 기능 요구사항을 표현 하기 위해 액터(호텔리어), 커맨드(컨펌) 추가. 뷰는 기존의 리저베이션 디테일 위치 이동.  

<img width="643" alt="miro_20210122" src="https://user-images.githubusercontent.com/58290368/105454042-9ea6a980-5cc4-11eb-95bd-439831fc6575.png">

#### 2021.01.25

- MSAEZ 툴에서 이벤트스토밍 작업
- 사용자 입장에서 꼭 필요할 때 카카오톡 알림을 받을 수 있도록 

<img width="1346" alt="hotel_msaez_20210125" src="https://user-images.githubusercontent.com/58290368/105668328-70250a80-5f20-11eb-9982-c26b1f9b1fd4.png">

### 기능 요구사항을 커버하는지 검증
1. 고객이 원하는 객실을 선택 하여 예약한다.(O)
2. 고객이 결제 한다.(O)
3. 예약이 신청 되면 예약 신청 내역이 호텔에 전달 된다.(O)
4. 호텔이 확인 하여 예약을 확정 한다.(O)
5. 고객이 예약 신청을 취소할 수 있다.(O)
6. 예약이 취소 되면 호텔 예약이 취소 된다.(O)
7. 고객이 예약 진행 상황을 중간 중간 조회 한다.(O)
8. 예약 상태가 바뀔 때 마다 카카오톡으로 알림을 보낸다.(O)
9. 고객이 예약 취소를 하면 예약 정보는 삭제되나, 객실 관리팀에서는 이력 관리를 위해 취소 장부를 별도 저장한다.(O)

### 비기능 요구사항을 커버하는지 검증
1. 트랜잭션 
   - 결제가 되지 않은 예약건은 아예 호텔 예약 신청이 되지 않아야 한다. `Sync 호출`(O)
   - Request-Response 방식 처리

2. 장애격리
   - 객실관리 기능이 수행 되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다.(O)
   - Eventual Consistency 방식으로 트랜잭션 처리(Pub/Sub)

## 헥사고날 아키텍처 다이어그램 도출
- 비지니스 로직은 내부에 순수한 형태로 구현
- 그 이외의 것을 어댑터 형식으로 설계 하여 해당 비지니스 로직이 어느 환경에서도 잘 도작하도록 설계

<img width="1029" alt="had_20210125" src="https://user-images.githubusercontent.com/58290368/105652095-d6e3fd00-5efb-11eb-9c65-1a2e2b26a7d1.png">

# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084 이다)

```
cd /Users/imdongbin/Documents/study/MSA/hotel/app
mvn spring-boot:run

cd /Users/imdongbin/Documents/study/MSA/hotel/hotel
mvn spring-boot:run 

cd /Users/imdongbin/Documents/study/MSA/hotel/pay
mvn spring-boot:run  

cd /Users/imdongbin/Documents/study/MSA/hotel/customer
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 예를 들어 price, payMethod 등)

```
package hotel;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String status;
    private Integer price;
    private String payMethod;

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
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public Integer getPrice() {
        return price;
    }

    public void setPrice(Integer price) {
        this.price = price;
    }
    public String getPayMethod() {
        return payMethod;
    }

    public void setPayMethod(String payMethod) {
        this.payMethod = payMethod;
    }

}
```

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package hotel;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long>{

}
```

- 적용 후 REST API 의 테스트
```
# app 서비스의 주문처리
http localhost:8081/orders hotelId=1001 roomType=standard

# hotel 서비스의 예약처리
http localhost:8082/reservations orderId=1

# 주문 상태 확인
http localhost:8081/orders/1

HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 03 Feb 2021 01:10:31 GMT
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
    "hotelId": "1001",
    "roomType": "standard",
    "status": null
}

```

## 폴리글랏 퍼시스턴스

이 항목은 현재 환경에서 가능한 항목 구현을 마무리 하고 추가적으로 고려 예정

## 폴리글랏 프로그래밍

이 항목은 현재 환경에서 가능한 항목 구현을 마무리 하고 추가적으로 고려 예정

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(app)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (app) PaymentService.java

package hotel.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="pay", url="http://localhost:8083")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

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

        hotel.external.Payment payment = new hotel.external.Payment();

        AppApplication.applicationContext.getBean(hotel.external.PaymentService.class)
            .pay(payment);

    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:
```
# 결제 (pay) 서비스를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders hotelId=1002 roomType=delux   

#Fail
HTTP/1.1 500 
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Wed, 03 Feb 2021 01:27:19 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error",
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction",
    "path": "/orders",
    "status": 500,
    "timestamp": "2021-02-03T01:27:19.505+0000"
}

http localhost:8081/orders hotelId=1003 roomType=suite   

#Fail
HTTP/1.1 500 
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Wed, 03 Feb 2021 01:28:00 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error",
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction",
    "path": "/orders",
    "status": 500,
    "timestamp": "2021-02-03T01:28:00.481+0000"
}

#결제서비스 재기동
cd /Users/imdongbin/Documents/study/MSA/hotel/pay
mvn spring-boot:run

#주문처리
http localhost:8081/orders hotelId=1002 roomType=delux   

#Success
HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Wed, 03 Feb 2021 01:29:55 GMT
Location: http://localhost:8081/orders/7
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/7"
        },
        "self": {
            "href": "http://localhost:8081/orders/7"
        }
    },
    "hotelId": "1002",
    "roomType": "delux",
    "status": null
}

http localhost:8081/orders hotelId=1003 roomType=suite

#Success
HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Wed, 03 Feb 2021 01:31:16 GMT
Location: http://localhost:8081/orders/9
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/9"
        },
        "self": {
            "href": "http://localhost:8081/orders/9"
        }
    },
    "hotelId": "1003",
    "roomType": "suite",
    "status": null
}
```

## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제가 이루어진 후에 호텔 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 호텔 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
#Payment.java

package hotel;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

...

@PostPersist
    public void onPostPersist(){
        PayApproved payApproved = new PayApproved();
        BeanUtils.copyProperties(this, payApproved);
        payApproved.publishAfterCommit();
    }
```

- 호텔 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 카톡/이메일 등으로 호텔은 노티를 받고, 예약 상황을 확인 하고, 최종 예약 상태를 UI에 입력할테니, 우선 예약정보를 DB에 받아놓은 후, 이후 처리는 해당 Aggregate 내에서 하면 되겠다.

```
# PolicyHandler.java

package hotel;

import hotel.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
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
    ReservationRepository reservationRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPayApproved_(@Payload PayApproved payApproved){

        if(payApproved.isMe()){
            System.out.println("##### listener  : " + payApproved.toJson());
            // 결제 승인 되었으니 호텔에 예약 확인 하라고 카톡 알림 처리 필요
            Reservation reservation = new Reservation();
            reservation.setOrderId(payApproved.getOrderId());
            reservation.setStatus("Confirming reservation");

            reservationRepository.save(reservation);
        }
    }

}
```

호텔 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 호텔 시스템이 유지보수로 인해 잠시 내려간 상태라도 예약 주문을 받는데 문제가 없어야 한다.

```
# 호텔 서비스 (hotel) 를 잠시 내려놓음 (ctrl+c)

# 주문처리
http localhost:8081/orders hotelId=3001 roomType=suite   #Success

# 결제처리
http localhost:8083/payments orderId=1 price=100000 payMethod=card   #Success

# 주문 상태 확인
http localhost:8081/orders/1     

# 주문상태 안바뀜 확인
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 03 Feb 2021 05:07:37 GMT
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
    "hotelId": "3001",
    "roomType": "suite",
    "status": null
}

# hotel 서비스 기동
cd /Users/imdongbin/Documents/study/MSA/hotel/hotel
mvn spring-boot:run

# 주문상태 확인
http localhost:8081/orders/1

# 주문 상태가 "Confirming reservation"으로 확인
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 03 Feb 2021 05:08:24 GMT
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
    "hotelId": "3001",
    "roomType": "suite",
    "status": "Confirming reservation"
}
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, pipeline build script 는 각 프로젝트 폴더 이하 cloudbuild.yml 에 포함되었다. (MSAEZ서 추출한 소스 코드 내 포함)

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# app 서비스, application.yml

feign:
  hystrix:
    enabled: true

# To set thread isolation to SEMAPHORE
#hystrix:
#  command:
#    default:
#      execution:
#        isolation:
#          strategy: SEMAPHORE

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) Payment.java (Entity)

    @PrePersist
    public void onPrePersist(){

        if("cancle".equals(payMethod)) {
            // 예시 푸드 딜리버리처럼 행위 필드를 하나 더 추가 하려다가 payMethod에 cancle 들어오면 취소 요청인 것으로 정의
            PayCanceled payCanceled = new PayCanceled();
            BeanUtils.copyProperties(this, payCanceled);
            payCanceled.publish();
        } else {
            PayApproved payApproved = new PayApproved();
            BeanUtils.copyProperties(this, payApproved);

            // 바로 이벤트를 보내버리면 주문정보가 커밋되기도 전에 예약 상태 변경 이벤트가 발송되어 주문테이블의 상태가 바뀌지 않을 수 있다.
            // TX 리스너는 커밋이 완료된 후에 이벤트를 발생하도록 만들어준다.
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                @Override
                public void beforeCommit(boolean readOnly) {
                    payApproved.publish();
                }
            });

            try { // 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"hotelId": "4001", "roomType": "standard"}'

defaulting to time-based testing: 60 seconds

{	"transactions":			        1054,
	"availability":			       80.34,
	"elapsed_time":			       59.74,
	"data_transferred":		        0.30,
	"response_time":		        5.47,
	"transaction_rate":		       17.64,
	"throughput":			        0.00,
	"concurrency":			       96.58,
	"successful_transactions":	        1054,
	"failed_transactions":		         258,
	"longest_transaction":		        8.41,
	"shortest_transaction":		        0.44
}

```
- 80.34% 성공, 19.66% 실패

### 오토스케일 아웃
로칼 도커+쿠버네틱스 환경 조성 후 작업 예정.

## 무정지 재배포
로칼 도커+쿠버네틱스 환경 조성 후 작업 예정.

# 신규 개발 조직의 추가
...


