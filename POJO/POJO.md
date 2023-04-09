# POJO

POJO 的定义是无规则简单的对象。为了更好的将 POJO 进行划分，一般会根据不同的使用层对其进行划分，并将其统一命名为：xxxVO、xxxBO...



## PO/DO

PO（persistent object）持久对象/DO（domain object）领域实体对象。

1. 主要用于 DO 层，即数据库层；
2. 只用作存储数据，不做数据操作，即用于将数据库数据映射到PO/DO中。

使用场景：

1. 从数据库中读取数据，然后映射到PO/DO对象中；
2. 将 DTO 数据转成 PO/DO，然后更新或添加到数据库中。



## BO

BO（bussines object）业务层对象。

1. 主要用于 Service 层，即服务层层；
2. 用于聚合多个对象，然后进行业务逻辑处理。

使用场景：

1. 服务层接口中接收到的 DTO 对象转换成 BO 对象进行业务逻辑处理后，再转成 DTO 返回给控制层。



## DTO

DTO（Data Transfer Object）数据传输对象。

1. 主要用于服务间的调用中，传输的数据对象。

使用场景：

1. 主要用于各层之间的数据交互，从而进行解耦。



## VO

VO（view object/value object）表示层对象。

1. 主要用于 Controller 层，即控制层。

使用场景：

1. 控制层接口通过调用各个服务接口获取到 DTO，然后根据前端需要的数据，从 DTO 中获取并转成 VO 对象返回 。



## 总结

控制层：

```java
public class ActivityController {
    @Autowired
    private ActivityPrizeService prizeService;
    @Autowired
    private ActivityTaskService taskService;
    
    public ActivityInfoVO getActivityInfo(@RequestBody ActivityDTO activityDTO) {
        ActivityPrizeDTO prizeDTO = prizeService.getPrizeInfo(activityDTO);
        ActivityTaskDTO taskDTO = taskService.getTaskInfo(activityDTO);
        
        // 通过prizeDTO和taskDTO组装成ActivityInfoVO
        return ActivityInfoVO.builder...build();
    }
}
```

服务层：

```java
public class ActivityPrizeService {
    @Autowired
    private ActivityPrizeDOMapper prizeDOMapper;
    
    public ActivityPrizeDTO getPrizeInfo(@RequestBody ActivityDTO activityDTO) {
        // 通过将 activityDTO 转成ActivityPrizeBO
        ActivityPrizeBO prizeBO = ActivityPrizeBO.builder()...build();
        
        // 通过参数获取到 prizeDO（数据库的数据）
        ActivityPrizeDO prizeDO = prizeDOMapper.select(...);
        // 对prizeBO进行逻辑处理，prizeDO不做任何数据处理，只做为数据库数据的获取。
        
        // 通过prizeBO、prizeDO转成ActivityPrizeDTO
        return ActivityPrizeDTO.builder...build();
    }
}
```

```java
public class ActivityTaskService {
    @Autowired
    private ActivityTaskDOMapper taskDOMapper;
    
    public ActivityTaskDTO getTaskInfo(@RequestBody ActivityDTO activityDTO) {
        // 通过将 activityDTO 转成ActivityTaskBO
        ActivityTaskBO taskBO = ActivityTaskBO.builder()...build();
        
        // 通过参数获取到 taskDO（数据库的数据）
        ActivityTaskDO taskDO = taskDOMapper.select(...);
        // 对taskBO进行逻辑处理，taskDO不做任何数据处理，只做为数据库数据的获取。
        
        // 通过taskBO、taskDO转成ActivityTaskDTO
        return ActivityTaskDTO.builder...build();
    }
}
```

