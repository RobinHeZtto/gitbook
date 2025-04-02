# 关于应用架构的思考总结

## 目录

## 1. 先从分层开始

在分解复杂的软件系统时，**分层**是用得最多的技术之一。通过分层来分解整个系统，各个子系统以层次结构方式组织，以此隔离不同的**关注点（**&#x43;oncern Poin&#x74;**）**，同时每一层都依托在其下层之上，上层使用下层的服务，而下层对其上层一无所知。

![](broken-reference)

分层架构减小了整个系统的复杂度，层次间的依赖性，层次的具体实现可灵活替换，以此应对不同需求的变化，并且不影响其他层业务。

同时分层也有弊端，增加一个层次意味这多一个层次转换，过多层次可能会带来性能问题。除性能的影响以外，单层的修改可能引起多层级的**关联修改**，因为层次并不一定能独立封装所有东西，举个例子，后端或数据库一个字段的改变，可能会影响到数据到界面所有相关层次的修改。

**所以分层的关键在于：建立哪些层次？如何确定每一层的领域（职责）边界？**

### **三层架构**

**3-tier architecture**

**表现层**－实现用户界面

**业务逻辑层（领域层）**－实现领域（业务）逻辑

**数据层**－存取数据



遵循关注分离原则, 我的建议就是把一个项目分成三个层次，每个层次拥有自己的目的并且各自独立于堆放运作。 值得一提的是，每一层次使用其自有的数据模型以达到独立性的目的（大家可以看到，在代码中需要一个数据映射器来完成数据转换

![](broken-reference)

{% embed url="https://hung-yanbin.medium.com/android-%E7%9A%84-repository-patter-%E5%88%B0%E5%BA%95%E6%98%AF%E4%BB%80%E9%BA%BC-3bc61f1e402b" %}

## 2. Android中的分层

### MVC/MVP/MVVM

#### xx <a href="#zu-jian-hua-fen-ceng-jie-gou" id="zu-jian-hua-fen-ceng-jie-gou"></a>

### 组件化分层

###



