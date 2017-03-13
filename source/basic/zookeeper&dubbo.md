

zookeeper是dubbo推荐的注册中心。

[图片]

流程：
1.服务提供者启动时向/dubbo/com.foo.BarService/providers目录下写入URL
2.服务消费者启动时订阅/dubbo/com.foo.BarService/providers目录下的URL向/dubbo/com.foo.BarService/consumers目录下写入自己的URL
3.监控中心启动时订阅/dubbo/com.foo.BarService目录下的所有提供者和消费者URL

支持以下功能：
1.当提供者出现断电等异常停机时，注册中心能自动删除提供者信息。
2.当注册中心重启时，能自动恢复注册数据，以及订阅请求。
3.当会话过期时，能自动恢复注册数据，以及订阅请求。
4.当设置<dubbo:registry check="false" />时，记录失败注册和订阅请求，后台定时重试。
5.可通过<dubbo:registry username="admin" password="1234" />设置zookeeper登录信息。
6.可通过<dubbo:registry group="dubbo" />设置zookeeper的根节点，不设置将使用无根树。
7.支持*号通配符<dubbo:reference group="*" version="*" />，可订阅服务的所有分组和所有版本的提供者。


注意的是阿里内部并没有采用Zookeeper做为注册中心，而是使用自己实现的基于数据库的注册中心，即：Zookeeper注册中心并没有在阿里内部长时间运行的可靠性保障，此Zookeeper桥接实现只为开源版本提供，其可靠性依赖于Zookeeper本身的可靠性。
