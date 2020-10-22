# DT_6th_team5_carShare (자동차 공유 서비스)

5팀 자동차 공유 서비스 CNA개발 실습을 위한 프로젝트

# 구현 Repository
 1. 접수관리 : https://github.com/YoungDukGe1000Won/Skccuser29carShareOrder.git
 1. 결제관리 : https://github.com/YoungDukGe1000Won/Skccuser29carSharePayment.git
 1. 배송관리 : https://github.com/YoungDukGe1000Won/Skccuser29carShareDelivery.git
 1. 고객페이지 : https://github.com/YoungDukGe1000Won/Skccuser29carShareStatusview.git
 1. 게이트웨이 : https://github.com/YoungDukGe1000Won/Skccuser29carShareGateway.git
 1. [개인] 쿠폰관리 : https://github.com/YoungDukGe1000Won/Skccuser29carShareCoupon.git

# Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [시나리오 테스트결과](#시나리오-테스트결과)
- [분석/설계](#분석설계)
- [구현](#구현)
  - [DDD 의 적용](#ddd-의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [서킷 브레이킹 / 장애격리](#서킷-브레이킹-/-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

## 기능적 요구사항
1. 고객이 공유차를 선택하여 렌탈한다.
1. 고객이 결제하여 접수한다.
1. 업체가 공유차를 고객위치로 가져다놓는다.
1. 고객이 렌탈을 취소할 수 있다.
1. 렌탈이 취소되면 배송이 취소된다.
1. 고객이 자신의 렌탈 정보를 조회한다.
1. [개인] 배송이 완료된 고객에 대해 쿠폰을 발행한다. 
1. [개인] 배송이 취소된 고객에 대해 쿠폰을 환수한다.

## 비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 접수가 성립되지 않아야 한다(Sync 호출)
    1. [개인] 쿠폰환수가 되지 않은 주문건은 배송취소가 되지 않아야 한다(Sync 호출) 
1. 장애격리
    1. 배송관리 기능이 수행되지 않더라도 접수는 정상적으로 처리 가능하다(Async(event-driven), Eventual Consistency)
    1. 접수시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다(Circuit breaker, fallback)
    1. [개인] 쿠폰관리 기능이 수행되지 않더라도 배송은 정상적으로 처리 가능하다(Async(event-driven), Eventual Consistency)
    1. [개인] 쿠폰관리 시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 쿠폰을 발행하도록 유도한다(Circuit breaker, fallback) 
1. 성능
    1. 고객이 본인의 렌탈 상태 및 이력을 접수시스템에서 확인할 수 있어야 한다(CQRS)
    1. [개인] 고객이 본인의 쿠폰 발행 상태를 접수 시스템에서 확인할 수 있어야한다(CQRS)


# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
![79684144-2a893200-826a-11ea-9a01-79927d3a0107](https://user-images.githubusercontent.com/42608068/96371393-84789f00-119c-11eb-80d9-ffbcab38ff84.png)


## [개인] TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/42608068/96682472-d34c5180-13b3-11eb-932a-d29f7614cc96.png)


## [개인] 이벤트스토밍
* MSAEz 로 모델링한 이벤트스토밍 결과:  
![image](https://user-images.githubusercontent.com/42608068/96685988-dac22980-13b8-11eb-99dd-a18bba72bd3e.png)

### 이벤트 도출 
![제목 없음1](https://user-images.githubusercontent.com/42608068/96541160-60bb7300-12da-11eb-8eda-4beb637fa24f.png)

### 부적격 이벤트 탈락
![제목 없음2](https://user-images.githubusercontent.com/42608068/96541195-6fa22580-12da-11eb-94c0-9efb0947e5aa.png)

### 액터, 커맨드 부착하여 읽기 좋게
![제목 없음3](https://user-images.githubusercontent.com/42608068/96541203-77fa6080-12da-11eb-8a8a-50a018a72961.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/42608068/96597010-4c05cc00-1328-11eb-8372-5241800cf7fe.png)

### 바운디드 컨텍스트로 묶기
![제목 없음5](https://user-images.githubusercontent.com/42608068/96541235-919ba800-12da-11eb-8c49-84655f2ca88e.png)

```
# 도메인 서열
        - Core Domain:  접수 및 배송 관리 : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는  1주일 1회 미만
        - Supporting Domain:   접수 상태 페이지 : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 80% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   결제 관리 : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)
```

### 폴리시 부착 
![제목 없음6](https://user-images.githubusercontent.com/42608068/96541251-99f3e300-12da-11eb-99f9-8a9027a7b855.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![제목 없음7](https://user-images.githubusercontent.com/42608068/96541279-a11af100-12da-11eb-9d0d-3cf209f7216b.png)

### [개인] 완성된 모형
![image](https://user-images.githubusercontent.com/42608068/96680713-004b3500-13b1-11eb-813e-5b6e332108e4.png)

### 완성본에 대한 기능적 요구사항을 커버하는지 검증
![제목 없음12](https://user-images.githubusercontent.com/42608068/96543922-72077e00-12e0-11eb-91bf-ae6aaf5e8fbb.png)
    
    - 고객이 공유차를 선택하여 렌탈한다 (ok)
    - 고객이 결제하여 접수한다 (ok)
    - 업체가 공유차를 고객위치로 가져다놓는다 (ok)
    - [개인] 배송이 완료된 고객에 대해 쿠폰을 발행한다 (ok)

![제목 없음13](https://user-images.githubusercontent.com/42608068/96543936-79c72280-12e0-11eb-98a2-0c67478f6926.png)

    - 고객이 주문을 취소할 수 있다 (ok)
    - 렌탈이 취소되면 배송이 취소된다 (ok)
    - [개인] 배송이 취소된 고객에 대해 쿠폰을 회수한다 (ok)

![제목 없음14](https://user-images.githubusercontent.com/42608068/96543997-9cf1d200-12e0-11eb-9a71-9aa743f7de44.png)
   
    - 고객이 자신의 렌탈 정보를 조회한다 (ok)
    - [개인] 고객이 자신의 쿠폰 정보를 조회한다 (ok)
    
### 완성본에 대한 비기능적 요구사항을 커버하는지 검증
![제목없음22](https://user-images.githubusercontent.com/42608068/96582783-c2013780-1316-11eb-8bfc-dba64c7af837.png)

    1. 트랜잭션
    - 고객의 주문에 따라 결제가 진행된다(결제가 정상적으로 완료되지 않으면 주문이 되지 않는다) > Sync
    - 고객의 결제 완료에 따라 배송이 진행된다 > Async
    - [개인] 배송취소에 따라 쿠폰 회수가 진행된다(쿠폰이 정상적으로 회수되지 않으면 배송이 취소 되지 않는다) > Sync
    - [개인] 배송완료에 따라 쿠폰 발행이 진행된다 > Async
    2. 장애격리
    - 배송 서비스에 장애가 발생하더라도 주문 및 결제는 정상적으로 처리 가능하다 > Async(event driven)
    - [개인] 쿠폰 서비스에 장애가 발생하더라도 배송은 정상적으로 처리 가능하다 > Async(event driven)
    - 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
    3. 성능
    - 고객은 본인의 상태 정보를 확인할 수 있다 > CQRS
    - [개인] 고객은 본인의 쿠폰 발행 상태 정보를 확인할 수 있다 > CQRS

## [개인] 헥사고날 아키텍처 다이어그램 도출
![image](https://user-images.githubusercontent.com/42608068/96682973-84eb8280-13b4-11eb-9251-1d0792f08603.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

   
# 구현

## 시나리오 테스트결과
| 기능 | 이벤트 Payload |
|---|:---:|
| 1.고객이 공유차 렌탈을 접수한다.</br>2. 결제가 정상적으로 완료되면 접수가 진행된다. (Sync)</br>3. 접수가 완료되면 배송이 시작된다. (Async)</br>4. 배송이 시작되면 접수정보의 상태를 변경한다. (Async)</br>[개인]5. 배송이 완료되면 쿠폰 발행이 시작된다. (Async)</br>6. 쿠폰이 발행되면 접수정보의 상태를 변경한다. (Async)|![image](https://user-images.githubusercontent.com/42608068/96695286-71481800-13c4-11eb-9f72-6db8cf6c9645.png)|
| 7.고객이 공유차 렌탈을 취소한다.</br>[개인]8. 쿠폰 회수가 정상적으로 완료되면 배송 취소가 진행된다. (Sync)</br>9. 배송 취소가 정상적으로 완료되면 결제 취소가 진행된다. (Sync)</br>10.결제 취소도 정상적으로 이어지면 접수가 최종적으로 취소된다. (Async)|![image](https://user-images.githubusercontent.com/42608068/96697397-d7359f00-13c6-11eb-806a-95bc5584a03b.png)|
| 11.고객이 접수 상태를 조회한다.|![image](https://user-images.githubusercontent.com/42608068/96839432-27266b80-1484-11eb-9c3b-8f934bebb380.png)|

## DDD 의 적용
분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* customerpage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| order | 접수 관리 | 8081 | http://localhost:8081/orders |http://Skccuser29carshareorder:8080/orders |
| delivery | 배송 관리 | 8082 | http://localhost:8082/deliveries | http://Skccuser29carsharedelivery:8080/deliveries |
| customerpage | 상태 조회 | 8083 | http://localhost:8083/customerpages | http://Skccuser29carsharestatusview:8080/customerpages |
| payment | 결제 관리 | 8084 | http://localhost:8084/payments | http://Skccuser29carsharepayment:8080/payments |
| coupon | 쿠폰 관리 | 8085 | http://localhost:8084/coupons | http://Skccuser29carsharepayment:8080/coupons |

## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://user29carshareorder:8080
          predicates:
            - Path=/orders/** 
        - id: delivery
          uri: http://user29carsharedelivery:8080
          predicates:
            - Path=/deliveries/**,/cancellations/**
        - id: statusview
          uri: http://user29carsharestatusview:8080
          predicates:
            - Path= /customerpages/**
        - id: payment
          uri: http://user29carsharepayment:8080
          predicates:
            - Path=/payments/**,/paymentCancellations/**
        - id: coupon
          uri: http://user29carsharecoupon:8080
          predicates:
            - Path=/coupons/**,/couponCancellations/**
```


## 폴리글랏 퍼시스턴스

CQRS 를 위한 customerpage 서비스 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.
[개인] CQRS 를 위한 쿠폰관리 서비스 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

```
pom.xml 에 적용
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 접수(order)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
[개인] 분석단계에서의 조건 중 하나로 배송취소(deliveryCancel)->쿠폰회수(couponCancel) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
- FeignClient 서비스 구현

```
# PaymentService.java

@FeignClient(name="payment", contextId ="payment", url="${api.payment.url}", fallback = PaymentServiceFallback.class)
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
```
[개인]
# CouponCancellationService.java

@FeignClient(name="coupon", url="${api.coupon.url}", fallback = CouponCancellationServiceFallback.class)
public interface CouponCancellationService {

    @RequestMapping(method= RequestMethod.POST, path="/couponCancellations")
    public void couponOffer(@RequestBody CouponCancellation couponCancellation);

}
```
- 접수요청을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        carshare.external.Payment payment = new carshare.external.Payment();
        payment.setOrderId(this.getId());
        payment.setProductId(this.getProductId());
        payment.setQty(this.getQty());
        payment.setStatus("OrderApproved");
        OrderApplication.applicationContext.getBean(carshare.external.PaymentService.class)
            .pay(payment);
    }
```
- 배송취소요청을 받은 직후(@PostPersist) 쿠폰회수를 요청하도록 처리
```
[개인]
#Cancellation.java (Entity)
 @PostPersist
    public void onPostPersist(){
        DeliveryCanceled deliveryCanceled = new DeliveryCanceled();
        BeanUtils.copyProperties(this, deliveryCanceled);
        deliveryCanceled.publishAfterCommit();

        skccuser.external.CouponCancellation couponCancellation = new skccuser.external.CouponCancellation();
        couponCancellation.setOrderId(this.getOrderId());
        couponCancellation.setPaymentId(this.getPaymentId());
        couponCancellation.setDeliveryId(this.getId());
        couponCancellation.setStatus("deliveryCancel");
        DeliveryApplication.applicationContext.getBean(skccuser.external.CouponCancellationService.class)
            .couponOffer(couponCancellation);
    }
```
- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 서비스가 장애가 나면 접수요청 못받는다는 것을 확인


```
#결제(payment) 서비스를 잠시 내려놓음 (ctrl+c)

#접수요청 처리
http localhost:8081/orders productId=1001 qty=1 status="order"   #Fail
http localhost:8081/orders productId=1002 qty=3 status="order"   #Fail

#결제 서비스 재기동
cd carsharepayment
mvn spring-boot:run

#접수요청 처리 성공
http localhost:8081/orders productId=1001 qty=1 status="order"   #Success
http localhost:8081/orders productId=1002 qty=3 status="order"   #Success
```
```
[개인]
#쿠폰(coupon) 서비스를 잠시 내려놓음 (ctrl+c)

#배송취소요청 처리
http localhost:8082/cancellations orderId=1 paymentId=1 status="deliveryCancel"   #Fail
http localhost:8082/cancellations orderId=2 paymentId=2 status="deliveryCancel"   #Fail

#쿠폰 서비스 재기동
cd user29carsharecoupon
mvn spring-boot:run

#배송취소요청 처리 성공
http localhost:8082/cancellations orderId=1 paymentId=1 status="deliveryCancel"   #Success
http localhost:8082/cancellations orderId=2 paymentId=2 status="deliveryCancel"   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, Fallback 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제가 이루어진 후에 배송 서비스로 이를 알려주는 행위는 동기식이 아니라 비동기식으로 처리하여 배송 서비스의 처리를 위해 결제가 블로킹되지 않도록 처리한다.
[개인] 배송이 이루어진 후에 쿠폰 서비스로 이를 알려주는 행위는 동기식이 아니라 비동기식으로 처리하여 쿠폰 서비스의 처리를 위해 결제가 블로킹되지 않도록 처리한다.
 
- 이를 위하여 결제이력 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package carshare;

@Entity
@Table(name="Payment_table")
public class Payment {

 ...
    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();    
    }
}
```
- [개인] 이를 위하여 발송이력 기록을 남긴 후에 곧바로 발송승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)

```
package skccuser;

@Entity
@Table(name="Delivery_table")
public class Delivery {

...
    @PostPersist
    public void onPostPersist(){
        Shipped shipped = new Shipped();
        BeanUtils.copyProperties(this, shipped);
        shipped.publishAfterCommit();
    }
}
```

- 배송 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다
```
package carshare;

...

@Service
public class PolicyHandler{

   @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Ship(@Payload Paid paid){

        if(paid.isMe()){
            Delivery delivery = new Delivery();
            delivery.setOrderId(paid.getOrderId());
            delivery.setPaymentId(paid.getId());
            delivery.setStatus("Shipped");

            deliveryRepository.save(delivery) ;
        }
    }

}

```

- 쿠폰 서비스에서는 배송완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다
```
package skccuser;

...

@Service
public class PolicyHandler{

   @StreamListener(KafkaProcessor.INPUT)
    public void wheneverShipped_CouponCancel(@Payload Shipped shipped){

        if(shipped.isMe()){
            Coupon coupon = new Coupon();
            coupon.setOrderId(shipped.getOrderId());
            coupon.setPaymentId(shipped.getPaymentId());
            coupon.setDeliveryId(shipped.getId());
            coupon.setStatus("offered");

            couponRepository.save(coupon) ;
        }
    }

}

```


배송 서비스는 접수/결제 서비스와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 배송 서비스가 유지보수로 인해 잠시 내려간 상태라도 접수신청을 받는데 문제가 없다.

```
#배송(delivery) 서비스를 잠시 내려놓음 (ctrl+c)

#접수요청 처리
http localhost:8081/orders productId=1003 qty=2 status="order"   #Success
http localhost:8081/orders productId=1004 qty=4 status="order"   #Success

#접수상태 확인
http localhost:8081/orders     # 주문상태 안바뀜 확인

#배송 서비스 기동
cd carsharedelivery
mvn spring-boot:run

#접수상태 확인
http localhost:8081/orders     # 접수상태가 "shipped(배송됨)"으로 확인
```

[개인] 쿠폰 서비스는 배송 서비스와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 쿠폰 서비스가 유지보수로 인해 잠시 내려간 상태라도 배송신청을 받는데 문제가 없다.

```
#쿠폰(coupon) 서비스를 잠시 내려놓음 (ctrl+c)

#배송요청 처리
http localhost:8082/deliveries orderId=1 productId=1 status="paid"   #Success
http localhost:8082/deliveries orderId=2 productId=4 status="paid"   #Success

#접수상태 확인
http localhost:8081/orders     # 주문상태 안바뀜 확인

#쿠폰 서비스 기동
cd user29carsharecoupon
mvn spring-boot:run

#접수상태 확인
http localhost:8081/orders     # 접수상태가 "offered(쿠폰발행됨)"으로 확인
```


# 운영

## CI/CD 설정

order에 대해 repository를 구성하였고, CI/CD플랫폼은 AWS의 CodeBuild를 사용했다.
![image](https://user-images.githubusercontent.com/70302900/96588525-b87bcd80-131e-11eb-90c8-8c4d1c4c1078.png)

Git Hook 설정으로 연결된 GitHub의 소스 변경 발생 시 자동 배포된다.
![image](https://user-images.githubusercontent.com/70302900/96588864-19a3a100-131f-11eb-8b72-846538a6ae42.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

### 서킷 브레이킹

* 서킷 브레이커 pending time 설정 변경
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: coupon
  namespace: skccuser29carshare
spec:
  host: user29carsharecoupon
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      interval: 1s
      consecutiveErrors: 2
      baseEjectionTime: 10s
      maxEjectionPercent: 100
EOF
```

* 부하테스트 툴(Siege) 설치 및 Order 서비스 Load Testing 
  - 동시 사용자 5명
  - 2초 실행 
```
root@siege-5c7c46b788-z47lb:/# siege -c5 -t2S -v http://a57016fa5c6b54052ae0192f630c1851-418262153.ap-northeast-1.elb.amazonaws.com:8080/coupons
** SIEGE 4.0.4
** Preparing 5 concurrent users for battle.
The server is now under siege...
HTTP/1.1 503     0.06 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.06 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.07 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 200     0.08 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.09 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.05 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.06 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.04 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.08 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.05 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.05 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.05 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 503     0.04 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 200     0.05 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.07 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
                          .
			  .
			  .
Lifting the server siege...
Transactions:                    121 hits
Availability:                  70.76 %
Elapsed time:                   1.56 secs
Data transferred:               0.05 MB
Response time:                  0.06 secs
Transaction rate:              77.56 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                    4.94
Successful transactions:         121
Failed transactions:              50
Longest transaction:            0.11
Shortest transaction:           0.01
```


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- Deployment 배포시 resource 설정 적용
```
    spec:
      containers:
          ...
          resources:
            limits:
              cpu: 500m 
            requests:
              cpu: 200m 
```

- replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 5프로를 넘어서면 replica 를 10개까지 늘려준다
```
kubectl autoscale deploy user29carsharecoupon -n skccuser29carshare --min=1 --max=10 --cpu-percent=5

NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/user29carsharecoupon   Deployment/user29carsharecoupon   1%/5%           1         10        0          101m
```

- 오토스케일이 어떻게 되고 있는지 HPA 모니터링을 걸어둔다, 어느정도 시간이 흐른 후, 스케일 아웃이 벌어지는 것을 확인할 수 있다
```
kubectl get deploy user29carsharecoupon -n skccuser29carshare -w 

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
user29carsharecoupon   1/1     1            1           20m
user29carsharecoupon   1/4     1            1           20m
user29carsharecoupon   1/4     1            1           20m
user29carsharecoupon   1/4     1            1           20m
user29carsharecoupon   1/4     4            1           20m
user29carsharecoupon   1/5     4            1           20m
user29carsharecoupon   1/5     4            1           20m
user29carsharecoupon   1/5     4            1           20m
user29carsharecoupon   1/10    8            1           20m
```
- kubectl get으로 HPA을 확인하면 CPU 사용률이 121%로 증가됐다.
```
NAME                                                       REFERENCE                         TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/user29carsharecoupon   Deployment/user29carsharecoupon   121%/5%         1         10        0          101m
```

## 무정지 재배포
먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler, 서킷 브레이킹 설정을 제거함 Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함 Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포되며, siege 모니터링 툴로 재배포 작업 중 서비스 중단됨을 확인(503 에러 발생)
배포기간중 Availability 가 평소 100%에서 80% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문으로 판단됨.
```
HTTP/1.1 200     0.04 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 200     0.06 secs:     377 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.05 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.05 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.07 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
HTTP/1.1 503     0.03 secs:      81 bytes ==> GET  /coupons
                          .
			  .
			  .
Lifting the server siege...
Transactions:                    218 hits
Availability:                  80.76 %
Elapsed time:                  10.22 secs
Data transferred:               0.05 MB
Response time:                  0.06 secs
Transaction rate:              71.26 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    1.01
Successful transactions:         175
Failed transactions:              43
Longest transaction:            1.20
Shortest transaction:           0.01

```

- 이를 막기위해 Readiness Probe 설정함(buildspec.yml 설정)
```
		  readinessProbe:
                    httpGet:
                      path: /
                      port: 8080
                    initialDelaySeconds: 30
                    timeoutSeconds: 2
                    periodSeconds: 5
                    failureThreshold: 10
```

- 동일한 시나리오로 재배포 한 후 Availability 확인: 배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.
```
Lifting the server siege...
Transactions:                     74 hits
Availability:                 100.00 %
Elapsed time:                   1.93 secs
Data transferred:               0.03 MB
Response time:                  0.13 secs
Transaction rate:              38.34 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                    4.88
Successful transactions:          74
Failed transactions:               0
Longest transaction:            1.10
Shortest transaction:           0.03
```

## Liveness Probe
- pod 삭제

![image](https://user-images.githubusercontent.com/16017769/96661174-6d95a080-1386-11eb-9f76-ab9a995c6286.png)

- 자동 생성된 pod 확인

![image](https://user-images.githubusercontent.com/16017769/96661206-81d99d80-1386-11eb-8b9d-539e36ef02e8.png)


## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.
Application에서 특정 도메일 URL을 ConfigMap 으로 설정하여 운영/개발등 목적에 맞게 변경가능합니다.  

* my-config.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: skccuser29carshare
data:
  api.coupon.url: http://user29carsharecoupon:8080
```
my-config라는 ConfigMap을 생성하고 key값에 도메인 url을 등록한다. 

* carshareorder/buildsepc.yaml (configmap 사용)
```
 cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          namespace: $_NAMESPACE
          labels:
            app: $_PROJECT_NAME
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: $_PROJECT_NAME
          template:
            metadata:
              labels:
                app: $_PROJECT_NAME
            spec:
              containers:
                - name: $_PROJECT_NAME
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$_AWS_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  ports:
                    - containerPort: 8080
                  env:
                    - name: api.payment.url
                      valueFrom:
                        configMapKeyRef:
                          name: my-config
                          key: api.coupon.url
                  imagePullPolicy: Always
                
        EOF
```
Deployment yaml에 해단 configMap 적용

* CouponCancellationService.java
```
@FeignClient(name="coupon", url="${api.coupon.url}")
public interface CouponCancellationService {

    @RequestMapping(method= RequestMethod.POST, path="/couponCancellations")
    public void couponOffer(@RequestBody CouponCancellation couponCancellation);

}
```
url에 configMap 적용

* kubectl describe pod user29carsharecoupon-bdd8c8c4c-l52h6  -n skccuser29carshare
```
Containers:
  carshareorder:
    Container ID:   docker://f3c983b12a4478f3b4a7ee5d7fea308638903eb62e0941edd33a3bce5f5f6513
    Image:          496278789073.dkr.ecr.ap-southeast-1.amazonaws.com/carshareorder:9289bba10d5b0758ae9f6279d56ff77b818b8b63
    Image ID:       docker-pullable://496278789073.dkr.ecr.ap-southeast-1.amazonaws.com/carshareorder@sha256:95395c95d1bc19ceae8eb5cc0b288b38dc439359a084610f328407dacd694a81
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 21 Oct 2020 02:13:01 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:  500m
    Requests:
      cpu:        200m
    Liveness:     http-get http://:8080/ delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:    http-get http://:8080/ delay=30s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.coupon.url:  <set to the key 'api.coupon.url' of config map 'my-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-5gx6w (ro)

```
kubectl describe 명령으로 컨테이너에 configMap 적용여부를 알 수 있다. 



