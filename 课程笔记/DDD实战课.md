# 课程笔记：DDD实战课

我认为微服务拆分的关键是，越是相关的部分交互就越频繁，所以拆分原则就是尽可能地减少节点之间的网络通信。

## 01

DDD 核心思想是通过领域驱动设计方法定义领域模型，从而确定业务和应用边界，保证业务模型与代码模型的一致性。

DDD 不是架构，而是一种架构设计方法论，它通过边界划分将复杂业务领域简单化，帮我们设计出清晰的领域和应用边界，可以很容易地实现架构演进。DDD 包括战略设计和战术设计两部分。

战略设计主要从业务视角出发，建立业务领域模型，划分领域边界，建立通用语言的限界上下文，限界上下文可以作为微服务设计的参考边界。战术设计则从技术视角出发，侧重于领域模型的技术实现，完成软件开发和落地，包括：聚合根、实体、值对象、领域服务、应用服务和资源库等代码逻辑的设计和实现。

采用用例分析、场景分析和用户旅程分析，尽可能全面不遗漏地分解业务领域，并梳理领域对象之间的关系。

DDD 主要关注：从业务领域视角划分领域边界，构建通用语言进行高效沟通，通过业务抽象，建立领域模型，维持业务和代码的逻辑一致性。

DDD 的拆分考虑到领域专家的范围，能提供完整契合的专业映射。


## 02

DDD 的领域就是这个边界内要解决的业务问题域。

子域就是对领域的进一步划分。

核心思想就是将问题域逐步分解，降低业务理解和系统实现的复杂度。

在领域不断划分的过程中，领域会细分为不同的子域，子域可以根据自身重要性和功能属性划分为三类子域，它们分别是：核心域、通用域和支撑域。

决定产品和公司核心竞争力的子域是核心域，它是业务成功的主要因素和公司的核心竞争力。

没有太多个性化的诉求，同时被多个子域使用的通用功能子域是通用域。还有一种功能子域是必需的，但既不包含决定产品和公司核心竞争力的功能，也不包含通用功能的子域，它就是支撑域。

核心域、支撑域和通用域的主要目标是：通过领域划分，区分不同子域在公司内的不同功能属性和重要性，从而公司可对不同子域采取不同的资源投入和建设策略，其关注度也会不一样。

【主要是针对核心域和支撑域来说的，一堆子域，只有几个是核心的，所以资源投入要有优先级。】


## 03

DDD 中 “通用语言”和“限界上下文”这两个重要的概念。在事件风暴过程中，通过团队交流达成共识的，能够简单、清晰、准确描述业务涵义和规则的语言就是通用语言。

【通用语言在不同领域需要建立映射。】

限界上下文：解决通用语言在不同领域的上下文语义问题。限界就是领域的边界，而上下文则是语义环境。

领域边界就是通过限界上下文来定义的。比如商城来说，展示是商品，运输是货物，所以不同。理论上限界上下文就是微服务的边界。我们将限界上下文内的领域模型映射到微服务，就完成了从问题域到软件的解决方案。


## 04

实体和值对象是组成领域模型的基础单元。

商品是商品上下文的一个实体，通过唯一的商品 ID 来标识，不管这个商品的数据如何变化，商品的 ID 一直保持不变，它始终是同一个商品。

一般来说，实体与持久化对象是一对一，代码上也是充血模型。

有些复杂场景下，实体与持久化对象则可能是一对多或者多对一的关系。在 DDD 中用来描述领域的特定方面，并且是一个没有标识符的对象，叫作值对象，比如人员地址。人员是实体，地址对象以属性的形式嵌入到人员实体里。

DDD 提倡从领域模型设计出发，而不是先设计数据模型。

【 就拿 人员和地址，某些领域，人员是实体，地址是值对象，某些领域，地址是实体，包含的人员是值对象。 】


## 05

聚合（Aggregate）和聚合根（AggregateRoot）领域模型内的实体和值对象就好比个体，而能让实体和值对象协同工作的组织就是聚合，它用来确保这些领域对象在实现共同的业务逻辑时，能保证数据的一致性。

聚合就是由业务和逻辑紧密关联的实体和值对象组合而成的，聚合是数据修改和持久化的基本单元，每一个聚合对应一个仓储，实现数据的持久化。

聚合有一个聚合根和上下文边界，这个边界根据业务单一职责和高内聚原则，定义了聚合内部应该包含哪些实体和值对象，而聚合之间的边界是松耦合的。按照这种方式设计出来的微服务很自然就是“高内聚、低耦合”的。

聚合在 DDD 分层架构里属于领域层，领域层包含了多个聚合，共同实现核心业务逻辑。聚合内实体以充血模型实现个体业务能力，以及业务逻辑的高内聚。

跨多个实体的业务逻辑通过领域服务来实现，跨多个聚合的业务逻辑通过应用服务来实现。比如有的业务场景需要同一个聚合的 A 和 B 两个实体来共同完成，我们就可以将这段业务逻辑用领域服务来实现；而有的业务逻辑需要聚合 C 和聚合 D 中的两个服务共同完成，这时你就可以用应用服务来组合这两个服务。如果把聚合比作组织，那聚合根就是这个组织的负责人。

聚合根也称为根实体，它不仅是实体，还是聚合的管理者。聚合根 作为实体本身，拥有实体的属性和业务行为，实现自身的业务逻辑。其次它作为聚合的管理者，在聚合内部负责协调实体和值对象按照固定的业务规则协同完成共同的业务逻辑。在聚合之间，它还是聚合对外的接口人，以聚合根 ID 关联的方式接受外部任务和请求，在上下文内实现聚合之间的业务协同。也就是说，聚合之间通过聚合根 ID 关联引用，如果需要访问其它聚合的实体，就要先访问聚合根，再导航到聚合内部实体，外部对象不能直接访问聚合内实体。--- DDD 领域建模通常采用事件风暴，它通常采用用例分析、场景分析和用户旅程分析等方法，通过头脑风暴列出所有可能的业务行为和事件，然后找出产生这些行为的领域对象，并梳理领域对象之间的关系，找出聚合根，找出与聚合根业务紧密关联的实体和值对象，再将聚合根、实体和值对象组合，构建聚合。

聚合的一些设计原则，参阅原文。1. 在一致性边界内建模真正的不变条件。2. 设计小聚合。3. 通过唯一标识引用其它聚合。4. 在边界之外使用最终一致性。5. 通过应用层实现跨聚合的服务调用。


## 06

领域事件是领域模型中非常重要的一部分，用来表示领域中发生的事件。

一个领域事件将导致进一步的业务操作，在实现业务解耦的同时，还有助于形成完整的业务闭环。

领域事件处理包括：事件构建和发布、事件数据持久化、事件总线、消息中间件、事件接收和处理等。


## 07

DDD 分层架构就是优化后的四层架构。包括 用户接口层、应用层、领域层和基础层。看原文图，在应用层来封装领域层。

DDD 分层架构有一个重要的原则：每层只能与位于其下方的层发生耦合。

架构根据耦合的紧密程度又可以分为两种：严格分层架构和松散分层架构。

优化后的 DDD 分层架构模型就属于严格分层架构，任何层只能对位于其直接下方的层产生依赖。而传统的 DDD 分层架构则属于松散分层架构，它允许某层与其任意下方的层发生依赖。

在严格分层架构中，领域服务只能被应用服务调用，而应用服务只能被用户接口层调用，服务是逐层对外封装或组合的，依赖关系清晰。

而在松散分层架构中，领域服务可以同时被应用层或用户接口层调用，服务的依赖关系比较复杂且难管理，甚至容易使核心业务逻辑外泄。

DDD 分层架构如何推动架构演进？—— 自己看，主要牵涉到领域模型的拆分根合并。结合具体的使用情况进行拆分合并。三层架构如何演进到 DDD 分层架构？—— 自己看。


## 08

微服务架构模型：整洁架构又名“洋葱架构”，整洁架构的层就像洋葱片一样，它体现了分层的设计思想。

在整洁架构里，同心圆代表应用软件的不同部分，从里到外依次是领域模型、领域服务、应用服务和最外围的容易变化的内容，比如用户界面和基础设施。

整洁架构最主要的原则是依赖原则，它定义了各层的依赖关系，越往里依赖越低，代码级别越高，越是核心能力。外圆代码依赖只能指向内圆，内圆不需要知道外圆的任何情况。

六边形架构又名“端口适配器架构”。追溯微服务架构的渊源，一般都会涉及到六边形架构。六边形架构的核心理念是：应用是通过端口与外部进行交互的。我想这也是微服务架构下 API 网关盛行的主要原因吧。核心业务逻辑（应用程序和领域模型）与外部资源（包括 APP、Web 应用以及数据库资源等）完全隔离，仅通过适配器进行交互。它解决了业务逻辑与用户界面的代码交错问题，很好地实现了前后端分离。

六边形架构各层的依赖关系与整洁架构一样，都是由外向内依赖。可以说，这三种架构都考虑了前端需求的变与领域模型的不变。需求变幻无穷，但变化总是有矩可循的，用户体验、操作习惯、市场环境以及管理流程的变化，往往会导致界面逻辑和流程的多变。但总体来说，不管前端如何变化，在企业没有大的变革的情况下，核心领域逻辑基本不会大变，所以领域模型相对稳定，而用例和流程则会随着外部应用需求而随时调整。把握好这个规律，我们就知道该如何设计应用层和领域层了。自己看。

中台本质上是领域的子域，它可能是核心域，也可能是通用域或支撑域。