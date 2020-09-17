# Intensive Lv2. 정재승

음식을 주문하고 요리하여 배달하는 현황을 확인 할 수 있는 CNA의 개발  
개인과제 추가 내용  
 : 주문 완료 시 포인트를 적립, 주문 취소 시 포인트를 취소

# Table of contents

- [Restaurant](# )
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출](#동기식-호출)
    - [비동기식 호출과 Saga Pattern](#비동기식-호출과-Saga-Pattern)
    - [Gateway](#Gateway)
    - [CQRS](#CQRS)
  - [운영](#운영)
    - [AWS를 활용한 코드 자동빌드 배포 환경구축](#AWS를-활용한-코드-자동빌드-배포-환경구축)
    - [서킷 브레이킹과 오토스케일](서킷-브레이킹과-오토스케일)
    - [무정지 재배포](#무정지-재배포)
    - [마이크로서비스 로깅 관리를 위한 PVC 설정](#마이크로서비스-로깅-관리를-위한-PVC-설정)
    - [SelfHealing](#SelfHealing)
  - [첨부](#첨부)

# 서비스 시나리오

음식을 주문하고, 포인트를 적립, 요리현황 및 배달현황을 조회한다.

## 기능적 요구사항

1. 고객이 주문을 하면 주문정보를 바탕으로 요리가 시작된다.
1. 주문이 완료되면 포인트 적립을 해준다.                                                 <--추가 부분
1. 요리가 완료되면 배달이 시작된다. 
1. 고객이 주문취소를 하게 되면 요리가 취소된다.
1. 주문이 취소되면 포인트 적립을 취소해준다.                                             <--추가 부분
1. 고객 주문에 재고가 없을 경우 주문이 취소된다. 
1. 고객은 Mypage를 통해, 주문과 요리, 배달, 포인트 적립의 전체 상황을 조회할수 있다.

## 비기능적 요구사항
1. 장애격리
    1. 주문시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 주문하도록 유도한다.
    1. 주문, 포인트, 요리, 배달, 마이페이지 시스템이 죽을 경우 재기동 될 수 있도록 한다.
1. 운영
    1. 마이크로서비스별로 로그를 한 곳에 모아 볼 수 있도록 시스템을 구성한다.
    1. 마이크로서비스의 개발 및 빌드 배포가 한번에 이루어질 수 있도록 시스템을 구성한다.
    1. 서비스라우팅을 통해 한개의 접속점으로 서비스를 이용할 수 있도록 한다.
    1. 주문 시스템이 과중되면 Replica를 추가로 띄울 수 있도록 한다.                       <--추가 부분
    1. 포인트 적립/취소 서비스가 과중되면 Replica를 추가로 띄울 수 있도록 한다.           <--추가 부분
    

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과 : http://www.msaez.io/#/storming/6FSFbn3irgYzkgbhPPVBMcEg2Un1/mine/4742e06e8ca55cd185a89fba17420008/-MHG2NGoPl6nOPcnXPQ-
![eventstorming](https://user-images.githubusercontent.com/68719144/93326489-d6539f80-f853-11ea-836e-a3eb99b91d2b.jpg)  

### 이벤트 도출
1. 주문됨
1. 주문취소됨
1. 요리재고체크됨
1. 요리완료
1. 배달
1. 포인트 적립됨               <--추가 부분
1. 포인트 적립 취소됨          <--추가 부분


### 어그리게잇으로 묶기

  * 고객의 주문(Order), 식당의 요리(Cook), 배달(Delivery), 포인트(Point) 은 그와 연결된 command와 event 들에 의하여 트랙잭션이 유지되어야 하는 단위로 묶어 줌.

### Policy 부착 

### Policy와 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res)

### 기능적 요구사항 검증
 * 고객이 메뉴를 주문한다.(ok)
 * 주문된 주문정보를 레스토랑으로 전달한다.(ok)
 * 주문정보를 바탕으로 요리가 시작된다.(ok)
 * 요리가 완료되면 배달이 시작된다.(ok)
 * 고객은 본인의 주문을 취소할 수 있다.(ok)
 * 주문이 취소되면 요리를 취소한다.(ok)
 * 주문이 취소되면, 요리취소 내용을 고객에게 전달한다.(ok)
 * 고객이 주문 시 재고량을 체크한다.(ok)
 * 재고가 없을 경우 주문이 취소된다.(ok)
 * 주문이 완료되면 포인트를 적립한다. (ok)                                  <--추가 부분
 * 포인트 적립이 되면, 완료내용을 고객에게 전달한다.                        <--추가 부분
 * 주문이 취쇠되면 포인트 적립을 취소한다. (ok)                             <--추가 부분
 * 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.(ok)  <--추가 부분

</br>
</br>



# 구현:

분석/설계 단계에서 도출된 아키텍처에 따라, 각 BC별로 마이크로서비스들을 스프링부트 + JAVA로 구현하였다. 각 마이크로서비스들은 Kafka와 RestApi로 연동되며 스프링부트의 내부 H2 DB를 사용한다.


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 포인트- Point 마이크로서비스).

```
import org.springframework.beans.BeanUtils;
import javax.persistence.*;

@Entity
@Table(name="Point_table")
public class Point {

    //수정
    private static int pointQty=10;

    private boolean pointFlowChk=true;
    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String status;
    private Long sendDate;
    private String pointKind;
    
    ....
}
```
- JPA를 활용한 Repository Pattern을 적용하여 이후 데이터소스 유형이 변경되어도 별도의 처리 없이 사용 가능한 Spring Data REST 의 RestRepository 를 적용하였다
```
import org.springframework.data.repository.PagingAndSortingRepository;
public interface PointRepository extends PagingAndSortingRepository<Order, Long>{

}
```
</br>

## 동기식 호출

분석단계에서의 조건 중 하나로 주문->취소 간의 호출은 트랜잭션으로 처리. 호출 프로토콜은 Rest Repository의 REST 서비스를 FeignClient 를 이용하여 호출.
- 포인트(point) 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
@FeignClient(name="point", url="${api.url.point}")
public interface PointService {

    @RequestMapping(method= RequestMethod.POST, path="/points")
    public void pointSend(@RequestBody Point point);

}
```


</br>

## 비동기식 호출과 Saga Pattern

주문 접수 및 포인트 적립, 포인트 취소는 비동기식으로 처리하며 시스템 상황에 따라 접수 및 취소가 블로킹 되지 않도록 처리 한다. 
포인트 적립이 완료되었을 경우 주문단계로 비동기식 포인트 적립 완료 발생(publish)
 
```

    @PostPersist
    public void onPostPersist(){
        System.out.println("@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 26 ");
        System.out.println(this.getStatus());
        if("Point : Point SENDED".equals(this.getStatus())){
            //ORDER -> Point SEND 경우
            PointSended pointSended = new PointSended();
            BeanUtils.copyProperties(this, pointSended);
            pointSended.publishAfterCommit();
        }
    }

    @PrePersist
    public void onPrePersist(){

        System.out.println(this.getStatus() + "++++++++++++++++++++++++++++++++");

        if("ORDER : point SEND".equals(this.getStatus())){
            this.setPointKind("100 point!!");
            this.setStatus("Point : Point SENDED");
            this.setSendDate(System.currentTimeMillis());
        }else {
            this.setStatus("Point : Point SEND CANCELLED");
            //ORDER -> point CANCEL 경우

            System.out.println("47 *****************************************");
            PointSendCancelled pointSendCancelled = new PointSendCancelled();
            BeanUtils.copyProperties(this, pointSendCancelled);
            pointSendCancelled.publishAfterCommit();
        }
    }
```

</br>

## Gateway
하나의 접점으로 서비스를 관리할 수 있는 Gateway를 통한 서비스라우팅을 적용 한다. Loadbalancer를 이용한 각 서비스의 접근을 확인 함.

```
# Gateway 설정(https://github.com/dew0327/final-cna-gateway/blob/master/target/classes/application.yml)
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/**
        - id: cook
          uri: http://cook:8080
          predicates:
            - Path=/cooks/**,/cancellations/**
        - id: delivery
          uri: http://delivery:8080
          predicates:
            - Path=/deliveries/**
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
        - id: point
          uri: http://point:8080
          predicates:
            - Path= /points/**
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
![gateway](https://user-images.githubusercontent.com/68719144/93342384-3ead7b80-f86a-11ea-8277-96d41180f75e.jpg)
![gateway-sample](https://user-images.githubusercontent.com/68719144/93342388-40773f00-f86a-11ea-9210-13e09eaadba6.jpg)


</br>

## CQRS
기존 코드에 영향도 없이 mypage 용 materialized view 구성한다. 고객은 주문 접수, 요리 상태, 포인트 적립현황, 배송현황 등을 한개의 페이지에서 확인 할 수 있게 됨.</br>
![mypage-sample](https://user-images.githubusercontent.com/68719144/93342738-a663c680-f86a-11ea-8a73-c5239d3cc303.jpg)

```
@StreamListener(KafkaProcessor.INPUT)
    public void whenPointSended_then_UPDATE_6(@Payload PointSended pointSended) {
      try {

            if (pointSended.isMe()) {
                // view 객체 조회
                Thread.sleep(1000);
                List<Mypage> mypageList = mypageRepository.findByOrderId(pointSended.getOrderId());
                System.out.println("################" + pointSended.getId());
                System.out.println("################" + pointSended.getStatus());
                System.out.println("################" + pointSended.getPointKind());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setPointId(pointSended.getId());
                    mypage.setPointStatus(pointSended.getStatus());
                    mypage.setPointSendDate(pointSended.getSendDate());
                    mypage.setPointKind(pointSended.getPointKind());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPointSendCancelled_then_UPDATE_7(@Payload PointSendCancelled pointSendCancelled) {
        try {
            if (pointSendCancelled.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByOrderId(pointSendCancelled.getOrderId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setPointId(pointSendCancelled.getId());
                    mypage.setPointStatus(pointSendCancelled.getStatus());
                    mypage.setPointSendDate(pointSendCancelled.getSendDate());
                    mypage.setPointKind(pointSendCancelled.getPointKind());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }

```


</br>
</br>


# 운영

## AWS를 활용한 코드 자동빌드 배포 환경구축

  * AWS codebuild를 설정하여 github이 업데이트 되면 자동으로 빌드 및 배포 작업이 이루어짐.
  * Github에 Codebuild를 위한 yml 파일을 업로드하고, codebuild와 연동 함
  * 각 마이크로서비스의 build 스펙
  ```
    https://github.com/dew0327/final-js-order/buildspec.yml
    https://github.com/dew0327/final-js-cook//buildspec.yml
    https://github.com/dew0327/final-js-delivery/buildspec.yml
    https://github.com/dew0327/final-js-gateway/buildspec.yml
    https://github.com/dew0327/final-js-mypage/buildspec.yml
    https://github.com/dew0327/final-js-point/buildspec.yml
  ```
  
</br>

## 서킷 브레이킹과 오토스케일

* 서킷 브레이킹 :
주문이 과도하여 포인트 적립에 부하가 발생할 경우 CB 를 통하여 장애격리. 500 에러가 1번 발생하면 1분간 CB 처리하여 30% 접속 차단
```
# AWS codebuild에 설정(https://github.com/dew0327/final-js-point/buildspec.yml)
 http:
   http1MaxPendingRequests: 1   # 연결을 기다리는 request 수를 1개로 제한 (Default 
   maxRequestsPerConnection: 1  # keep alive 기능 disable
 outlierDetection:
  consecutiveErrors: 1          # 5xx 에러가 1번 발생하면
  interval: 1s                  # 1초마다 스캔 하여
  baseEjectionTime: 1m         # 1분 동안 circuit breaking 처리   
  maxEjectionPercent: 30       # 30% 로 차단
```

* 오토스케일(HPA) :
CPU사용률 50% 초과 시 replica를 3개까지 확장해준다. 
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: skcchpa-point
  namespace: teamc
  spec:
    scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: $_PROJECT_NAME                # point (주문) 서비스 HPA 설정
    minReplicas: 1                      # 최소 1개
    maxReplicas: 3                      # 최대 3개
    targetCPUUtilizationPercentage: 50  # cpu사용율 50프로 초과 시 
```    
* 부하테스트(Siege)를 활용한 부하 적용 후 서킷브레이킹 / 오토스케일 내역을 확인한다.
![HPA, Circuit Breaker  SEIGE_STATUS](https://user-images.githubusercontent.com/68719144/93344893-2c810c80-f86d-11ea-9c42-9672aebe8eb2.jpg)
![HPA  TO-BE POD STATUS](https://user-images.githubusercontent.com/68719144/93342977-ea56cb80-f86a-11ea-8434-718734741fcc.jpg)

</br>

## 무정지 재배포

* 무정지 배포를 위해 ECR 이미지를 업데이트 하고 이미지 체인지를 시도 함. Github에 소스가 업데이트 되면 자동으로 AWS CodeBuild에서 컴파일 하여 이미지를 ECR에 올리고 EKS에 반영.
  이후 아래 옵션에 따라 무정지 배포 적용 된다.
  

```
# AWS codebuild에 설정(https://github.com/dew0327/final-js-point/buildspec.yml)
  spec:
    replicas: 2
    minReadySeconds: 10   # 최소 대기 시간 10초
    strategy:
      type: RollingUpdate
      rollingUpdate:
      maxSurge: 1         # 1개씩 업데이트 진행
      maxUnavailable: 0   # 업데이트 프로세스 중에 사용할 수 없는 최대 파드의 수

```

- 새버전으로의 배포 시작(V1로 배포)
kubectl set image deployment/point -n teamc point=271153858532.dkr.ecr.ap-northeast-1.amazonaws.com/admin15-point:v1

- siege를 이용한 부하 적용. Readiness Probe 설정을 통한 ZeroDownTime 구현
![zerotime deploy - seige](https://user-images.githubusercontent.com/68719144/93343081-09555d80-f86b-11ea-9b62-08d5f83365a2.jpg)


- Readiness Probe 설정을 통한 ZeroDownTime 설정.
```
  readinessProbe:
    tcpSocket:
      port: 8080
      initialDelaySeconds: 15      # 서비스 어플 기동 후 15초 뒤 시작
      periodSeconds: 20            # 20초 주기로 readinessProbe 실행 
```

- 새버전 배포 확인(V1 적용)
![zerotime deploy - describe](https://user-images.githubusercontent.com/68719144/93343091-0c504e00-f86b-11ea-8829-9b934b65b79f.jpg)





</br>

## 마이크로서비스 로깅 관리를 위한 PVC 설정
AWS의 EFS에 파일시스템을 생성(admin15-efs (fs-96929df7))하고 서브넷과 클러스터(admin15-cluster)를 연결하고 PVC를 설정해준다. 각 마이크로 서비스의 로그파일이 EFS에 정상적으로 생성되고 기록됨을 확인 함.
```
#AWS의 각 codebuild에 설정(https://github.com/dew0327/final-js-order/buildspec.yml)
volumeMounts:  
- mountPath: "/mnt/aws"    # POINT서비스 로그파일 생성 경로 fs-27564906
  name: volume                 
volumes:                                # 로그 파일 생성을 위한 EFS, PVC 설정 정보  
- name: volume
  persistentVolumeClaim:
  claimName: aws-efs  
```
![logs](https://user-images.githubusercontent.com/68719144/93344365-90ef9c00-f86c-11ea-9f01-1d7fbe1c0184.jpg)


</br>

## SelfHealing
운영 안정성의 확보를 위해 마이크로서비스가 아웃된 뒤에 다시 프로세스가 올라오는 환경을 구축한다. 프로세스가 죽었을 때 다시 기동됨을 확인함.
```
#AWS의 각 codebuild에 설정(https://github.com/dew0327/final-js-cook/buildspec.yml)
                    livenessProbe:
                      exec:
                        command:
                        - cat
                        - /mnt/aws/logs/cook-application.log
                      initialDelaySeconds: 20
                      periodSeconds: 3
```
기동 로그 파일 삭제   
![liveness-log-delete](https://user-images.githubusercontent.com/68719144/93350668-d9f71e80-f873-11ea-8497-6cdf167d85d3.jpg)  


프로세스 재기동 확인  
![liveness-restart](https://user-images.githubusercontent.com/68719144/93350643-d4013d80-f873-11ea-8420-1c1e22872b61.jpg)

</br>
</br>



# 첨부
팀프로젝트 구성을 위해 사용한 계정 정보 및 클러스터 명, Github 주소 등의 내용 공유 
* AWS 계정 명 : admin15
```
Region : ap-northeast1
EFS : admin15-efs (fs-27564906)
EKS : admin15-cluster
ECR : admin15-order / admin15-delivery / admin15-cook / admin15-mypage / admin15-gateway / admin15-point
Codebuild : admin15-order / admin15-delivery / admin15-cook / admin15-mypage / admin15-gateway / admin15-point
```
* Github :</br>
```
https://github.com/dew0327/final-js-gateway
https://github.com/dew0327/final-js-order
https://github.com/dew0327/final-js-delivery
https://github.com/dew0327/final-js-cook
https://github.com/dew0327/final-js-mypage
https://github.com/dew0327/final-js-point
```
