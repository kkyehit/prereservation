# 스마트폰 사전예약
# 개요
- 스마트폰을 사전 예약할 수 있는 시스템을 개발한다.

## 기능적 요구사항
1. 사용자는 스마트폰을 사전 예약할 수 있다.
2. 사용자가 사전 예약를 하면서 결재를 한다.
3. 사용자가 결재를 하면 영수증에서 예상 배송 날짜를 확인할 수 있다. ( 3주 뒤로 가정 )
4. 사용자는 사전예약을 취소할 수 있다.
5. 사용자가 사전예약을 취소하면 결제 및 영수증이 취소된다.
6. 사용자는 취소된 주문을 포함하여 주문 상세 정보를 확인 할 수 있다.

## 비기능적 요구사항
1. 트랜젝션 처리
- 사전 예약시 결제정보가 반드시 등록되어야 한다. 
    - REQ/RES
2. 장애격리
- 영수증 처리 서버에서 잠시 장애가 발생해도 사전예약 및 결제는 가능해야 한다
    - event-driven, 
- 결제 시스템에 요청이 너무 과중되지 않도록 해야 한다.
    - Circuit breaker, fallback
3. 성능
- 사용자가 사전예약된 정보를 요청할 때 각 서버에 요청하는 횟수를 최소화해야 한다.
    - CQRS
     
## Event Storming
![image](https://user-images.githubusercontent.com/53825723/132273067-e9c7c710-6399-4691-ae0a-1ebc86f0aab2.png)
- prereservation : 사용자의 사전 예약을 처리한다.
- payment : 사용자의 결제를 처리한다.
- receipt : 영수증을 발행한다.
- myreservation : 사용자의 예약 정보를 저장한다.

## Hexagonal Architecture
![image](https://user-images.githubusercontent.com/53825723/132273298-c2730168-9476-4b7b-8122-a3b4a72e98b1.png)
- 내부 로직 변화 없이 Database 변경이 가능하다.
    - h2 사용시 pom.xml
        ```xml
        <dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
        ```
    - hsqldb 사용시 pom.xml
        ```xml
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.4.1</version>
			<scope>runtime</scope>
		</dependency>
        ```
- 다른 서비스와 상관없이 다른 Database를 사용할 수 있다. (**Polyglot**)

# 개발환경 구성
- Container Registry : Azure Container Registry
- Deploy : Azure Kubernetes Service

- local 환경 셋팅
    ```bash
    # zookeeper 실행
    $kafka_home/bin/zookeeper-server-start.sh -daemon $kafka_home/config/zookeeper.properties &
    # kafka 실행
    $kafka_home/bin/kafka-server-start.sh -daemon $kafka_home/config/server.properties & 
    ```

- az 관련 셋팅
    ``` bash
    # az 계정에 로그인
    az login

    # acr에 로그인
    az acr login -n $ACR_NAME

    # aks 접속을 위한 정보 저장
    az aks get-credentials -n $AKS_NAME -g $RG_NAME
    ```

# 구현
## 서비스 실행
```bash
cd payment
mvn spring-boot:run
cd ..

cd prereservation
mvn spring-boot:run 
cd ..

cd receipt
mvn spring-boot:run
cd ..

cd gateway
mvn spring-boot:run
cd ..

cd myreservation
mvn spring-boot:run
cd ..
```
## 서비스 공통 구현
- 각 서비스에서 데이터가 생성되거나 변경될 때 관련 정보를 업데이트 한다.
```java
    @PrePersist
    public void onPrePersist() {
        createdAt = new Date();
        modifiedAt = createdAt;
    }

    @PreUpdate
    public void onPreUpdate() {
        modifiedAt = new Date();
    }

```
## event-driven
- 데이터가 변경되면 kafka에 메세지를 게시한다.
- 다른 서비스를 kafka의 메세지를 받아 관련 로직을 처리한다.
- payment 생성과 이와 관련된 receipt 생성
    - payment 서비스의 payment.java에서 이벤트 발행
        ```java
            @PostPersist
            public void onPostPersist(){
                PaymentCreated paymentCreated = new PaymentCreated();
                BeanUtils.copyProperties(this, paymentCreated);
                paymentCreated.publishAfterCommit();

            }
        ```
    - receipt의 PolicyHandler.java에서 kafka에 발행된 이벤트를 받아 영수증 생성
        ```java
            final private int DELIVERY_PREPER_DAY = 21;

            @StreamListener(KafkaProcessor.INPUT)
            public void wheneverPaymentCreated_PaymentCreated(@Payload PaymentCreated paymentCreated){

                if(!paymentCreated.validate()) return;

                System.out.println("\n\n##### listener PaymentCreated : " + paymentCreated.toJson() + "\n\n");

                Date expectedDeliveryAt = new Date();
                expectedDeliveryAt.setDate(paymentCreated.getCreatedAt().getDate() + DELIVERY_PREPER_DAY);

                Receipt receipt = new Receipt();
                receipt.setExpectedDeliveryAt(expectedDeliveryAt);
                receipt.setPaidId(paymentCreated.getId());
                BeanUtils.copyProperties(paymentCreated, receipt);
                receiptRepository.save(receipt);
            }
        ```

## correlation
- prereservation에 새로운 내용이 추가되면 prereservationCreated 이벤트룰 생성한다.
- prereservation이 생성될 때 payment 서비스에 생성 요청을 보낸다.
- payment 서비스는 payment가 생성되면 paymentCraeted 이벤트를 생성한다.
- receipt는 paymentCraeted 이벤트를 받아 receipt를 생성하고 receiptCreated 이벤트를 생성한다.
- myReservation은 prereservationCreated, paymentCreated, receiptCreated 이벤트를 받아 관련 정보를 업데이트 한다.
- http로 prereservation에 새로운 내용 추가
    ```bash
    http post localhost:8088/preReservations userId=1 userAddress="seoul" productId=1 productName=fold price=1300000 cardNo="1234-1234-1234-1234" status="order"
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132287475-438b8703-d9ea-4f86-a91b-b817d01b427d.png)
- prereservation 조회
    ```
    http localhost:8088/preReservations
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132287599-b2de11be-abaf-48cb-8ff5-cdf2af658df5.png)
- payment 조회
    ```
    http localhost:8088/payments
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132287644-e40b7da5-895e-41ca-a40a-674ddd351cd6.png)
- receipt 조회
    ```
    http localhost:8088/receipts
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132288723-084bd706-d1d9-4950-b199-98fb05726400.png)
- 카프카 메세지 확인
    ```
    $kafka_home/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic prereservation
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132288841-97186ea7-bc65-4a4e-a66a-575611c7c54c.png)

- prereservation에 예약 내역이 추가되면 서비스들끼리 이벤트를 주고받으면서 payment, receipt 정보가 생성되고 myReservation에서 모든 정보를 확인 할 수 있는 것을 볼 수 있다.

## CQRS
- CQRS는 명령 조회 책임 분리를 의미한다.
- myreservation(CQRS) 조회
    ```
    http localhost:8088/myReservations
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132288785-f1ca952c-7b04-476a-a204-1485d9b9e784.png)
## saga
- 특정 마이크로서비스에서의 작업이 실패하면 이전까지의 작업이 완료된 마이크서비스들에게 실패 이벤트를 보내 원자성(atomicity)을 보장하는 패턴
- 구현내용
    -  payment에서 작업 중 실패 시 prereservation 삭제 및 삭제 이벤트를 통해 이후 진행되었던 내용 삭제
        - prereservayion.java
            ```java
                    
            @PostPersist
            public void onPostPersist(){
                Payment payment = new Payment();
                try{
                    ReservationCreated reservationCreated = new ReservationCreated();
                    BeanUtils.copyProperties(this, reservationCreated);
                    reservationCreated.publishAfterCommit();

                    payment.setCardNo(cardNo);
                    payment.setPrice(price);
                    payment.setPreReservationId(id);
                    payment.setProductId(productId);
                    payment.setProductName(productName);
                    payment.setUserId(userId);
                    PrereservationApplication.applicationContext.getBean(prereservation.external.PaymentService.class).reservationCreated(payment);
            
                }catch(Exception e){
                    System.out.println("\n\n"+e+"\n\n");
                    PrereservationApplication.applicationContext.getBean(PreReservationRepository.class).delete(this);
                }

            }
            @PostRemove
            public void onPostRemove(){
                
                status = "reservation_cancelled";
                ReservationCancelled reservationCanceled = new ReservationCancelled();
                BeanUtils.copyProperties(this, reservationCanceled);
                reservationCanceled.publishAfterCommit();

            }
            ```
    - payment 서비스에 장애가 있는 경우에 주고받는 메세지
        - http로 prereservation에 새로운 내용 추가
            ```bash
            http post localhost:8088/preReservations userId=1 userAddress="seoul" productId=1 productName=fold price=1300000 cardNo="1234-1234-1234-1234" status="order"
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132290251-8ba023c6-9a05-4487-b36f-3fa0191f7daa.png)
        - 카프카 메세지 확인
            ```
            $kafka_home/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic prereservation
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132290295-0f410505-5cfd-4ea2-89c2-4657cfc34490.png)
        - myreservation(CQRS) 조회
            ```
            http localhost:8088/myReservations
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132290345-3979fe6b-fa2c-4c22-9510-c724545f1758.png)
        - prereservation이 취소되어 내용이 업데이트 된 것을 확인할 수 있다.

### 예약내역 삭제
- http로 prereservation에 내용 삭제
    ```bash
    http delete localhost:8088/preReservations/1
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132288938-6db6bcb9-2779-476b-8c95-d3c5e72de48b.png)
- payment 조회
    ```
    http localhost:8088/payments
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132289006-fa03bfe9-cc36-4939-91cd-12d2691aec99.png)
- receipt 조회
    ```
    http localhost:8088/receipts
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132289091-04f3d1d3-35ba-4a84-a4a1-a9aacb704dd1.png)
- myreservation(CQRS) 조회
    ```
    http localhost:8088/myReservations
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132289130-d8a75982-38c1-4242-968f-358d903f2abd.png)
- 카프카 메세지 확인
    ```
    $kafka_home/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic prereservation
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132288981-a743b8b3-8d7c-440b-8e6c-fc8a45d3b45d.png)
- 예약 내역을 삭제하면 payments, receipts 정보가 삭제되고 myReservations에 내용이 업데이트 되는 것을 볼 수 있다.

## req/res
- prereservation과 payment통신
    - FeginClient를 이용해 req/res를 구현한다.
    - PrereservationApplication.java에서 FeignClients활성화
        ```java
        @EnableFeignClients
        ```

    - FeignClients를 이용해 req/res를 수핼할 메서드 생성
        ```java
        @FeignClient(name="payment", url="http://localhost:8081")
        public interface PaymentService {
            @RequestMapping(method= RequestMethod.POST, path="/payments")
            public void reservationCreated(@RequestBody Payment payment) throws Exception;
        }
        ```
- 만약 payment 서버와 req/res 통신이 불가능하면 다음과 같은 에러가 발생한다.
    ![image](https://user-images.githubusercontent.com/53825723/132290566-a7399f1e-3a9d-485a-8a31-79f0bbe292c3.png)

## fallback
- fallback을 이용해 payment 서버와 req/res 통신이 불가능할 때 실행할 메서드를 지정할 수 있다.
- fallback 구현 
    - application.yaml
        ```yaml
        feign:
            hystrix:
                enabled: true
        ```
    - PaymentService.java
        ```java
        @FeignClient(name="payment", url="http://localhost:8081", fallback=PaymentServiceImpl.class) // FALLBAK 설정
        public interface PaymentService {)
            @RequestMapping(method= RequestMethod.POST, path="/payments")
            public void reservationCreated(@RequestBody Payment payment) throws Exception;
        }
        ```
    - PaymentServiceImpl.java (fallback 메서드)
        ```java  
        @Service
        public class PaymentServiceImpl implements PaymentService{

            @Override
            public void reservationCreated(Payment payment) throws Exception {
                // TODO Auto-generated method stub
                System.out.println("\n\n\n결제 서비스 과부화");
                System.out.println("\n결제 서비스 과부화");
                System.out.println("\n결제 서비스 과부화\n");
                throw new Exception();
            }
            
        }
        ```
- 결과

    ![image](https://user-images.githubusercontent.com/53825723/132290891-11597b37-cda5-42dd-9056-4a6ddefcea9a.png)
        - 지정한 fallback 메서드가 실행되는 것을 볼 수 있다.
## gateway
- gateway는 gateway로 들어오는 요청을 path에 따라 특정 서비스로 보내는 역할을 한다.
- 즉, 사용자는 하나의 주소로 서비스들을 사용할 수 있게 해준다.
- application.yaml
    ```yaml
    spring:
    profiles: default
    cloud:
        gateway:
        routes:
            - id: payment
            uri: http://localhost:8081
            predicates:
                - Path=/payments/** 
            - id: prereservation
            uri: http://localhost:8082
            predicates:
                - Path=/preReservations/** 
            - id: receipt
            uri: http://localhost:8083
            predicates:
                - Path=/receipts/** 
            - id: myreservation
            uri: http://localhost:8084
            predicates:
                - Path= /myReservations/**
    ```
- pom.xml
    ```xml
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-gateway</artifactId>
            </dependency>
    ```
- `http://localhost:8088/payments/` 으로 요청하면 `http://localhost:8081/payments/` 으로 요청을 보내고 `http://localhost:8088/preReservations/`으로 요청을 보내면 `http://localhost:8082/preReservations/`으로 요청을 보낸다.

# 운영
## kafka
-  헬름 설치
    ```
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

- kubernetes에 카프카 설치
    ```
    helm repo add incubator https://charts.helm.sh/incubator
    helm repo update
    kubectl create ns kafka
    helm install my-kafka --namespace kafka incubator/kafka
    ```
- 카프카 설치 확인
    ```
    kubectl get po -n kafka -o wide
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132285098-ee993efc-7fdc-4ebb-a83a-1c6ec3e52732.png)

## deploy
- ACR의 이미지를 사용해 배포하도록 deployment.yaml 이미지 경로 수정
    - 수정 전
        ```yaml
                - name: payment
                  image: username/payment:latest
        ```
    - 수정 후
        ```yaml
                - name: payment
                  image: <ACR_NAME>.auzrecr,io/payment:latest
                  imagePullPolicy: Always
        ```

- FeignClient URL 매개변수화
    - PaymentService.java
        ```java
        @FeignClient(name="payment", url="${url.payment}", fallback=PaymentServiceImpl.class) // FALLBAK 설정
        ```
    - application.ymal
        ```yaml
        url:
            payment: http://localhost:8081
        ---
        url:
            payment: http://payment:8080

        ```
- 빌드 및 배포

    ```bash

    cd payment
    # jar 파일 생성
    mvn package
    # 이미지 빌드
    docker build -t user02.azurecr.io/payment .
    # acr에 이미지 푸시
    docker push user02.azurecr.io/payment
    # kubernetes에 service, deployment 배포
    kubectl apply -f kubernetes
    # Pod 재배포 
    # Deployment가 변경되어야 새로운 이미지로 Pod를 실행한다.
    # Deployment가 변경되지 않아도 새로운 Image로 Pod 실행하기 위함
    kubectl rollout restart deployment payment  
    cd ..

    cd prereservation
    mvn package
    docker build -t user02.azurecr.io/prereservation .
    docker push user02.azurecr.io/prereservation
    kubectl apply -f kubernetes
    kubectl rollout restart deployment prereservation  
    cd ..

    cd receipt
    mvn package
    docker build -t user02.azurecr.io/receipt .
    docker push user02.azurecr.io/receipt
    kubectl apply -f kubernetes
    kubectl rollout restart deployment receipt  
    cd ..

    cd gateway
    mvn package
    docker build -t user02.azurecr.io/gateway .
    docker push user02.azurecr.io/gateway
    kubectl create deploy gateway --image=user02.azurecr.io/gateway   
    kubectl expose deploy gateway --type=LoadBalancer --port=8080 

    kubectl rollout restart deployment gateway
    cd ..

    cd myreservation
    mvn package
    docker build -t user02.azurecr.io/myreservation .
    docker push user02.azurecr.io/myreservation
    kubectl apply -f kubernetes
    kubectl rollout restart deployment myreservation  
    cd ..


    ```

- 배포 결과 확인
    ```bash
    kubectl get all
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132285865-a874533b-c9ad-4449-b17f-c1536a377d2b.png)

- 테스트
    - gateway에 요청
        ```bash
        http 20.200.200.225:8080
        ```
        ![image](https://user-images.githubusercontent.com/53825723/132293909-93a8c789-5684-4dfe-81ae-ea2f7c7e0cf8.png)
    - http로 prereservation에 새로운 내용 추가
        ```bash
        http post 20.200.200.225:8080/preReservations userId=1 userAddress="seoul" productId=1 productName=fold price=1300000 cardNo="1234-1234-1234-1234" status="order"
        ```
        ![image](https://user-images.githubusercontent.com/53825723/132299133-e301a1ca-3078-4878-9ebb-19f15ad2e0cd.png)
    - myreservation(CQRS) 조회
        ```
        http 20.200.200.225:8080/myReservations
        ```
        ![image](https://user-images.githubusercontent.com/53825723/132299202-e3259409-90a8-467c-b31a-ea5e7210273e.png)

## configMap
- prereservation에 configMap을 사용하여 활성 프로파일을 설정한다.
- dockerfile 수정
    ```dockerfile
    FROM openjdk:8u212-jdk-alpine
    COPY target/*SNAPSHOT.jar app.jar
    EXPOSE 8080
    ENTRYPOINT ["java","-Xmx400M","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar","--spring.profiles.active=${PROFILE}"]
    ```
- configMap 생성
    ```bash
    kubectl create configmap profile-cm --from-literal=profile=docker
    ```
    ```bash
    kubectl get cm profile-cm -o yaml 
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132285964-1d3e6039-21d9-429a-9afd-974d67168cbb.png)
- deployment.yml에 내용추가
    ```yaml
    ...
            imagePullPolicy: Always
            env:
            - name: PROFILE
                valueFrom:
                configMapKeyRef:
                    name: profile-cm
                    key: profile
            ports:
    ...
    ```
- 배포후 실행 로그 확인
    ```
    kubectl logs pod/prereservation-65f457c9b8-4289f
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132286372-923ae802-6134-411f-8a35-04ad3f6ef7e0.png)

- 컨테이너 내부의 환경변수 확인
    ```
    kubectl exec pod/prereservation-65f457c9b8-4289f -it -- sh
    / # env
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132286517-5a2d57db-a25b-4e08-ad8c-500c7655e6dd.png)

## siege 설치
- kubernetes에서 부하테스트를 위한 seige 설치
- siege.yaml
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: siege
    spec:
    containers:
    - name: siege
        image: apexacme/siege-nginx
    ```
- siege pod 배포
    ```bash
    kubectl apply -f siege.yaml
    ```
- siege pod 배포 확인
    ```
    kubectl get pod siege
    ```
    ![image](https://user-images.githubusercontent.com/53825723/132304430-c976de45-b610-4ad2-8495-d4d137303619.png)
- siege 접속
    ```
    kubectl exec -it pod/siege -c siege -- /bin/bash
    ```
- siege 사용법 예시 (myreservation에 워크로드를 1000명, 1분간 걸어준다.)
    ```
    siege -c1000 -t60S  -v http://myreservation:8080/myReservations
    ```
## circuit breaker
- 클라이언트에서 원격 서버로 요청을 전송할 때 특정 임계치(threshold)를 넘어서면, 원격 서버에 문제가 있다고 판단하여 빠르게 에러를 발생시키는 방법 (Fail-Fast)
- 일정시간 동안 특정 마이크로 서비스에 대한 요청이 많이 생기면 서비스가 다운될 수 있다.
- 이를 방지하기 위해 다운 되기 전 요청을 차단한다. 
- circuit breaker 설정
    - payment 서비스와 req/res 요청을 하는 prereservation에 타임 아웃 설정을 추가한다. ( 1500 밀리 초 )
        - application.yaml
            ```yaml 
            hystrix:
                command:
                    default:
                        execution:
                            isolation:
                                thread:
                                    timeoutInMilliseconds: 620
            ```
    - payment 서비스의 응답이 1초 이상 걸린다고 가정하고 코드를 수정한다.
        - Payment.java
            ```java
                @PostPersist
                public void onPostPersist(){
                    try {
                        Thread.sleep((long) (400 + Math.random() * 220));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    PaymentCreated paymentCreated = new PaymentCreated();
                    BeanUtils.copyProperties(this, paymentCreated);
                    paymentCreated.publishAfterCommit();
                }
            ```
    - 재배포 후 siege에서 부하 테스트를 진행한다.
        ```bash
        kubectl exec -it pod/siege -c siege -- /bin/bash
        siege -c2 -t60S -v  --content-type "application/json" 'http://prereservation:8080/preReservations POST {"userId":1}'
        ```
        ![image](https://user-images.githubusercontent.com/53825723/132346929-66eef274-db77-4332-8526-923d0c471c8b.png)
    - prereservation 서비스 로그 확인
        ```
        kubectl log prereservation-6c7c5855d4-xsn69
        ```
        ![image](https://user-images.githubusercontent.com/53825723/132345636-a9262a9f-5941-4458-9972-255a5f32f2f6.png)
    - 시간이 오래 걸리면 회로가 차단되는 것을 볼 수 있다.

## Autoscale (HPA)
- 일정시간 동안 특정 Pod에 요청이 많이 발생해 과부화가 생기는 경우 Pod의 수를 늘려 요청을 분산할 수 있다.
- kubernetes의 HPA는 pod를 모니터링 하고 설정된 리소스가 기준을 넘어서 일정시간 유지되면 Pod의 수를 늘린다.
- HPA 적용
    - prereservation의 Deployment에 리소스 기준 생성
        - 최대 CPU : 500m, 최소 CPU : 200m 
        ```
            imagePullPolicy: Always
            resources:
                limits:
                    cpu: 500m
                requests:
                    cpu: 200m
        ```
    - HPA 생성
        - prereservation의 replica를 동적으로 늘려주도록 HPA 를 설정한다. 
        - 아래 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다
        ```
        kubectl autoscale deployment prereservation --cpu-percent=15 --min=1 --max=10
        ```
        ```
        kubectl get hpa
        ```
        ![image](https://user-images.githubusercontent.com/53825723/132349467-082798a3-00f1-41c2-9dbc-127687de326c.png)
    - siege로 부하 테스트
        ```bash
        kubectl exec -it pod/siege -c siege -- /bin/bash
        siege -c2 -t60S -v  --content-type "application/json" 'http://prereservation:8080/preReservations POST {"userId":1}'
        ```
    - Pod 수 변화
        - 부하 테스트 전
            ```
            kubectl get deploy prereservation -w
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132349579-939a973c-c70a-4395-b0a4-c2ec78e9d295.png)
        - 부하 테스트 후
            ![image](https://user-images.githubusercontent.com/53825723/132349762-dd0d3995-3a7d-4600-b024-4650cf98238d.png)
        - 다음 명령어로도 실시간으로 변하는 것을 확인할 수 있다.
            ```
            watch kubectl get pod
            ```

## Zero-downtime deploy (readiness probe)
- 서비스가 실행된 후 사용자에게 기능을 제공하기까지 시간이 걸릴 수 있다.
- Readiness가 설정되지 않으면 새로운 Pod가 배포될 때 서비스가 실행되면 Kubernetes는 pod를 Running 상태로 전환하고 새로운 pod는 Service를 통해 노출한다.
- 기존 Pod는 삭제된다.
- pod는 준비가 되지 않은 상태로 사용자의 요청을 받을 수 있다.
- 이는 재배포 시 사용자 입장에서 중단이 발생할 수 있음을 의미한다.
    - Readness 설정 없이 siege로 지속적인 접속 테스트시 재배포
        - 재배포 명령어
            ```
            kubectl rollout restart deployment prereservation  
            ```
        - siege로 부하테스트
            ```
            kubectl exec -it pod/siege -c siege -- /bin/bash
            siege -c255 -t30S -v  --content-type "application/json" 'http://prereservation:8080/preReservations POST {"userId":1}'
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132357251-d51a721b-6936-4ca0-b706-c26d196205ba.png)
        - 재배포시 서비스 중단이 발생한다.


- 만약 Readiness 설정이 되어 있으면 Kubernetes는 Pod가 사용자의 요청이 준비될 때 까지 기다린 후 Service에 연결한다.
- 기존 Pod는 새로운 Pod가 준비된 후 삭제된다.
- 즉, service를 통해 pod로 요청하는 사용자 입장에서는 중단이 발생하지 않는다.
    - Readinss 설정 후 siege로 지속적인 접속 테스트시 재배포
        - Readinss 설정
            ```yaml
            readinessProbe:
              httpGet:
                path: '/actuator/health'
                port: 8080
              initialDelaySeconds: 10
              timeoutSeconds: 2
              periodSeconds: 5
              failureThreshold: 10
            ```
        - 재배포 명령어
            ```
            kubectl rollout restart deployment prereservation  
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132354529-71440a29-238a-43f5-85f1-023b5fc06018.png)
        - siege로 부하테스트
            ```
            kubectl exec -it pod/siege -c siege -- /bin/bash
            siege -c255 -t30S -v  --content-type "application/json" 'http://prereservation:8080/preReservations POST {"userId":1}'     
            ```
            ![image](https://user-images.githubusercontent.com/53825723/132356700-ec66c57a-c23e-4976-995c-3cf4e3dd45c6.png)
        - 재배포시 서비스 중단이 발생하지 않는다.

## self-healing (liveness probe)
- 마이크로서비스가 9090 포트를 통해 기능을 제공한다고 가정
- liveness를 사용해 9090포트가 정상 작동하는지 주기적으로 확인
    ```yaml
          livenessProbe:
            httpGet:
              path: '/actuator/health'
              port: 9090
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 5
    ```
- 만약 정상 작동하지 않는다면 pod를 재시작한다.
![image](https://user-images.githubusercontent.com/53825723/132358236-45cb30c9-cf23-403b-82ec-62dfb364d6f4.png)
    - 9090포트로 통신할 수 없어 계속해서 재시작 하는 모습을 볼 수 있다.
    - liveness를 사용하지 않는다면, 9090포트가 비정상적으로 작동해도 계속해서 Running 상태이다.

- 현제 프로젝트에서는 8080 포트를 사용하므로 deployment에 8080포트에 대한 liveness 설정을 추가한다.
    ```yaml
      livenessProbe:
        httpGet:
          path: '/actuator/health'
          port: 8080
        initialDelaySeconds: 120
        timeoutSeconds: 2
        periodSeconds: 5
        failureThreshold: 5
    ```