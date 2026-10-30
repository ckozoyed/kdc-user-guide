# Chapter 2 — KDC Architecture and Core Concepts

## 2.1 Architecture Overview

Kristen Data Comms (KDC) provides an object-oriented communication architecture that enables distributed applications to exchange software Agents across networks.

KDC separates application behavior from the underlying communication infrastructure. Developers create Agents using objects native to their programming language, while KDC manages serialization, transmission, reconstruction, and dispatch to registered application APIs.

A central feature of KDC is the relationship between an Agent and the application services it visits.

**An Agent interacts with an application's services through a registered API supplied by KDC when its selected visitor method is invoked.**

The sending application identifies the visitor method and API class associated with the requested operation. The receiving application supplies the registered service implementation.

This separation allows Agents to interact with different application services without requiring knowledge of their concrete implementations.

The primary architectural components are:

- **Agent** — Represents an operation and carries its associated data.
- **Reply** — Communicates the result of an operation and may carry additional data.
- **Comms Entity** — A communication endpoint, such as a KDC Client or Server.
- **Client** — Sends Agents and participates in receiving Agents and Replies.
- **Server** — Accepts incoming communication and coordinates Agent processing.
- **Application API** — Defines the operations through which an Agent interacts with local application services.
- **Serializer** — Converts Agents and Replies between native objects and transmitted representations.
- **Transport** — Carries serialized Agents and Replies across the network.
- **Event Handler** — Receives callbacks about communication activity and lifecycle events.

Together, these components provide a consistent Agent-based programming model across supported communication transports.

## 2.2 The KDC Communication Architecture

A typical KDC application consists of communication endpoints that exchange serialized Agents.

The sending application creates an Agent and identifies the visitor method and application API class to be used at the destination.

KDC serializes the Agent and transmits it using the configured network transport.

At the receiving endpoint, KDC reconstructs the Agent using a compatible class already available to the application.

When the Agent is eligible for dispatch, KDC uses the Agent's selected visitor method and API class to identify the appropriate registered service API.

KDC invokes the visitor method, passing the registered API as a parameter. The Agent then interacts with the local application service through that API.

A simplified interaction is illustrated below.

```text
              SENDING APPLICATION
          +---------------------------+
          |     Application Logic     |
          |             |             |
          |             v             |
          |           Agent           |
          |             |             |
          |   Set visitor method      |
          |   and API class           |
          |             |             |
          |             v             |
          |         KDC Client        |
          |             |             |
          |          Serializer       |
          +-------------|-------------+
                        |
                        | Serialized Agent
                        v
                 NETWORK TRANSPORT
                        |
                        v
          +---------------------------+
          |  RECEIVING COMMS ENTITY   |
          |                           |
          |        Deserializer       |
          |             |             |
          |             v             |
          |     Reconstructed Agent   |
          |             |             |
          |             v             |
          |        KDC Dispatch       |
          |             |             |
          |    Resolve registered API |
          |             |             |
          |             v             |
          |    Selected visit(api)    |
          |             |             |
          |             v             |
          |    Application Service    |
          +---------------------------+
```

The diagram illustrates the fundamental KDC dispatch model.

Although a server commonly receives an Agent from a client, **both KDC Clients and Servers can receive Agents and interact with registered application APIs**, depending on the transport and implementation.

Agents may also be delivered to applications through event-handler callbacks rather than visitor-method dispatch.

For a working Java example and an accompanying architecture diagram, see the [KDC Java Example](https://kristensoft.com/java-example/).

## 2.3 Agents

The **Agent** is the central abstraction in KDC.

An Agent is an application-defined object that represents an operation and contains the data associated with that operation.

Agents are created using classes native to the application's programming language.

An Agent can contain:

- A unique identifier
- An application API class name
- A visitor method name
- Application-defined data
- Synchronous or asynchronous communication information
- Serialization and communication metadata

KDC transmits the serialized state of an Agent rather than its executable class definition.

The receiving endpoint reconstructs the Agent using a compatible class already available to that application.

### Agent Identity and Data

Each Agent has a unique identifier that helps associate communication activity and Replies with the originating operation.

Agents can also carry application-defined data, allowing information to travel with the operation it represents.

This data may include input parameters, intermediate results, or information accumulated during interactions with multiple services.

### Sender-Selected Visitor Methods

An Agent may define one or more visitor methods, each designed to interact with a particular application API.

Before transmitting an Agent, the sending application selects the visitor method and API class to be used at the receiving endpoint.

These selections are stored in the Agent using:

- `visitorMethodName` — Identifies the visitor method to invoke.
- `apiClassName` — Identifies the application API required by that method.

KDC uses these values during dispatch.

The receiving application must have a compatible Agent class and the appropriate API registered.

**The sender selects the requested operation, while the receiver controls which application services are available.**

This enables the same Agent class to participate in different operations at different communication endpoints.

A complete example of creating an Agent, selecting its visitor method, and registering an application API will be presented in Chapter 4.

## 2.4 Application APIs and Dependency Injection

One of KDC's defining architectural features is the interaction between Agents and registered application APIs.

An application API defines an interface through which an Agent interacts with the receiving application's services.

The receiving application provides an implementation of that interface and registers it through the parameters used to configure a KDC Comms Entity.

Because Clients and Servers are both Comms Entities, this registration model can support Agent dispatch at either type of endpoint.

### How API Injection Works

When KDC receives a dispatchable Agent:

1. KDC reconstructs the Agent from its serialized representation.
2. KDC reads the sender-selected `visitorMethodName` and `apiClassName`.
3. KDC identifies the appropriate registered API implementation.
4. KDC invokes the selected visitor method, passing the registered API as a parameter.
5. The Agent uses the supplied interface to interact with the receiving application's local services.

Conceptually:

```text
             RECEIVING COMMS ENTITY
          +----------------------------+
          |                            |
          |   Registered Service API   |
          |             |              |
          |             v              |
          |        KDC Dispatch        |
          |             |              |
          |       Supplies API         |
          |             |              |
          |             v              |
          |    Selected visit(api)     |
          |             |              |
          |             v              |
          |    Local Service Methods   |
          |                            |
          +----------------------------+
```

**Dependency injection occurs when KDC passes the registered API implementation to the Agent's selected visitor method.**

The Agent does not need to instantiate the service or discover its concrete implementation.

The receiving application supplies the service implementation, and the Agent interacts with it through the registered API.

This separation allows applications to provide different implementations while supporting compatible Agents.

### Dispatch Eligibility

Not every received Agent is necessarily dispatchable.

Dispatch requires a compatible visitor method and an appropriate registered application API.

An Agent may also be received through an event-handler callback without invoking a registered application service.

## 2.5 Multiple Visitor Methods and Multi-Service Agents

An Agent may define multiple visitor methods, each designed to interact with a different application API.

This enables an Agent to participate in a sequence of distributed operations.

For example, an Agent might visit an order service, an inventory service, and a payment service.

Before each transmission, the sending application selects the visitor method and API class appropriate for the next destination.

At each endpoint, KDC invokes the selected visitor method and supplies the locally registered API.

```text
                   ORDER AGENT
                       |
                       v
          +---------------------------+
          |       ORDER SERVICE       |
          |                           |
          |   visitOrder(orderApi)    |
          |                           |
          |   Validate order          |
          |   Update Agent data       |
          +-------------|-------------+
                        |
                        | Select next visitor
                        | Transmit Agent
                        v
          +---------------------------+
          |     INVENTORY SERVICE     |
          |                           |
          | visitInventory(invApi)    |
          |                           |
          | Check availability        |
          | Update Agent data         |
          +-------------|-------------+
                        |
                        | Select next visitor
                        | Transmit Agent
                        v
          +---------------------------+
          |      PAYMENT SERVICE      |
          |                           |
          |  visitPayment(payApi)     |
          |                           |
          |  Process payment          |
          |  Update Agent data        |
          +---------------------------+
```

The same logical Agent can carry application-defined data between services, while each service supplies its own implementation through its registered API.

The application determines the Agent's next destination and selects the appropriate visitor method for that interaction.

### Application to Microservices

A microservices architecture divides an application into relatively independent services, each responsible for a particular business capability.

These services communicate to coordinate operations and complete business workflows.

KDC's multi-visitor Agent model offers a useful approach to certain microservice interactions.

An Agent can carry the state of a distributed operation while interacting with different services through their respective APIs.

Potential applications include:

- Order processing across multiple business services
- Distributed workflow coordination
- Data collection and enrichment
- Multi-stage validation and processing
- Distributed control operations

KDC provides the Agent communication and dispatch mechanisms. The application remains responsible for determining service destinations, coordinating workflow steps, and handling failures or transactions spanning multiple services.

A complete multi-service Agent example will be presented in a later chapter.

### Role-Dependent Agent Behavior

An Agent may interact with different application components depending on the endpoint it visits.

**Server-side interaction:** An Agent's visitor method can use a registered API to interact with business services, perform calculations, access databases, or update application state.

**Client-side interaction:** An Agent's visitor method can use a registered API to interact with client application objects, update a graphical user interface, or process information received from the server.

In both cases, KDC invokes the sender-selected visitor method and passes the locally registered API as a parameter.

This allows an Agent to participate in distributed operations across servers and clients while remaining independent of their concrete service implementations.

For example, a server might send an asynchronous Agent to a client whose selected visitor method updates the client's user interface.

This illustrates how the same Agent programming model can support both business-service operations and client-side application interaction.

## 2.6 Replies

A **Reply** represents a response associated with an Agent operation.

Replies provide a consistent mechanism for communicating the outcome of an operation.

A Reply may contain:

- Success or failure information
- A descriptive message
- Response codes
- Application-defined data
- An indication of whether it is the final Reply

The `Reply` class can also be subclassed to create application-specific response types.

For example, an application might define an `OrderReply` containing additional information about an order-processing operation.

### Multiple Replies

Some operations may produce more than one Reply.

A long-running operation could provide intermediate results or progress updates followed by a final Reply.

The `lastReply` property identifies whether a Reply is the final response in a sequence.

Multiple-Reply behavior depends on the communication implementation and will be explored with asynchronous communication examples in a later chapter.

## 2.7 Comms Entities, Clients, and Servers

KDC provides communication endpoints through its Comms Entity architecture.

A Comms Entity provides common communication capabilities and configuration behavior used by Clients and Servers.

Application APIs are registered through Comms Entity parameters, allowing receiving endpoints to interact with Agents through the appropriate local services.

### Clients

A KDC Client provides the application-facing interface for communicating with remote endpoints.

Depending on the implementation, a Client may:

- Establish and close network connections
- Send synchronous or asynchronous Agents
- Receive Replies
- Receive incoming Agents
- Invoke registered application APIs for incoming Agents
- Notify application event handlers
- Manage communication timeouts and connection behavior

### Servers

A KDC Server accepts incoming network communication and coordinates Agent processing.

For connection-oriented transports such as TCP, a Server accepts connections and creates KDC Client communication objects associated with the accepted sockets.

These objects allow the server side of a connection to communicate with the connected endpoint.

Depending on the transport, a Server may:

- Listen for incoming connections or datagrams
- Accept and manage connected clients
- Receive serialized Agents
- Reconstruct and dispatch Agents
- Send Replies
- Send Agents to connected clients
- Invoke registered application APIs
- Notify event handlers about communication activity

Connectionless transports, such as UDP and Multicast, use different communication mechanisms and do not require the same accepted-socket model.

### Parameters

KDC Clients and Servers are configured using parameter objects.

Parameters define communication behavior and can include:

- Network addresses and ports
- Transport-specific settings
- Timeouts and buffer sizes
- Registered application APIs
- Event handlers
- Serialization configuration
- Security and data-processing options

Parameter builders help applications construct these configurations.

Specific parameter classes and examples will be introduced in later chapters.

## 2.8 Event Handlers and Callbacks

KDC provides event handlers that notify applications about communication activity.

Event handlers receive callbacks when supported communication events occur.

Depending on the implementation, these may include:

- Connection establishment
- Connection closure
- Agent transmission
- Agent reception
- Reply reception
- Communication errors

Event handlers serve a different purpose from registered application APIs.

**Application APIs** provide operations that Agents invoke through their visitor methods.

**Event handlers** provide callbacks that allow the application to observe and respond to communication events.

### Receiving Agents Through Event Handlers

Applications can receive Agents through event-handler callbacks without relying on visitor-method dispatch.

This provides an alternative approach when an application wants to process an incoming Agent directly.

The two receiving mechanisms are:

| Mechanism | Behavior |
|---|---|
| Registered Application API | KDC invokes the Agent's selected visitor method with the registered API |
| Event Handler | KDC delivers the received Agent through an application callback |

The appropriate mechanism depends on the application's requirements and the capabilities of the selected transport implementation.

## 2.9 Synchronous and Asynchronous Communication

KDC supports synchronous and asynchronous Agent communication patterns.

### Synchronous Agents

A synchronous Agent is sent when the calling application needs to wait for a Reply before continuing.

A typical interaction is:

1. The client creates an Agent.
2. The client sends the Agent synchronously.
3. The receiving endpoint processes the Agent.
4. A Reply is returned.
5. The sending operation completes with the Reply or an error or timeout.

```text
CLIENT                         SERVICE
  |                               |
  |------ Synchronous Agent ----->|
  |                               |
  |        Process Agent          |
  |                               |
  |<----------- Reply ------------|
  |                               |
  v
Continue Processing
```

### Asynchronous Agents

An asynchronous Agent is sent without requiring the sending operation to wait synchronously for remote processing to complete.

This allows the application to continue other work while the Agent is being processed.

```text
CLIENT                         SERVICE
  |                               |
  |------ Asynchronous Agent ---->|
  |                               |
  v                         Process Agent
Continue                           |
Processing                         |
  |                               |
  |<------- Later Response -------|  (if applicable)
```

Depending on the language and transport implementation, asynchronous communication may be handled through event callbacks, registered application APIs, or methods for retrieving received Replies.

KDC-Java includes methods such as `takeReply()` and `hasReply()` for supported Reply-retrieval patterns.

The underlying buffering and scheduling mechanisms are implementation details and are not necessary for understanding the Agent programming model.

Synchronous communication, asynchronous communication, and multiple Replies will be explored through working examples in a later chapter.

## 2.10 Serialization and Transport Independence

KDC separates the Agent programming model from the underlying network transport.

An Agent represents an application operation independently of whether it is transmitted over TCP, UDP, Multicast, HTTP, or WebSocket.

**The Agent's programming model remains consistent across transports.**

The transport determines how serialized data is delivered, while the Agent defines the application operation and associated data.

This allows developers to work with the same general Agent concepts across different communication environments.

### Supported Language Implementations

| Implementation | Supported Transports | Serialization |
|---|---|---|
| KDC-Java | TCP, UDP, Multicast, HTTP, WebSocket | Multiple serialization technologies |
| KDC-JavaScript | WebSocket client | JSON |
| KDC-Python | TCP client | JSON |

JavaScript and Python currently provide client implementations, while KDC-Java provides both client and server capabilities.

### Serialization Technologies

KDC-Java supports several approaches to object serialization.

| Technology | Description |
|---|---|
| **JSON (Jackson)** | Uses the Jackson library to serialize and deserialize Java objects |
| **Protocol Buffers** | Supports user-defined Protocol Buffer messages and schemas |
| **Kristen Objects (KO)** | Serializes ordinary Java objects (POJOs) into a Protocol Buffer representation for communication |
| **Java Serialization** | Uses Java's built-in object serialization mechanism |

Kristen Objects provides a way to communicate Java objects using a Protocol Buffer representation without requiring developers to define a separate application-specific Protocol Buffer schema for every POJO.

The availability of a serialization technology for a particular Agent communication path depends on the implementation and configuration.

For cross-language communication, endpoints must use compatible wire formats and Agent definitions.

A compatible Agent can retain its application-level meaning across different transports, even though transport framing and communication behavior may differ.

No serialization technology should be assumed to offer superior performance without benchmarks representative of the application's workload.

Java's built-in deserialization also requires particular care when handling untrusted data.

## 2.11 Security and Data Processing

KDC provides configurable mechanisms for processing and protecting serialized Agent data.

Depending on the implementation, these include:

- **Class whitelisting** — Restricts which classes are permitted during deserialization.
- **AES-GCM encryption** — Provides authenticated encryption of serialized data.
- **Compression** — Reduces serialized data size where beneficial.
- **CRC checking** — Detects certain forms of accidental data corruption.

Class whitelisting helps prevent unrestricted object reconstruction but does not replace authentication, authorization, or application-level data validation.

CRC checking is not a substitute for cryptographic integrity protection.

Applications must ensure compatible communication settings and appropriately manage security-sensitive configuration, including encryption keys.

Detailed configuration examples will be presented in later chapters.

## 2.12 Design Patterns and Architectural Principles

KDC incorporates several established software design patterns and architectural principles.

Understanding these patterns helps explain how KDC separates application behavior from communication infrastructure.

| Pattern | Application in KDC |
|---|---|
| **Command** | An Agent encapsulates an operation and its associated data for remote communication and processing |
| **Visitor** | Agent visitor methods provide a structured way to interact with application services |
| **Dependency Injection** | KDC passes the registered API implementation into the selected visitor method |
| **Strategy** | Different transport and serialization implementations provide alternative communication and encoding approaches |
| **Observer** | Event handlers receive callbacks about communication activity |
| **Builder** | Parameter builders simplify the construction of Client and Server configurations |

### Command, Visitor, and Dependency Injection

Three concepts are particularly important to the KDC programming model.

**Command:** An Agent encapsulates an operation and its associated data. It can be transmitted to another endpoint for processing.

**Visitor:** The Agent defines visitor methods through which it interacts with application services. The sender selects the visitor method appropriate for the destination.

**Dependency Injection:** KDC invokes the selected visitor method and supplies the receiving endpoint's registered API as a parameter.

Conceptually:

```text
       COMMAND
         |
         v
        Agent
         |
         | Sender selects visitor method
         v
       VISITOR
         |
         | KDC invokes selected method
         v
   DEPENDENCY INJECTION
         |
         | Registered API supplied
         v
   APPLICATION SERVICE
```

Together, these mechanisms allow an Agent to carry operation state while relying on locally supplied service implementations.

KDC's visitor mechanism is adapted to its Agent communication model and does not necessarily follow every detail of the traditional Visitor design pattern.

### Separation of Concerns

KDC also separates several important application responsibilities:

- **Application data** is carried by Agents and their associated objects.
- **Operation coordination** is expressed through Agent visitor methods.
- **Service implementation** remains within the receiving application.
- **Network communication** is managed by KDC.

This separation allows Agents to interact with different application components depending on the endpoint they visit.

For example, an Agent might invoke server-side business services at one endpoint and interact with a client-side GUI at another.

This resembles aspects of Model-View-Controller (MVC), particularly the separation of data and operation coordination. However, KDC is not itself an MVC framework and does not prescribe a presentation-oriented View component.

## 2.13 Component Responsibilities

The following table summarizes the primary responsibilities of KDC's architectural components.

| Component | Responsibility |
|---|---|
| Agent | Represents an operation and carries application data |
| Visitor Method | Uses an injected API to interact with a local service |
| Reply | Communicates an operation's outcome and response data |
| Comms Entity | Provides common communication and configuration capabilities |
| Client | Sends Agents and participates in receiving Agents and Replies |
| Server | Accepts communication and coordinates processing |
| Application API | Exposes local application services to Agents |
| Parameters | Configure endpoints, registered APIs, and communication behavior |
| Event Handler | Receives communication event callbacks |
| Serializer | Converts Agents and Replies to and from transmitted representations |
| Transport | Delivers serialized data between endpoints |

The most important relationship to understand is:

**Agent → Selected Visitor Method → Registered Application API → Local Service**

KDC provides the communication infrastructure and invokes the sender-selected visitor method with the appropriate registered API.

The Agent uses that API to interact with the receiving application's services.

This separation is fundamental to the KDC programming model.

## 2.14 Next Steps

With the core architecture established, the next chapter introduces the installation and configuration process for KDC.

**Chapter 3 — Getting Started** will cover development prerequisites, available language implementations, project setup, and preparation for building a working KDC application.

Subsequent chapters will introduce practical examples involving:

- Creating Agents and subclassing Replies
- Using `setData()` and `getData()` with enum-based keys
- Selecting visitor methods and application API classes
- Configuring Clients and Servers through parameter builders
- Registering application APIs
- Receiving Agents through event handlers
- Sending synchronous and asynchronous Agents
- Receiving Replies and handling multiple responses
- Using multiple visitor methods across distributed services

These examples will build on the architectural concepts introduced here, progressing from a simple client/server interaction to more advanced distributed application patterns.