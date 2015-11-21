Binding-Aware Components
=========================

In general, core binding aware components are variations of the binding-independent components, implementing similar contracts, but not using the binding-independent representation of data. Data used by binding-aware components is represented by generated Java proxies and TOs to facilitate static typing and to ease application and plugin development.
核心binding aware组件是bi组件的变体，实现类似接口但不适用bi的数据表达方式

The following components implements similar contracts to binding-independent components:
组件实现了类似bi组件接口：

- Binding Aware Broker – the functionality (from Providers’ and Consumers’ perspective) is similar to the Binding-Independent Broker with the exception, that the generated proxies and TOs are used to represent data. 类似bi broker，此外用代理和TO来表示数据
- Binding Aware Consumer
- Binding Aware Provider

The BI Data Repository does not have its binding-aware counter-part, but all other data processing contracts do.
bi data repository不存与之对应binding-aware组件

#Binding Aware Broker

The Binding Aware Broker is responsible for translating operations and calls between binding-aware components and the Binding Independent Broker.
负责翻译binding-aware和bi broker之间调用

*Provided Functionality*

Provider and Consumer Registration 提供provider与consumer注册
Providers and Consumers need to register in order to use the Binding-Independent SAL layer and to expose their respective functionality via the SAL layer.
provider与consumer注册到broker上，以便于使用BI SAL并通过BI SAL暴露功能

Consumer Registration
During registration the Broker gets the functionality from the Consumer. It uses the getFunctionality method from the Consumer contract, to register that functionality into the system

A Consumer is required to use the returned session for all communication with the Broker or any Broker service. The session is injected into the Consumer by invoking the method injectConsumerSession of the Consumer contract.

Provider Registration
During registration, the Broker gets the functionality from the Provider, using the method getFunctionality from Provider contract, to register that functionality into the system. A Provider is required to use the returned session for all communication with broker or any Broker service. The session is injected into the Provider by invoking the method injectProviderSession of the Provider contract.

Translation Proxy Generation
The Binding-Aware Broker is responsible for generation of Java proxy classes that implement generated Java-specific binding APIs TOs, and mappers, that translate these data and calls to binding-independent formats.
binding-aware broker负责产生java proxy类，这些类用于翻译数据及以bi格式调用

The generated proxies also translate from the binding-independent format to generated Java-specific binding formats.

The Binding-Aware Broker does not generate proxies and mappers directly, but uses the Binding Generator.
Binding-Aware Broker不直接产生代理与映射，而是使用binding generator

See Section Binding Generator for more information.

Call Translation
The Binding-Aware Broker is responsible for translation between binding-aware calls (generated Java language binding objects and functions) to the binding independent format. The binding independent format of calls is then used to communicate with the Binding-Independent Broker and with binding-independent Providers and Consumers such as the BI Data Repository. The Binding-Aware Broker also translates between binding-independent data and calls routed from the Binding-Independent Broker to model-specific Java structures. The call translation is done by generated mappers.

See Section Binding Generator for more information.

Binding Generator
The Binding Generator is an infrastructure component responsible for runtime generation of functionality and implementation of model bindings.
binding generator负责实时产生功能和实现model binding

Provided Functionality

Generation of Proxies
The Binding Generator is responsible for generation of various types of proxies, which are defined in the Binding Model, and allows access to the Providers’ functionality, regardless the type of Provider.
binding generator负责生成各种代理，访问provider功能

Proxies provide a simple programmatic access to the Binding-Aware Broker functionality (e.g. RPC calls) and wrap the translation between data in a statically typed format to data in the binding-independent DOM format.
代理提供简单可编程访问ba broker功能并封装数据

Generation of Data Transfer Object Builders
Transfer objects are not generated directly, but the Binding Generator creates an implementation of builders which are to be exposed to Consumers to expose the data.
TO并不直接产生，但是binding generator创建builder的实现给consumer

Builder = getBuilder(TransferObjectClass)
Where:

Builder: An implementation of generated Java interface for building immutable DTOs.
TransferObjectClass: An Java Class object representing the DTO for which the builder should be generated.
Generation of TO Mappers
Generated mappers are not designed to be directly used to the Consumers, but to be used by Binding-Aware Broker

Mapper = getMapper(TransferObjectClass)
Where:

Mapper: An implementation of the DTOMapper Contract, specific for the supplied TransferObjectClass
TransferObjectClass: A Java Class object representing the interface describing the data transfer object.
Mapper = getMapper(SchemaPath)
Where:

Mapper: An implementation of the DTOMapper Contract, specific for the supplied TransferObjectClass
SchemaPath: An path in YANG schema representing the schema definition for the data.

DTOMapper contract
Implementations of DTOMapper contract are responsible for two-way mapping / translation of data in Binding-aware format (instances of generated DTOs) and binding-indepenendent Data DOM format.

TO = createDTO(DOMNode)
Where:

TO: DTO representing the data
DOMNode: Data DOM representation of data
DOMNode = createDOMNode(TO)
Where:

TO: DTO representing the data
DOMNode: Data DOM representation of data
Dependencies
The Binding Generator is dependent on:

Model Schema
Binding-independent Data Form
Binding Model
Model Schema Repository

Generation Workflow

The binding generation workflow is variation of the Netconf/YANG client workflow.

A Binding specification is generated during the development phase of consumers and providers, the format is set for generated sources of Java interfaces and TOs
The translation specification is dependent on the implementation of the Translation Compiler. The possibilities for the translation specification are:
Generated java source code – if static compilation is required
Annotations on interfaces in the binding specification
Packaged source YANG models
The Runtime Binding Generator is responsible for runtime generation of proxies, TOs, TO builders and mappers.
The generation workflow is as follows:

At development time:
Developer adds a YANG model to the project (Consumer or Provider)
Developer invokes the Java Binding Generator
The Java Binding Generator generates the Binding Specification (Java interfaces of TOs, RPC services)
Developer writes code using the generated Binding Specification and the Binding-Aware Broker APIs.
Developer compiles the project
YANG models are also bundled into compiled library
At runtime:
Project is deployed to the Controller
Project code requests RPC services and/or TO Builders via Binding-Aware Broker APIs
The Binding-Aware Broker gets the instances of TO Builders and proxies from the Binding Generator and returns these instances to the project code
Project code uses generated proxies.
Proxies invokes Binding-Aware Broker functionality
Binding-Aware Broker invokes TO translation (by using Generated Mapper) if the target is Binding-independent Provider. Otherwise Binding-Aware Broker passes TOs directly to the Provider.
