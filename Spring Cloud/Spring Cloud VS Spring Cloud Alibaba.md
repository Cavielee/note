Spring Cloud VS Spring Cloud Alibaba



Spring Cloud 2020.0.0发布   12月

Spring Cloud 2020.0.0版本**彻底删除**掉了Netflix除Eureka外的**所有**组件。



| Netfilx      | Netfilx | Spring Cloud 2020.0.0     | Spring Cloud Alibaba |
| ------------ | ------- | ------------------------- | -------------------- |
| 熔断    | Hystrix | Resilience4j              | Sentinel |
| 负载均衡  | Ribbon  | Spring Cloud Loadbalancer | Dubbo LD |
| 服务路由      | Zuul    | Spring Cloud Gateway      | Dubbo Proxy |
| 服务注册发现 | eureka server/client | eureka server/client | Nacos |
| Netfilx      | -   | Spring Cloud Config      | Nacos |



https://segmentfault.com/a/1190000022949339