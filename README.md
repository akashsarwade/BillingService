# WkflBillingOrder - Complete Kafka Architecture & Implementation Guide ## Overview The WkflBillingOrder microservice uses **Apache Kafka** as its event streaming platform to support asynchronous communication between multiple billing subsystems. This guide explains how Kafka is configured, used, and integrated into the billing order orchestration workflow. --- ## TABLE OF CONTENTS 1. [Kafka Architecture Overview](#1-kafka-architecture-overview) 2. [Spring Cloud Stream Integration](#2-spring-cloud-stream-integration) 3. [Event Channels Configuration](#3-event-channels-configuration) 4. [Event Producers](#4-event-producers) 5. [Event Consumers](#5-event-consumers) 6. [Complete Event Flow](#6-complete-event-flow) 7. [Multi-Tenant Cloud & Comcast Strategy](#7-multi-tenant-cloud--comcast-strategy) 8. [Kafka Binders and Transport](#8-kafka-binders-and-transport) 9. [Error Handling & Resilience](#9-error-handling--resilience) 10. [Configuration & Properties](#10-configuration--properties) --- # 1. KAFKA ARCHITECTURE OVERVIEW ## 1.1 What is Kafka in This Project? Kafka is used as an **event streaming backbone** for asynchronous, event-driven communication in the WkflBillingOrder microservice. ### Why Kafka? - **Decoupling**: Services communicate through events, not direct API calls - **Scalability**: Handles high-volume order events asynchronously - **Durability**: Events are persisted for replay and recovery - **Multi-tenancy**: Separate topics for Cloud and Comcast networks - **Resilience**: Fallback mechanisms ensure no message loss ### Kafka Topics in WkflBillingOrder The service uses **multiple Kafka topics** organized by event type and network: | Topic Name | Type | Purpose | Network | |-----------|------|---------|---------| | billing-fulfillment-events-cloud | Input | Receive fulfillment completion events | Cloud | | billing-fulfillment-events-comcast | Input | Receive fulfillment completion events | Comcast | | billing-provisioning-events-cloud | Input | Receive device provisioning events | Cloud | | billing-provisioning-events-comcast | Input | Receive device provisioning events | Comcast | | billing-writeback-events-cloud | Input | Receive AMDOCS sync requests | Cloud | | billing-writeback-events-comcast | Input | Receive AMDOCS sync requests | Comcast | | wkfl-provisioning-request-cloud | Output | Publish provisioning orders to Fulfillment | Cloud | | wkfl-provisioning-request-comcast | Output | Publish provisioning orders to Fulfillment | Comcast | | wkfl-cancel-order-request-cloud | Input | Receive cancel order requests | Cloud | | wkfl-cancel-order-request-comcast | Input | Receive cancel order requests | Comcast | | wkfl-cancel-order-response-cloud | Output | Publish cancel responses | Cloud | | wkfl-cancel-order-response-comcast | Output | Publish cancel responses | Comcast | --- ## 1.2 Kafka Broker Configuration The application connects to **three different Kafka brokers** depending on environment: ### Binder Types:
yaml
spring.cloud.stream.binders:
  longhorn:
    type: kafka          # Longhorn Kafka cluster
  kafka:
    type: kafka          # Standard Kafka cluster (Comcast)
  msk:
    type: kafka          # AWS Managed Streaming (Cloud/MSK)
### Broker Selection: - **longhorn**: Legacy/standard Kafka broker - **kafka**: Comcast on-premise Kafka cluster - **msk**: AWS MSK (Managed Streaming for Kafka) for Cloud deployments --- # 2. SPRING CLOUD STREAM INTEGRATION ## 2.1 What is Spring Cloud Stream? **Spring Cloud Stream** is an abstraction layer that: - Simplifies Kafka integration in Spring applications - Provides a programming model via @EnableBinding and @StreamListener - Handles message serialization/deserialization - Supports multiple message brokers (Kafka, RabbitMQ, etc.) ## 2.2 Key Components ### 1. **Binder** - Acts as middleware between application and Kafka - Abstracts broker-specific details - Manages connections, authentication, SSL/TLS ### 2. **Channel** (Input/Output) - **Input Channel** (@Input): Consumes messages from Kafka topic - **Output Channel** (@Output): Publishes messages to Kafka topic - Defined in channel interfaces ### 3. **Stream Listener** - Method that receives messages from a channel - Automatically deserializes Kafka message to Java object - Annotated with @StreamListener ### 4. **Event Binding** - Connects channel interfaces to Spring application - Enabled via @EnableBinding(ChannelInterface.class) - Creates message flow at startup --- # 3. EVENT CHANNELS CONFIGURATION ## 3.1 Channel Interface Pattern Channel interfaces define the message bindings for a specific event type: ### Example 1: BillingEventChannels (Input Channels)
java
public interface BillingEventChannels {
    
    // Channel names (Kafka topic names)
    String FULFILLMENT_REQUEST_CHANNEL_CLOUD = "billing-fulfillment-events-cloud";
    String FULFILLMENT_REQUEST_CHANNEL_COMCAST = "billing-fulfillment-events-comcast";
    
    String PROVISIONING_REQUEST_CHANNEL_CLOUD = "billing-provisioning-events-cloud";
    String PROVISIONING_REQUEST_CHANNEL_COMCAST = "billing-provisioning-events-comcast";
    
    String WRITE_BACK_REQUEST_CHANNEL_CLOUD = "billing-writeback-events-cloud";
    String WRITE_BACK_REQUEST_CHANNEL_COMCAST = "billing-writeback-events-comcast";
    
    // Input channel definitions (listening to topics)
    @Input(FULFILLMENT_REQUEST_CHANNEL_CLOUD)
    SubscribableChannel fulfillmentRequestChannelCloud();
    
    @Input(FULFILLMENT_REQUEST_CHANNEL_COMCAST)
    SubscribableChannel fulfillmentRequestChannelComcast();
    
    @Input(PROVISIONING_REQUEST_CHANNEL_CLOUD)
    SubscribableChannel provisioningRequestChannelCloud();
    
    @Input(PROVISIONING_REQUEST_CHANNEL_COMCAST)
    SubscribableChannel provisioningRequestChannelComcast();
    
    @Input(WRITE_BACK_REQUEST_CHANNEL_CLOUD)
    SubscribableChannel writeBackRequestChannelCloud();
    
    @Input(WRITE_BACK_REQUEST_CHANNEL_COMCAST)
    SubscribableChannel writeBackRequestChannelComcast();
}
**Purpose:** - Defines 6 input channels for consuming billing events - Each channel maps to a Kafka topic - Cloud and Comcast channels keep traffic segregated --- ### Example 2: ProvisioningOrderEventChannels (Output Channels)
java
public interface ProvisioningOrderEventChannels {
    
    String WORKFLOW_PROVISIONING_REQUEST_CLOUD = "wkfl-provisioning-request-cloud";
    String WORKFLOW_PROVISIONING_REQUEST_COMCAST = "wkfl-provisioning-request-comcast";
    
    // Output channel definitions (publishing to topics)
    @Output(WORKFLOW_PROVISIONING_REQUEST_CLOUD)
    SubscribableChannel provisionOrderRequestChannelCloud();
    
    @Output(WORKFLOW_PROVISIONING_REQUEST_COMCAST)
    SubscribableChannel provisionOrderRequestChannelComcast();
}
**Purpose:** - Defines 2 output channels for publishing provisioning orders - Messages published here are sent to Kafka - Services can subscribe to these topics elsewhere --- ### Example 3: CancelOrderEventChannels (Mixed Channels)
java
public interface CancelOrderEventChannels {
    
    // Response channels (output)
    String CANCEL_ORDER_WORKFLOW_RESPONSE_CHANNEL_CLOUD = "wkfl-cancel-order-response-cloud";
    String CANCEL_ORDER_WORKFLOW_RESPONSE_CHANNEL_COMCAST = "wkfl-cancel-order-response-comcast";
    
    // Request channels (input)
    String CANCEL_ORDER_WORKFLOW_REQUEST_CHANNEL_CLOUD = "wkfl-cancel-order-request-cloud";
    String CANCEL_ORDER_WORKFLOW_REQUEST_CHANNEL_COMCAST = "wkfl-cancel-order-request-comcast";
    
    // Publish cancel responses
    @Output(CANCEL_ORDER_WORKFLOW_RESPONSE_CHANNEL_CLOUD)
    SubscribableChannel cancelOrderResponseChannelCloud();
    
    @Output(CANCEL_ORDER_WORKFLOW_RESPONSE_CHANNEL_COMCAST)
    SubscribableChannel cancelOrderResponseChannelComcast();
    
    // Listen to cancel requests
    @Input(CANCEL_ORDER_WORKFLOW_REQUEST_CHANNEL_CLOUD)
    SubscribableChannel cancelOrderRequestChannelCloud();
    
    @Input(CANCEL_ORDER_WORKFLOW_REQUEST_CHANNEL_COMCAST)
    SubscribableChannel cancelOrderRequestChannelComcast();
}
**Purpose:** - Single interface handles both input and output - Cancel order feature is bidirectional (request-response) --- ## 3.2 Channel Naming Convention | Channel Part | Description | Example | |-------------|-------------|---------| | Prefix | Feature area | billing-, wkfl- | | Event Type | Type of event | fulfillment-events, provisioning-request | | Network | Cloud or Comcast | -cloud, -comcast | **Examples:** - billing-fulfillment-events-cloud = Billing fulfillment events on Cloud network - wkfl-provisioning-request-comcast = Workflow provisioning requests on Comcast network - wkfl-cancel-order-response-cloud = Cancel order responses on Cloud network --- # 4. EVENT PRODUCERS ## 4.1 What is an Event Producer? An **Event Producer** is a service component that **publishes events to Kafka topics**. It: - Sends messages to output channels - Uses Hystrix circuit breaker for resilience - Implements fallback strategies ## 4.2 Producer Pattern ### Template for Creating a Producer:
java
@EnableBinding(ChannelInterface.class)  // 1. Bind to channel interface
public class MyEventProducer extends EventProducer {
    
    @Autowired
    private ChannelInterface channels;  // 2. Inject channel interface
    
    @Autowired
    private MbosCommonStreamProperties streamProperties;  // 3. Stream config
    
    @HystrixCommand(fallbackMethod = "fallbackMethod")  // 4. Circuit breaker
    public void publishEvent(MyEvent event) {  // 5. Publish method
        // 6. Call produceEvent() helper (from EventProducer base class)
        produceEvent(
            event,
            generateMessageKey(),
            streamProperties.isCloudEnabled(),
            channels.cloudChannel(),
            channels.comcastChannel()
        );
    }
    
    // 7. Fallback if Cloud fails
    public void fallbackMethod(MyEvent event) {
        produceEvent(event, generateMessageKey(), channels.comcastChannel());
    }
}
--- ## 4.3 Provisioning Order Producer **Location:** ProvisioningOrderRequestProducer.java
java
@EnableBinding(ProvisioningOrderEventChannels.class)
public class ProvisioningOrderRequestProducer extends EventProducer {
    
    @Autowired
    private ProvisioningOrderEventChannels provisioningOrderEventChannels;
    
    @Autowired
    private MbosCommonStreamProperties mbosCommonStreamProperties;
    
    @HystrixCommand(fallbackMethod = "publishProvisionOrderEventToComcast")
    public void publishProvisionOrderEvent(ProvisioningOrderCreateEvent event) {
        logger.info("Publishing Provision Order event, isCloudEnabled: {}", 
                   mbosCommonStreamProperties.isCloudEnabled());
        
        // Publish to both Cloud and Comcast channels
        produceEvent(
            event,
            generateMessageKey(),
            mbosCommonStreamProperties.isCloudEnabled(),
            provisioningOrderEventChannels.provisionOrderRequestChannelCloud(),
            provisioningOrderEventChannels.provisionOrderRequestChannelComcast()
        );
    }
    
    // Fallback: Only publish to Comcast if Cloud fails
    public void publishProvisionOrderEventToComcast(ProvisioningOrderCreateEvent event) {
        produceEvent(event, generateMessageKey(), 
                    provisioningOrderEventChannels.provisionOrderRequestChannelComcast());
        logger.info("Published Provision Order event to Comcast (Cloud fallback): {}", event);
    }
}
### Flow:
1. publishProvisionOrderEvent() called with order
2. @HystrixCommand attempts Cloud + Comcast publish
3. If Cloud fails → Hystrix triggers fallback
4. publishProvisionOrderEventToComcast() publishes to Comcast only
5. Order is guaranteed to be published somewhere
### Usage in Application:
java
@Service
public class BillingOrderWorkflowService {
    
    @Autowired
    private ProvisioningOrderRequestProducer provisioningProducer;
    
    public void startBillingWorkflow(WorkflowRequest request) {
        // ... workflow logic ...
        
        // Publish provisioning order to Kafka
        ProvisioningOrderCreateEvent event = new ProvisioningOrderCreateEvent();
        event.setOrderId(request.getOrderId());
        event.setAccountId(request.getAccountId());
        
        provisioningProducer.publishProvisionOrderEvent(event);
    }
}
--- ## 4.4 Cancel Order Producer **Location:** CancelOrderRequestEventProducer.java
java
@EnableBinding(CancelOrderEventChannels.class)
public class CancelOrderRequestEventProducer extends EventProducer {
    
    @Autowired
    private CancelOrderEventChannels cancelOrderEventChannels;
    
    @Autowired
    private MbosCommonStreamProperties mbosCommonStreamProperties;
    
    @HystrixCommand(fallbackMethod = "publishCancelRequestEventToComcast")
    public void publishCancelRequestEvent(CancelOrderRequest cancelRequest) {
        logger.info("Publishing Cancel request event, reason: {}, isCloudEnabled: {}", 
                   cancelRequest.getCancelReason(), 
                   mbosCommonStreamProperties.isCloudEnabled());
        
        produceEvent(
            cancelRequest,
            generateMessageKey(),
            mbosCommonStreamProperties.isCloudEnabled(),
            cancelOrderEventChannels.cancelOrderResponseChannelCloud(),
            cancelOrderEventChannels.cancelOrderResponseChannelComcast()
        );
    }
    
    public void publishCancelRequestEventToComcast(CancelOrderRequest cancelRequest) {
        produceEvent(cancelRequest, generateMessageKey(), 
                    cancelOrderEventChannels.cancelOrderResponseChannelComcast());
        logger.info("Published Cancel request event to Comcast (Cloud fallback): {}", cancelRequest);
    }
}
--- ## 4.5 Other Producers The application includes additional producers for different event types: | Producer | Output Channel | Event Type | Purpose | |----------|----------------|-----------|---------| | ProvisioningOrderRequestProducer | wkfl-provisioning-request-* | ProvisioningOrderCreateEvent | Send orders for device provisioning | | CancelOrderRequestEventProducer | wkfl-cancel-order-response-* | CancelOrderRequest | Publish cancel order requests | | CancelOrderResponseProducer | wkfl-cancel-order-response-* | CancelOrderResponse | Publish cancel order confirmations | | CancelInsuranceEventProducer | Custom channel | CancelInsuranceEvent | Cancel insurance enrollments | | ReQueueWorkflowRequestProducer | wkfl-service-request-* | WorkflowRequest | Requeue failed workflow requests | --- # 5. EVENT CONSUMERS ## 5.1 What is an Event Consumer? An **Event Consumer** is a service component that **listens to Kafka topics** and processes incoming events. It: - Implements @EnableBinding to connect to channels - Uses @StreamListener methods to handle messages - Automatically deserializes Kafka messages - Routes events to appropriate handlers --- ## 5.2 BillingEventsConsumer (Primary Consumer) **Location:** BillingEventsConsumer.java ### Purpose: Consumes billing-related events from three types of sources: 1. Fulfillment service (order delivery notifications) 2. Provisioning service (device activation status) 3. AMDOCS (billing write-back synchronization) ### Structure:
java
@EnableBinding(BillingEventChannels.class)
public class BillingEventsConsumer {
    
    @Autowired
    private BillingOrderWorkflowRequestEventHandler billingOrderWorkflowRequestEventHandler;
    
    // ============ FULFILLMENT EVENTS (Cloud) ============
    @StreamListener(BillingEventChannels.FULFILLMENT_REQUEST_CHANNEL_CLOUD)
    public void processBillingFulfillmentEvent(Map<String, Object> message) {
        logger.info("Received fulfillment event (Cloud): {}", message);
        processBillingEvents(message);
    }
    
    // ============ FULFILLMENT EVENTS (Comcast) ============
    @StreamListener(BillingEventChannels.FULFILLMENT_REQUEST_CHANNEL_COMCAST)
    public void processBillingFulfillmentEventComcast(Map<String, Object> message) {
        logger.info("Received fulfillment event (Comcast): {}", message);
        processBillingEvents(message);
    }
    
    // ============ PROVISIONING EVENTS (Cloud) ============
    @StreamListener(BillingEventChannels.PROVISIONING_REQUEST_CHANNEL_CLOUD)
    public void processBillingProvisioningEvent(Map<String, Object> message) {
        logger.info("Received provisioning event (Cloud): {}", message);
        processBillingEvents(message);
    }
    
    // ============ PROVISIONING EVENTS (Comcast) ============
    @StreamListener(BillingEventChannels.PROVISIONING_REQUEST_CHANNEL_COMCAST)
    public void processBillingProvisioningEventComcast(Map<String, Object> message) {
        logger.info("Received provisioning event (Comcast): {}", message);
        processBillingEvents(message);
    }
    
    // ============ WRITE-BACK EVENTS (Cloud) ============
    @StreamListener(BillingEventChannels.WRITE_BACK_REQUEST_CHANNEL_CLOUD)
    public void processBillingWriteBackEvent(Map<String, Object> message) {
        logger.info("Received write-back event (Cloud): {}", message);
        processBillingEvents(message);
    }
    
    // ============ WRITE-BACK EVENTS (Comcast) ============
    @StreamListener(BillingEventChannels.WRITE_BACK_REQUEST_CHANNEL_COMCAST)
    public void processBillingWriteBackEventComcast(Map<String, Object> message) {
        logger.info("Received write-back event (Comcast): {}", message);
        processBillingEvents(message);
    }
    
    // ============ CORE EVENT PROCESSING LOGIC ============
    private void processBillingEvents(Map<String, Object> message) {
        // 1. Extract event type from message
        Object eventType = message.get("eventType");
        String eventTypeStr = eventType != null ? eventType.toString() : null;
        
        logger.info("Processing billing event type: {}", eventTypeStr);
        
        // 2. Route to appropriate handler based on event type
        if (StringUtils.equals(eventTypeStr, SEND_MESSAGE.name())) {
            // Message event (activation, shipment, etc.)
            billingOrderWorkflowRequestEventHandler.sendMessage(message);
        } else if (StringUtils.equals(eventTypeStr, START_PROCESS.name())) {
            // Process start event
            billingOrderWorkflowRequestEventHandler.startBillingProcess(message);
        }
    }
}
### Key Characteristics: 1. **@EnableBinding**: Connects to BillingEventChannels interface 2. **6 @StreamListener methods**: One for each channel (Cloud + Comcast variants) 3. **Automatic deserialization**: Messages converted from JSON to Map<String, Object> 4. **Event routing**: Routes based on eventType field 5. **Cloud-aware**: Processes both network variants in same consumer --- ## 5.3 Event Handler Chain
Kafka Message arrives on topic
    ↓
@StreamListener method triggered
    ↓
Message deserialized to Map<String, Object>
    ↓
processBillingEvents(message) called
    ↓
Extract eventType from message
    ↓
Route to BillingOrderWorkflowRequestEventHandler
    ├─ If SEND_MESSAGE → sendMessage(message)
    │  └─ Raises activation/shipment/delivery message
    │
    └─ If START_PROCESS → startBillingProcess(message)
       └─ Starts new billing workflow
    ↓
Workflow execution (BPMN orchestration)
--- ## 5.4 Cancel Order Consumer **Location:** CancelOrderRequestEventListener.java
java
@EnableBinding(CancelOrderEventChannels.class)
public class CancelOrderRequestEventListener {
    
    @Autowired
    private BillingOrderWorkflowRequestEventHandler billingOrderWorkflowRequestEventHandler;
    
    @StreamListener(CancelOrderEventChannels.CANCEL_ORDER_WORKFLOW_REQUEST_CHANNEL_CLOUD)
    public void processCancelOrderEvent(Map<String, Object> message) {
        logger.info("Received cancel order request (Cloud): {}", message);
        billingOrderWorkflowRequestEventHandler.startCancelProcess(message);
    }
    
    @StreamListener(CancelOrderEventChannels.CANCEL_ORDER_WORKFLOW_REQUEST_CHANNEL_COMCAST)
    public void processCancelOrderEventComcast(Map<String, Object> message) {
        logger.info("Received cancel order request (Comcast): {}", message);
        billingOrderWorkflowRequestEventHandler.startCancelProcess(message);
    }
}
--- ## 5.5 Other Consumers | Consumer | Listens To | Handles | Purpose | |----------|-----------|---------|---------| | BillingEventsConsumer | Billing channels | Fulfillment, Provisioning, Write-back | Main event processor | | CancelOrderRequestEventListener | Cancel order channels | Cancel requests | Order cancellation | | MatrixBillerOrderEventListener | Matrix channels | Billing events | Matrix billing system | | ErrorEventHandler | Error channels | Exceptions | Error tracking | --- # 6. COMPLETE EVENT FLOW ## 6.1 Scenario: New Billing Order Creation Flow
┌──────────────────────────────────────────────────────────────────┐
│ 1. REST API RECEIVES ORDER                                       │
│                                                                  │
│  POST /api/workflow/start                                       │
│  {                                                               │
│    "orderId": "ORD-12345",                                      │
│    "accountId": "ACC-98765",                                    │
│    "channel": "CLOUD"                                           │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 2. BILLING ORDER WORKFLOW SERVICE                                │
│                                                                  │
│  @Service BillingOrderWorkflowService                           │
│  ├─ Validate order                                              │
│  ├─ Create billing order in domain                              │
│  └─ Execute BPMN workflow (billingOrchestration_v2.bpmn)       │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 3. WORKFLOW CREATES PROVISIONING ORDER                           │
│                                                                  │
│  Flowable Task: CreateProvisioningOrderTask                     │
│  ├─ Prepare ProvisioningOrderCreateEvent                        │
│  └─ Inject ProvisioningOrderRequestProducer                     │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 4. PRODUCE PROVISIONING EVENT TO KAFKA                           │
│                                                                  │
│  @HystrixCommand                                                │
│  publishProvisionOrderEvent(event)                              │
│  ├─ Try: Publish to wkfl-provisioning-request-cloud             │
│  ├─ Try: Publish to wkfl-provisioning-request-comcast           │
│  └─ On Failure: Fallback to comcast only                        │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 5. KAFKA TOPICS STORE EVENTS                                     │
│                                                                  │
│  Topic: wkfl-provisioning-request-cloud                         │
│  {                                                               │
│    "orderId": "ORD-12345",                                      │
│    "accountId": "ACC-98765",                                    │
│    "eventType": "PROVISIONING_REQUEST",                         │
│    "timestamp": "2025-02-02T10:30:00Z",                         │
│    "key": "ORD-12345-123456"                                    │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 6. FULFILLMENT/PROVISIONING SERVICE CONSUMES EVENT              │
│                                                                  │
│  External Service (not WkflBillingOrder)                        │
│  ├─ Reads from wkfl-provisioning-request-* topic                │
│  ├─ Provisions device                                           │
│  └─ Publishes billing-provisioning-events-cloud                │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 7. WKFLBILLINGORDER RECEIVES COMPLETION EVENT                    │
│                                                                  │
│  Topic: billing-provisioning-events-cloud                       │
│  {                                                               │
│    "orderId": "ORD-12345",                                      │
│    "status": "PROVISIONED",                                     │
│    "eventType": "START_PROCESS"                                 │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 8. BILLING EVENTS CONSUMER RECEIVES MESSAGE                      │
│                                                                  │
│  @EnableBinding(BillingEventChannels.class)                     │
│  @StreamListener(PROVISIONING_REQUEST_CHANNEL_CLOUD)            │
│  processBillingProvisioningEvent(message) {                     │
│    processBillingEvents(message);                               │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 9. ROUTE TO APPROPRIATE HANDLER                                  │
│                                                                  │
│  if (eventType == START_PROCESS) {                              │
│    BillingOrderWorkflowRequestEventHandler                      │
│      .startBillingProcess(message)                              │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 10. CONTINUE WORKFLOW EXECUTION                                  │
│                                                                  │
│  Complete pending workflow step                                 │
│  ├─ Update order status                                         │
│  ├─ Save to database                                            │
│  └─ Continue BPMN process                                       │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 11. ORDER FULFILLMENT COMPLETES                                  │
│                                                                  │
│  Publish to billing-fulfillment-events-*                        │
│  {                                                               │
│    "orderId": "ORD-12345",                                      │
│    "status": "FULFILLED",                                       │
│    "eventType": "SEND_MESSAGE"                                  │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 12. HANDLE MESSAGE EVENT                                         │
│                                                                  │
│  BillingEventsConsumer                                          │
│  @StreamListener(FULFILLMENT_REQUEST_CHANNEL_CLOUD)             │
│  processBillingFulfillmentEvent(message)                        │
│  └─ sendMessage(message)                                        │
│     ├─ Raise delivery notification                              │
│     └─ Update workflow                                          │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│ 13. WORKFLOW COMPLETES                                           │
│                                                                  │
│  BPMN Process finishes                                          │
│  ├─ All tasks completed                                         │
│  ├─ Update order to COMPLETED                                   │
│  └─ Persist to database                                         │
└──────────────────────────────────────────────────────────────────┘
--- ## 6.2 Event Lifecycle
ORDER CREATED (REST API)
    ↓
PUBLISH Provisioning Order Event (Kafka)
    ├─ Channel: wkfl-provisioning-request-*
    ├─ Consumers: Fulfillment/Provisioning Service
    └─ Persistence: Kafka topic stores event
    ↓
EXTERNAL SERVICE PROCESSES
    ├─ Receives provisioning request
    ├─ Activates device
    └─ Publishes completion event
    ↓
PUBLISH Provisioning Completion Event (Kafka)
    ├─ Channel: billing-provisioning-events-*
    ├─ Consumer: BillingEventsConsumer
    └─ Persistence: Kafka topic stores event
    ↓
CONSUME Event (Spring Cloud Stream)
    ├─ @StreamListener triggers
    ├─ Message deserialized
    └─ Handler processes
    ↓
CONTINUE WORKFLOW
    ├─ Update order status
    ├─ Execute next workflow task
    └─ Repeat cycle if needed
    ↓
ORDER COMPLETES
--- # 7. MULTI-TENANT CLOUD & COMCAST STRATEGY ## 7.1 Problem: Single System, Multiple Networks WkflBillingOrder serves **two different networks**: - **Cloud**: AWS infrastructure (MSK - Managed Streaming for Kafka) - **Comcast**: On-premise infrastructure (Kafka cluster) Events for different networks must not mix. ## 7.2 Solution: Channel Segregation Each event type has **separate channels for Cloud and Comcast**:
yaml
CHANNEL STRATEGY:
├─ Fulfillment Events
│  ├─ billing-fulfillment-events-cloud    (Cloud network)
│  └─ billing-fulfillment-events-comcast  (Comcast network)
│
├─ Provisioning Events
│  ├─ billing-provisioning-events-cloud    (Cloud network)
│  └─ billing-provisioning-events-comcast  (Comcast network)
│
├─ Provisioning Requests (Outbound)
│  ├─ wkfl-provisioning-request-cloud      (Cloud network)
│  └─ wkfl-provisioning-request-comcast    (Comcast network)
│
└─ Cancel Order Events
   ├─ wkfl-cancel-order-*-cloud            (Cloud network)
   └─ wkfl-cancel-order-*-comcast          (Comcast network)
## 7.3 Network Routing Logic ### Producer Side (Publishing):
java
@HystrixCommand(fallbackMethod = "publishProvisionOrderEventToComcast")
public void publishProvisionOrderEvent(ProvisioningOrderCreateEvent event) {
    
    // 1. Check if Cloud is enabled
    boolean isCloudEnabled = mbosCommonStreamProperties.isCloudEnabled();
    
    // 2. Publish to appropriate channel(s)
    produceEvent(
        event,
        generateMessageKey(),
        isCloudEnabled,  // Publish to Cloud if enabled
        provisioningOrderEventChannels.provisionOrderRequestChannelCloud(),
        provisioningOrderEventChannels.provisionOrderRequestChannelComcast()  // Always Comcast
    );
}

// Fallback if Cloud Kafka fails
public void publishProvisionOrderEventToComcast(ProvisioningOrderCreateEvent event) {
    // Publish to Comcast only
    produceEvent(event, generateMessageKey(), 
                provisioningOrderEventChannels.provisionOrderRequestChannelComcast());
}
### Consumer Side (Receiving):
java
@EnableBinding(BillingEventChannels.class)
public class BillingEventsConsumer {
    
    // Cloud network message handler
    @StreamListener(BillingEventChannels.FULFILLMENT_REQUEST_CHANNEL_CLOUD)
    public void processBillingFulfillmentEvent(Map<String, Object> message) {
        // Process Cloud events
        processBillingEvents(message);
    }
    
    // Comcast network message handler
    @StreamListener(BillingEventChannels.FULFILLMENT_REQUEST_CHANNEL_COMCAST)
    public void processBillingFulfillmentEventComcast(Map<String, Object> message) {
        // Process Comcast events
        processBillingEvents(message);
    }
}
## 7.4 Tenant Identification **Request Header Method:**
java
@RestController
@PostMapping("/api/workflow/start")
public ResponseEntity<ProcessResponse> startWorkflow(
    @RequestBody WorkflowRequest request,
    @RequestHeader(value = "X-Tenant-Id", required = false) String tenantId) {
    
    // tenantId identifies Cloud or Comcast
    // Service routes to appropriate producer channel
}
**Configuration Method:**
yaml
mbos:
  stream:
    cloud-enabled: true      # Cloud network enabled
    comcast-enabled: true    # Comcast network enabled
--- # 8. KAFKA BINDERS AND TRANSPORT ## 8.1 What are Binders? Binders are **abstraction layers** that connect Spring Cloud Stream to Kafka clusters. The application uses **three different binders** for different environments: ### Binder 1: MSK (AWS Managed Streaming for Kafka) **For:** Cloud deployments on AWS
yaml
spring.cloud.stream.binders.msk:
  type: kafka
  environment.spring.cloud.stream.kafka.binder:
    brokers: ${mbosCommonStreams.msk.binder.brokers}  # AWS MSK endpoint
    defaultBrokerPort: 9094                           # MSK port
    zk-nodes: ${mbosCommonStreams.msk.binder.zk-nodes}
    configuration:
      ssl.truststore.location: /app/WEB-INF/classes/awsTrustServices.truststore.jks
      security.protocol: SSL                          # TLS/SSL encryption
**Features:** - AWS-managed Kafka cluster - SSL/TLS security - High availability through AWS - Auto-scaling support ### Binder 2: Kafka (Standard Kafka) **For:** Comcast on-premise deployments
yaml
spring.cloud.stream.binders.kafka:
  type: kafka
  environment.spring.cloud.stream.kafka.binder:
    brokers: ${mbosCommonStreams.kafka.binder.brokers}
    zk-nodes: ${mbosCommonStreams.kafka.binder.zk-nodes}
    replicationFactor: 3
    minPartitionCount: 3
    auto-add-partitions: false
**Features:** - Standard on-premise Kafka - Replicated for durability - Partition management - Standard security ### Binder 3: Longhorn **For:** Legacy/specialized Kafka deployments
yaml
spring.cloud.stream.binders.longhorn:
  type: kafka
  environment.spring.cloud.stream.kafka.binder:
    brokers: ${mbosCommonStreams.longhorn.binder.brokers}
--- ## 8.2 Binder Configuration Properties ### Producer Configuration
yaml
bindings:
  wkfl-provisioning-request-comcast:
    producer:
      sync: true                          # Wait for broker acknowledgment
      header-mode: none                   # No header serialization
**Properties:** - sync: true - Synchronous publish (wait for confirmation) - header-mode: none - Don't send message headers ### Consumer Configuration
yaml
bindings:
  billing-fulfillment-events-comcast:
    consumer:
      configuration:
        max.poll.records: 10              # Batch size per poll
        max.poll.interval.ms: 300000      # Max time between polls (5 min)
      enableDlq: false                    # Don't enable dead letter queue
**Properties:** - max.poll.records - Number of messages per batch - max.poll.interval.ms - Maximum time to process batch - enableDlq - Dead letter queue for failed messages --- ## 8.3 Replication and Durability
yaml
replicationFactor: 3                      # Each message replicated to 3 brokers
minPartitionCount: 3                      # Minimum 3 partitions
auto-add-partitions: false                # Manual partition management
**How it works:**
Message Published:
    ↓
Written to Broker 1 (Leader)
    ↓
Replicated to Broker 2 (Replica)
    ↓
Replicated to Broker 3 (Replica)
    ↓
ALL ACK received → Producer confirmed
**Result:** If any single broker fails, message is still available on others. --- # 9. ERROR HANDLING & RESILIENCE ## 9.1 Circuit Breaker Pattern The application uses **Hystrix circuit breaker** to handle Kafka failures gracefully. ### Normal Operation:
publishProvisionOrderEvent(event)
    ├─ @HystrixCommand attempts primary method
    ├─ Try publish to Cloud Kafka
    ├─ Try publish to Comcast Kafka
    └─ Success → return
### Failure Scenario:
publishProvisionOrderEvent(event)  // Cloud Kafka down
    ├─ @HystrixCommand attempts primary method
    ├─ Try publish to Cloud Kafka
    ├─ FAILS after timeout (10 seconds)
    └─ @HystrixCommand triggers fallback
        ├─ publishProvisionOrderEventToComcast(event) called
        ├─ Try publish to Comcast Kafka only
        └─ Success → order still published
### Configuration:
yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000  # 10 second timeout
--- ## 9.2 Dead Letter Queues (DLQ) For messages that fail multiple retries:
yaml
bindings:
  billing-fulfillment-events-comcast:
    consumer:
      enableDlq: false                    # DLQ disabled (retry instead)
- enableDlq: false - Messages retry forever instead of going to DLQ - Alternative: Enable DLQ to separate poison messages --- ## 9.3 Message Serialization Error Handling If message cannot be deserialized to Map<String, Object>:
java
@StreamListener(BillingEventChannels.FULFILLMENT_REQUEST_CHANNEL_CLOUD)
public void processBillingFulfillmentEvent(Map<String, Object> message) {
    try {
        logger.info("Received event: {}", message);
        processBillingEvents(message);
    } catch (Exception e) {
        logger.error("Failed to process message: {}", message, e);
        // Message remains in topic for replay
    }
}
**On error:** 1. Exception logged 2. Message remains in Kafka topic 3. Can be replayed after fix deployed 4. No automatic retry within consumer --- ## 9.4 Long Processing Time Handling If message processing takes longer than max.poll.interval.ms:
yaml
bindings:
  billing-fulfillment-events-comcast:
    consumer:
      configuration:
        max.poll.interval.ms: 300000      # 5 minutes
**Scenarios:** - **Normal processing** (< 5 min) → Message acknowledged - **Long processing** (> 5 min) → Kafka thinks consumer crashed - Message redelivered to consumer group - Another consumer might process it - Duplicate processing possible **Solution for long tasks:** - Use Flowable async tasks instead of stream listeners - Move heavy processing to separate service - Increase max.poll.interval.ms value --- # 10. CONFIGURATION & PROPERTIES ## 10.1 application.yml Configuration ### Spring Cloud Stream Main Config:
yaml
spring:
  cloud:
    stream:
      binders:              # Define available Kafka clusters
        msk:               # AWS MSK for Cloud
        kafka:             # Standard Kafka for Comcast
        longhorn:          # Legacy Kafka
      
      # Bindings: map channels to topics
      bindings:
        # Inbound channels (listening)
        billing-fulfillment-events-cloud:
          content-type: application/json
          consumer:
            concurrency: 1  # Single threaded consumer
            
        billing-provisioning-events-cloud:
          content-type: application/json
          
        # Outbound channels (publishing)
        wkfl-provisioning-request-cloud:
          content-type: application/json
          producer:
            sync: true      # Wait for confirmation
--- ## 10.2 Environment-Specific Properties
yaml
mbosCommonStreams:
  
  # MSK (Cloud) Configuration
  msk:
    binder:
      brokers: kafka-msk-prod.us-west-2.amazonaws.com:9094
      zk-nodes: zookeeper-msk.us-west-2.amazonaws.com:2181
      replication-factor: 3
      min-partition-count: 3
  
  # Standard Kafka (Comcast) Configuration
  kafka:
    binder:
      brokers: kafka-comcast.internal:9092
      zk-nodes: zookeeper-comcast.internal:2181
      replication-factor: 3
      min-partition-count: 3
    
    bindings:
      consumer:
        configuration:
          max-poll-records: 10
          max-poll-interval-ms: 300000
--- ## 10.3 Stream Feature Properties
yaml
mbos:
  stream:
    cloud-enabled: true      # Publish to Cloud Kafka topics
    comcast-enabled: true    # Publish to Comcast Kafka topics
**Usage in Code:**
java
@Autowired
private MbosCommonStreamProperties streamProperties;

public void publishEvent(Event event) {
    produceEvent(
        event,
        generateMessageKey(),
        streamProperties.isCloudEnabled(),  // Check if Cloud enabled
        channelCloud(),                      // Cloud channel
        channelComcast()                     // Comcast channel (fallback)
    );
}
--- ## 10.4 Consumer Concurrency
yaml
bindings:
  billing-fulfillment-events-cloud:
    consumer:
      concurrency: 1        # Single-threaded consumer
**Effect:** - concurrency: 1 → Single listener processes messages sequentially - concurrency: 3 → Three parallel listeners (higher throughput, less ordering guarantee) **For orders:** Keep concurrency: 1 to preserve processing order --- # 11. KAFKA MESSAGING PATTERNS USED ## Pattern 1: Request-Response Used for cancel order operations:
WkflBillingOrder
    ├─ PUBLISH: wkfl-cancel-order-request-*
    │  (Cancel order request)
    │
    └─ CONSUME: wkfl-cancel-order-response-*
       (Cancel order response from external service)
## Pattern 2: Event Broadcast Fulfillment service publishes completion, multiple subscribers listen:
FulfillmentService
    ├─ PUBLISH: billing-fulfillment-events-cloud
    │
    └─ CONSUMED BY:
       ├─ WkflBillingOrder
       ├─ Analytics Service
       └─ Notification Service
## Pattern 3: Workflow Trigger Events trigger BPMN workflow continuation:
ExternalService
    ├─ PUBLISH: billing-provisioning-events-cloud
    │
    └─ WkflBillingOrder
       ├─ CONSUME
       ├─ startBillingProcess(message)
       └─ Resume workflow from saved state
--- # SUMMARY ## Kafka in WkflBillingOrder: 1. **Event Backbone**: Decouples billing order system from fulfillment, provisioning, and billing 2. **Async Communication**: Services communicate through events, not API calls 3. **Multi-Tenant**: Separate Cloud and Comcast channels for network segregation 4. **Resilient**: Circuit breakers ensure graceful degradation on failures 5. **Durable**: Messages persisted for replay and debugging 6. **Scalable**: Can handle high-volume event streams with Kafka's distributed architecture ## Key Components: - **Channels**: Define message topics (input/output) - **Producers**: Publish events to Kafka - **Consumers**: Listen and process events - **Spring Cloud Stream**: Abstracts Kafka complexity - **Binders**: Connect to specific Kafka clusters - **Hystrix**: Provides circuit breaker protection ## Main Use Cases: 1. **Order Provisioning**: Publish orders for device activation 2. **Order Fulfillment**: Receive delivery notifications 3. **Order Cancellation**: Handle cancel requests 4. **AMDOCS Sync**: Synchronize order status with billing system The Kafka architecture enables WkflBillingOrder to operate as a true microservice, communicating asynchronously with multiple dependent systems while maintaining reliability and fault tolerance.
