# 과제
## 1. Event Storming

![image](https://user-images.githubusercontent.com/8733715/212526240-80400cae-0933-47ab-a90d-7e9caf70aa9f.png)




# 구현 : 1. saga : 소스코드 복사해서 붙여넣기  (일부코드 복사)

```
package food.deliverymy.infra;

import javax.naming.NameParser;

import javax.naming.NameParser;
import javax.transaction.Transactional;

import food.deliverymy.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;
import food.deliverymy.domain.*;

@Service
@Transactional
public class PolicyHandler{
    @Autowired OrderRepository orderRepository;
    @Autowired PaymentRepository paymentRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void whatever(@Payload String eventString){}

    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='Rejected'")
    public void wheneverRejected_CancelPayment(@Payload Rejected rejected){

        Rejected event = rejected;
        System.out.println("\n\n##### listener CancelPayment : " + rejected + "\n\n");

        // Sample Logic //
        Payment.cancelPayment(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='OrderCanceled'")
    public void wheneverOrderCanceled_CancelPayment(@Payload OrderCanceled orderCanceled){

        OrderCanceled event = orderCanceled;
        System.out.println("\n\n##### listener CancelPayment : " + orderCanceled + "\n\n");

        // Sample Logic //
        Payment.cancelPayment(event);        

    }

    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='Accepted'")
    public void wheneverAccepted_UpdateStatus(@Payload Accepted accepted){

        Accepted event = accepted;
        System.out.println("\n\n##### listener UpdateStatus : " + accepted + "\n\n");

        // Sample Logic //
        Order.updateStatus(event);
    }
    
    @StreamListener(value=KafkaProcessor.INPUT, condition="headers['type']=='Rejected'")
    public void wheneverRejected_UpdateStatus(@Payload Rejected rejected){

        Rejected event = rejected;
        System.out.println("\n\n##### listener UpdateStatus : " + rejected + "\n\n");

        // Sample Logic //
        Order.updateStatus(event);
    }
}
```


# 구현 : 2. CQRS : 

![image](https://user-images.githubusercontent.com/8733715/212527690-4dd4d06d-7996-4d51-b6a3-f8dade8cc889.png)

```
package food.deliverymy.infra;

import food.deliverymy.domain.*;
import food.deliverymy.config.kafka.KafkaProcessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.List;
import java.util.Optional;

@Service
public class OrderStatusViewHandler {

    @Autowired
    private OrderStatusRepository orderStatusRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1 (@Payload OrderPlaced orderPlaced) {
        try {

            if (!orderPlaced.validate()) return;

            // view 객체 생성
            OrderStatus orderStatus = new OrderStatus();
            // view 객체에 이벤트의 Value 를 set 함
            orderStatus.setId(orderPlaced.getId());
            orderStatus.setFoodId(orderPlaced.getFoodId());
            orderStatus.setStatus("주문됨");
            // view 레파지 토리에 save
            orderStatusRepository.save(orderStatus);

        }catch (Exception e){
            e.printStackTrace();
        }
    }


    @StreamListener(KafkaProcessor.INPUT)
    public void when_then_UPDATE_(@Payload  ) {
        try {
            if (!.validate()) return;
                // view 객체 조회
            Optional<OrderStatus> orderStatusOptional = orderStatusRepository.findById(.getOrderId());

            if( orderStatusOptional.isPresent()) {
                 OrderStatus orderStatus = orderStatusOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderStatus.setStatus("결재됨");    
                // view 레파지 토리에 save
                 orderStatusRepository.save(orderStatus);
                }


        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenAccepted_then_UPDATE_2(@Payload Accepted accepted) {
        try {
            if (!accepted.validate()) return;
                // view 객체 조회
            Optional<OrderStatus> orderStatusOptional = orderStatusRepository.findById(accepted.getOrderId());

            if( orderStatusOptional.isPresent()) {
                 OrderStatus orderStatus = orderStatusOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderStatus.setStatus("접수됨");    
                // view 레파지 토리에 save
                 orderStatusRepository.save(orderStatus);
                }


        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenDeliveryStarted_then_UPDATE_3(@Payload DeliveryStarted deliveryStarted) {
        try {
            if (!deliveryStarted.validate()) return;
                // view 객체 조회
            Optional<OrderStatus> orderStatusOptional = orderStatusRepository.findById(deliveryStarted.getOrderId());

            if( orderStatusOptional.isPresent()) {
                 OrderStatus orderStatus = orderStatusOptional.get();
            // view 객체에 이벤트의 eventDirectValue 를 set 함
                orderStatus.setStatus("배달시작됨");    
                // view 레파지 토리에 save
                 orderStatusRepository.save(orderStatus);
                }


        }catch (Exception e){
            e.printStackTrace();
        }
    }


}


```
# 구현 : 3. Compensation / Correlation : 소스 넣기

### StoreOrderRepository.java
```
package food.deliverymy.domain;

import food.deliverymy.domain.*;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;
import java.util.Optional;

@RepositoryRestResource(collectionResourceRel="storeOrders", path="storeOrders")
public interface StoreOrderRepository extends PagingAndSortingRepository<StoreOrder, Long>{
    Optional<StoreOrder> findByOrderId(Long orderId);

}

```

### StoreOrder.java
```
    public static void notifyOrderCancelled(OrderCanceled orderCanceled){

        /** Example 1:  new item 
        StoreOrder storeOrder = new StoreOrder();
        repository().save(storeOrder);

        */

        /** Example 2:  finding and process */
        
        repository().findById(orderCanceled.getId()).ifPresent(storeOrder->{
            storeOrder.setStatus("주문 취소");  // do something
            repository().save(storeOrder);
         });
    }

```



