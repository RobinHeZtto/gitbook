# Restful

**一、起源**

&#x20;       REST这个词，是[Roy Thomas Fielding](http://en.wikipedia.org/wiki/Roy_Fielding)（HTTP 1.0和1.1的主要设计者、Apache服务器软件的作者之一、Apache基金会的第一任主席）在他2000年的[博士论文](http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)中提出的。

![](<../../../.gitbook/assets/image (297).png>)

**二、名称**

&#x20;       Fielding将他对互联网软件的架构原则，定名为**REST**，即**Representational State Transfe**r的缩写。中译"**表现层状态转化**"。如果一个架构符合REST原则，就称它为RESTful架构。

&#x20;       **要理解RESTful架构，最好的方法就是去理解Representational State Transfer这个词组到底是什么意思，它的每一个词代表了什么涵义。**&#x5982;果你把这个名称搞懂了，也就不难体会REST是一种什么样的设计。

**三、资源（Resources）**

&#x20;       REST的名称"表现层状态转化"中，省略了主语。"表现层"其实指的是"资源"（Resources）的"表现层"。

&#x20;       **所谓"资源"，就是网络上的一个实体，或者说是网络上的一个具体信息。**&#x5B83;可以是一段文本、一张图片、一首歌曲、一种服务，总之就是一个具体的实在。你可以用一个URI（统一资源定位符）指向它，每种资源对应一个特定的URI。要获取这个资源，访问它的URI就可以，因此URI就成了每一个资源的地址或独一无二的识别符。所谓"上网"，就是与互联网上一系列的"资源"互动，调用它的URI。

**四、表现层（Representation）**

&#x20;       "资源"是一种信息实体，它可以有多种外在表现形式。**我们把"资源"具体呈现出来的形式，叫做它的"表现层"（Representation）。**

&#x20;       比如，文本可以用txt格式表现，也可以用HTML格式、XML格式、JSON格式表现，甚至可以采用二进制格式；图片可以用JPG格式表现，也可以用PNG格式表现。

&#x20;       URI只代表资源的实体，不代表它的形式。严格地说，有些网址最后的".html"后缀名是不必要的，因为这个后缀名表示格式，属于"表现层"范畴，而URI应该只代表"资源"的位置。它的具体表现形式，应该在HTTP请求的头信息中用Accept和Content-Type字段指定，这两个字段才是对"表现层"的描述。

**五、状态转化（State Transfer）**

&#x20;       访问一个网站，就代表了客户端和服务器的一个互动过程。在这个过程中，势必涉及到数据和状态的变化。

&#x20;       互联网通信协议HTTP协议，是一个无状态协议。这意味着，所有的状态都保存在服务器端。因此，**如果客户端想要操作服务器，必须通过某种手段，让服务器端发生"状态转化"（State Transfer）。而这种转化是建立在表现层之上的，所以就是"表现层状态转化"。**

&#x20;       客户端用到的手段，只能是HTTP协议。具体来说，就是HTTP协议里面，四个表示操作方式的动词：GET、POST、PUT、DELETE。它们分别对应四种基本操作：**GET用来获取资源，POST用来新建资源（也可以用于更新资源），PUT用来更新资源，DELETE用来删除资源。**

**六、综述**

综合上面的解释，总结一下什么是RESTful架构：

　　（1）每一个URI代表一种资源；

　　（2）客户端和服务器之间，传递这种资源的某种表现层；

　　（3）客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。

**七、误区**

&#x20;       RESTful架构有一些典型的设计误区。**最常见的一种设计错误，就是URI包含动词。**&#x56E0;为"资源"表示一种实体，所以应该是名词，URI不应该有动词，动词应该放在HTTP协议中。

&#x20;       举例来说，某个URI是/posts/show/1，其中show是动词，这个URI就设计错了，正确的写法应该是/posts/1，然后用GET方法表示show。

&#x20;       如果某些动作是HTTP动词表示不了的，你就应该把动作做成一种资源。比如网上汇款，从账户1向账户2汇款500元，错误的URI是：

> 　　POST /accounts/1/transfer/500/to/2

&#x20;       正确的写法是把动词transfer改成名词transaction，资源不能是动词，但是可以是一种服务：

> 　　POST /transaction HTTP/1.1\
> 　　Host: 127.0.0.1\
> 　　\
> 　　from=1\&to=2\&amount=500.00

&#x20;       **另一个设计误区，就是在URI中加入版本号**：

> 　　http://www.example.com/app/1.0/foo
>
> 　　http://www.example.com/app/1.1/foo
>
> 　　http://www.example.com/app/2.0/foo

&#x20;       因为不同的版本，可以理解成同一种资源的不同表现形式，所以应该采用同一个URI。版本号可以在HTTP请求头信息的Accept字段中进行区分（参见[Versioning REST Services](http://www.informit.com/articles/article.aspx?p=1566460)）：

> 　　Accept: vnd.example-com.foo+json; version=1.0
>
> 　　Accept: vnd.example-com.foo+json; version=1.1
>
> 　　Accept: vnd.example-com.foo+json; version=2.0
