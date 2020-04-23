- Rental Service -

개요 : 전자 제품 랜탈을 위한 주문, 결재, 재고관리, 배송, 통합 상황 정보 제공 



# Table of contents

- [예제 - Rental Service](#---)
  - [서비스 시나리오](#서비스 시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)

# 서비스 시나리오

시나리오 : 고객이 랜탈 요청을 하면 주문이 요청 되는데 주문은 반드시 재고 확인(Fegin Client) 후 가능함
                 재고 확인 후 결재 요청을 하게 되고 결재 확인이 되면 
                 랜탈 제품에 대한 배송을 시작함
                이런 모든 과정은 totalView(CQRS)로 저장 되어 조회 가능함
                고객이 별도의 취소가 가능하며
                제고가 부족할 때 임의로 모든 처리를 Cancel하는 Saga 패턴을 적용함


기능적 요구사항
1. 고객이 랜탈 요청
1. 재고 확인후 결재 확인
1. 배송 출발한다
1. 고객이 랜탈을 취소할 수 있다
1. 랜탈 취소되면 배송이 취소된다
1. 고객이 진행상태를 중간중간 조회한다(CQRS 적용)
1. 재고 부족시 주문이 자동 취소 된다
1. 이떄 재고 수량 변경, 환불, 제품회수 처리 (Saga 패턴 적용)


비기능적 요구사항
1. 트랜잭션
    1. 시스템 재고 확인(필수 Step 으로 동기 방식 필요)
1. 장애격리
    1. 재고 확인 외 기능은 타 시스템 동작 안해도 서비스 가능 Async (event-driven), Eventual Consistency
1. 성능
    1. 고객이 진행상태를 확인할 수 있는 Total View제공 되어야 한다  CQRS


# 분석/설계


* 이벤트스토밍 결과:  <img width="1185" alt="스크린샷 2020-04-23 오전 9 11 59" src="https://user-images.githubusercontent.com/61147091/80045512-9a393e80-8542-11ea-8aac-73ecf550a6fb.png">



## 헥사고날 아키텍처 다이어그램 도출

<img width="1212" alt="스크린샷 2020-04-23 오전 10 59 03" src="https://user-images.githubusercontent.com/61147091/80050805-9a8d0600-8551-11ea-99b5-e03455862fc6.png">

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

MAS 구현 소스는 다음과 같이 6개 레퍼지토리에 저장 되어 있음
1.https://github.com/Park-dc/gateway.git
2.https://github.com/Park-dc/bill_collection.git
3.https://github.com/Park-dc/order_manager.git
4.https://github.com/Park-dc/inventory_manager.git
5.https://github.com/Park-dc/logistics_manage.git
6.https://github.com/Park-dc/totalView.git


```
cd order_manage
mvn spring-boot:run

cd inventory_manager
mvn spring-boot:run 

cd bill_collection
mvn spring-boot:run  

cd logistics_manage
mvn spring-boot:run  

cd totalView
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 order_manager 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 

```
package rentalsvc;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private Long productId;


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
    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package rentalsvc;

@Transactional
public interface OrderRepository extends CrudRepository<Order, Long>{

	Long deleteByOrderId(Long orderId);

}
```
- 적용 후 REST API 의 테스트
```
# Order 서비스의 주문처리
http post http://52.231.107.179:8080/rentalRequest\?orderId\=2\&productId\=2\&qty\=10\&amount\=10

--> 주문 처리됨 이벤트
{"eventType":"RentalRequested","timestamp":"20200423013634","id":null,"orderId":2,"productId":2,"qty":10,"amount":10,"me":true}  
--> 요금수납됨 이벤트
{"eventType":"FeeReceived","timestamp":"20200423013634","id":null,"amount":10,"orderId":2,"me":true} 
--> 배송시작됨 이벤트
{"eventType":"DeliveryStarted","timestamp":"20200423013634","id":null,"orderId":2,"logistisId":null,"me":true} 


# Order 서비스의 주문취소
http get http://52.231.107.179:8080/rentalCancel\?productId\=1\&orderId\=1
--> 주문취소됨 이벤트 발생
{"eventType":"RentalCancellationOccured","timestamp":"20200423013817","id":null,"orderId":1,"productId":1,"me":true}
--> 요금반환됨 이벤트 발생
{"eventType":"FeeRefundCompleted","timestamp":"20200423013817","id":null,"amount":null,"orderId":1,"me":true}
--> 주문취소됨 이벤트 발생
{"eventType":"RetriveStarted","timestamp":"20200423013817","id":null,"orderId":1,"me":true}

# 재고 부족시 강제 주문 취소(각 product는 100개의 기본 재고 등록 됨 110개를 요청해봄)
http post http://52.231.107.179:8080/rentalRequest\?orderId\=2\&productId\=2\&qty\=110\&amount\=10
--> 주문요청됨 이벤트 발생
{"eventType":"RentalRequested","timestamp":"20200423015035","id":null,"orderId":2,"productId":2,"qty":110,"amount":10,"me":true}
--> 요금수남됨 이벤트 발생
{"eventType":"FeeReceived","timestamp":"20200423015035","id":null,"amount":10,"orderId":2,"me":true}
--> 배송시작됨 이벤트 발생
{"eventType":"DeliveryStarted","timestamp":"20200423015035","id":null,"orderId":2,"logistisId":null,"me":true}
--> 재고부족 이벤트 발생
{"eventType":"InventoryStockShortage","timestamp":"20200423015035","orderId":2,"productId":2,"me":true}
--> 환불 처리 이벤트 발생
{"eventType":"FeeRefundCompleted","timestamp":"20200423015035","id":null,"amount":null,"orderId":2,"me":true}
--> 배송회수 이벤트 
{"eventType":"RetriveStarted","timestamp":"20200423015035","id":null,"orderId":2,"me":true}


# 주문 상태 확인
http get http://52.231.107.179:8080/orders
--> 주문된 상태 정보 
{
    "_embedded": {
        "orders": [
            {
                "_links": {
                    "order": {
                        "href": "http://ordermanager:8080/orders/1"
                    },
                    "self": {
                        "href": "http://ordermanager:8080/orders/1"
                    }
                },
                "orderId": 1,
                "productId": 1
            },
            {
                "_links": {
                    "order": {
                        "href": "http://ordermanager:8080/orders/5"
                    },
                    "self": {
                        "href": "http://ordermanager:8080/orders/5"
                    }
                },
                "orderId": 2,
                "productId": 2
            },
            {
                "_links": {
                    "order": {
                        "href": "http://ordermanager:8080/orders/6"
                    },
                    "self": {
                        "href": "http://ordermanager:8080/orders/6"
                    }
                },
                "orderId": 2,
                "productId": 2
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://ordermanager:8080/profile/orders"
        },
        "search": {
            "href": "http://ordermanager:8080/orders/search"
        },
        "self": {
            "href": "http://ordermanager:8080/orders"
        }
    }
}

# 재고 상태 확인
http get http://52.231.107.179:8080/inventories            --> 전체
http get http://52.231.107.179:8080/inventories/1          --> productId 1번 재고 확인

--> productId 1
{
    "_links": {
        "inventory": {
            "href": "http://inventorymanager:8080/inventories/1"
        },
        "self": {
            "href": "http://inventorymanager:8080/inventories/1"
        }
    },
    "productId": 1,
    "productName": "IPAD",
    "qty": 70
}



```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문->재고확인 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (재고관리) InventoryService.java

package rentalsvc.external;
...
@Service
@FeignClient(name = "inventoryCheck", url = "http://inventorymanager:8080")
public interface InventoryService {

    @RequestMapping(method = RequestMethod.GET, path = "/inventoryCheck")
    public Long inventoryCheck(@RequestParam("productId") Long productId);

}
```

- 재고 확인 직후 결제를 요청하도록 처리
```
	    
@RequestMapping(value = "/rentalRequest",
        method = RequestMethod.POST,
        produces = "application/json;charset=UTF-8")

public void rentalRequest(@RequestParam Long orderId,@RequestParam Long productId,@RequestParam Long qty,@RequestParam Long amount)
        throws Exception {
        System.out.println("##### /order/rentalRequest  called #####");
        
.....
                
        Long dd = Application.applicationContext.getBean(InventoryService.class).inventoryCheck(productId);
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 재고(Inventory) 서비스를 잠시 내려놓음 (ctrl+c)

#주문처리
http post http://localhost:8081/rentalRequest\?orderId\=2\&productId\=2\&qty\=10\&amount\=10   #Fail

#재고서비스 재기동
#주문처리
http post http://localhost:8081/rentalRequest\?orderId\=2\&productId\=2\&qty\=10\&amount\=10   #Success
```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


재고확인 후에 결재시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 기타 시스템의 처리가 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 재고 확인 후에 곧바로 랜트요청 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
       Order order = new Order();
        RentalRequested rentalRequested = new RentalRequested();
        rentalRequested.setOrderId(orderId);
        rentalRequested.setProductId(productId);
        rentalRequested.setQty(qty);
        rentalRequested.setAmount(amount);
        BeanUtils.copyProperties(this, rentalRequested);
        rentalRequested.publish();
```
-  비용 서비스에서는 랜트요청승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package rentalsvc;
...

 @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRentalRequestOccured_ChargeFree(@Payload RentalRequested rentalRequested){

        if(rentalRequested.isMe()){
            System.out.println("##### listener  랜트요청 이벤트 수신(RentalRequested) : " + rentalRequested.toJson());
            
           FeeReceived feeReceived = new FeeReceived();
            
	           feeReceived.setOrderId(rentalRequested.getOrderId());
	           feeReceived.setAmount(rentalRequested.getAmount());
	           feeReceived.publish();
	            
           System.out.println("##### 요금수납 완료 이벤트 발송 : (FeeReceived)");
        }
    }


# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 GCP를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 azure_pipeline.yml 에 포함되었다.

order_manager CI/CD를 위한 yamul파일

# Deploy to Azure Kubernetes Service
# Build and push image to Azure Container Registry; Deploy to Azure Kubernetes Service
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- master

resources:
- repo: self

variables:
- group: common-value
  # containerRegistry: 'event.azurecr.io'
  # containerRegistryDockerConnection: 'acr'
  # environment: 'aks.default'
- name: imageRepository
  value: 'ordermanager'
- name: dockerfilePath
  value: '**/Dockerfile'
- name: tag
  value: '$(Build.BuildId)'
  # Agent VM image name
- name: vmImageName
  value: 'ubuntu-latest'
- name: MAVEN_CACHE_FOLDER
  value: $(Pipeline.Workspace)/.m2/repository
- name: MAVEN_OPTS
  value: '-Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'


stages:
- stage: Build
  displayName: Build stage
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: CacheBeta@1
      inputs:
        key: 'maven | "$(Agent.OS)" | **/pom.xml'
        restoreKeys: |
           maven | "$(Agent.OS)"
           maven
        path: $(MAVEN_CACHE_FOLDER)
      displayName: Cache Maven local repo
    - task: Maven@3
      inputs:
        mavenPomFile: 'pom.xml'
        options: '-Dmaven.repo.local=$(MAVEN_CACHE_FOLDER)'
        javaHomeOption: 'JDKVersion'
        jdkVersionOption: '1.8'
        jdkArchitectureOption: 'x64'
        goals: 'package'
    - task: Docker@2
      inputs:
        containerRegistry: $(containerRegistryDockerConnection)
        repository: $(imageRepository)
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
        tags: |
          $(tag)

- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build

  jobs:
  - deployment: Deploy
    displayName: Deploy
    pool:
      vmImage: $(vmImageName)
    environment: $(environment)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Kubernetes@1
            inputs:
              connectionType: 'Kubernetes Service Connection'
              namespace: 'default'
              command: 'apply'
              useConfigurationFile: true
              configurationType: 'inline'
              inline: |
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: $(imageRepository)
                  labels:
                    app: $(imageRepository)
                spec:
                  replicas: 1
                  selector:
                    matchLabels:
                      app: $(imageRepository)
                  template:
                    metadata:
                      labels:
                        app: $(imageRepository)
                    spec:
                      containers:
                        - name: $(imageRepository)
                          image: $(containerRegistry)/$(imageRepository):$(tag)
                          ports:
                            - containerPort: 8080
                          readinessProbe:
                            httpGet:
                              path: /actuator/health
                              port: 8080
                            initialDelaySeconds: 10
                            timeoutSeconds: 2
                            periodSeconds: 5
                            failureThreshold: 10
                          livenessProbe:
                            httpGet:
                              path: /actuator/health
                              port: 8080
                            initialDelaySeconds: 120
                            timeoutSeconds: 2
                            periodSeconds: 5
                            failureThreshold: 5
              secretType: 'dockerRegistry'
              containerRegistryType: 'Azure Container Registry'
          - task: Kubernetes@1
            inputs:
              connectionType: 'Kubernetes Service Connection'
              namespace: 'default'
              command: 'apply'
              useConfigurationFile: true
              configurationType: 'inline'
              inline: |
                apiVersion: v1
                kind: Service
                metadata:
                  name: $(imageRepository)
                  labels:
                    app: $(imageRepository)
                spec:
                  ports:
                    - port: 8080
                      targetPort: 8080
                  selector:
                    app: $(imageRepository)
              secretType: 'dockerRegistry'
              containerRegistryType: 'Azure Container Registry'

