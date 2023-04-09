# Easyexcel

Java 解析、生成 Excel 比较有名的框架有 Apache poi、jxl。

但上述框架都存在一个严重的问题就是非常的耗内存，当数据量很多时，很容易导致 OOM。

poi 有一套 SAX 模式的 API 可以一定程度的解决一些内存溢出的问题，但POI还是有一些缺陷，比如07版 Excel 解压缩以及解压后存储都是在内存中完成的，内存消耗依然很大。

Easyexcel 是阿里提供的解析 Excel 工具，其重写了 poi 对07版 Excel 的解析，大大的减少了内存消耗。（一个3M的excel用POI sax解析依然需要100M左右内存，改用easyexcel可以降低到几M）03版依赖POI的sax模式，在上层做了模型转换的封装，让使用者更加简单方便。

使用者通过 Easyexcel 封装好的接口和注解，就可以快速实现 Excel 的导入导出。



> 开发文档：
>
> https://easyexcel.opensource.alibaba.com/docs/current/



# 导包

```yaml
implementation 'com.alibaba:easyexcel:3.0.5'
```



# 读取

读取即将 excel 文件的表的每一行数据根据行头和对象字段映射关系转成对象列表。



## EasyExcel.read()

```java
public static ExcelReaderBuilder read(String pathName, Class head, ReadListener readListener) {
    ExcelReaderBuilder excelReaderBuilder = new ExcelReaderBuilder();
    // 读取 File 文件
    excelReaderBuilder.file(pathName);
    if (head != null) {
        // Sheet 表每一行数据映射到那个entity实体类
        excelReaderBuilder.head(head);
    }
    if (readListener != null) {
        // 注册读取监听器
        excelReaderBuilder.registerReadListener(readListener);
    }
    return excelReaderBuilder;
}
```

该方法需要传入三个参数，用于定义读取信息：

* **pathName**：指定 Excel 文件的绝对路径。
* **head**：Sheet 表每一行数据映射到那个entity实体类
* **readListener**：读取监听器，用于处理读取事件、异常事件、读取完成事件



## ReadListener

读取监听器，用于处理读取事件、异常事件、读取完成事件等。



### onException()

`onException(Exception exception, AnalysisContext context)`

在转换异常、获取其他异常下会调用本接口。抛出异常则停止读取。如果这里不抛出异常则继续读取下一行。



### invokeHead()

`invokeHead(Map<Integer, ReadCellData<?>> headMap, AnalysisContext context)`

headMap：key 为第几列，val 为该列信息（如列类型、列名等）

每解析一行行头就会触发一次。

> 如果想将 headMap 转成 Map<Integer,String>
>
> * 方案1： 不实现 ReadListener 而是继承 AnalysisEventListener。AnalysisEventListener 提供 invokeHeadMap(Map<Integer, String> headMap, AnalysisContext context)
> * 方案2： 调用 ConverterUtils.convertToStringMap(headMap, context) 自动会转换



### invoke()

`invoke(T data, AnalysisContext context)`

每读取一条数据就会触发一次。可以在这里对数据进行处理。



### doAfterAllAnalysed()

`doAfterAllAnalysed(AnalysisContext context)`

**单个** Sheet 表读取完会触发一次该方法。



### extra()

`extra(CellExtra extra, AnalysisContext context)`

当单元格有额外信息（批注、超链接、合并单元格信息），则会调用监听器的 `extra()` 方法。

```java
@Slf4j
public class DemoExtraListener implements ReadListener<DemoExtraData> {

    @Override
    public void invoke(DemoExtraData data, AnalysisContext context) {}

    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {}

    @Override
    public void extra(CellExtra extra, AnalysisContext context) {
        log.info("读取到了一条额外信息:{}", JSON.toJSONString(extra));
        switch (extra.getType()) {
            case COMMENT:
                log.info("额外信息是批注,在rowIndex:{},columnIndex;{},内容是:{}", extra.getRowIndex(), extra.getColumnIndex(),
                    extra.getText());
                break;
            case HYPERLINK:
                if ("Sheet1!A1".equals(extra.getText())) {
                    log.info("额外信息是超链接,在rowIndex:{},columnIndex;{},内容是:{}", extra.getRowIndex(),
                        extra.getColumnIndex(), extra.getText());
                } else if ("Sheet2!A1".equals(extra.getText())) {
                    log.info(
                        "额外信息是超链接,而且覆盖了一个区间,在firstRowIndex:{},firstColumnIndex;{},lastRowIndex:{},lastColumnIndex:{},"
                            + "内容是:{}",
                        extra.getFirstRowIndex(), extra.getFirstColumnIndex(), extra.getLastRowIndex(),
                        extra.getLastColumnIndex(), extra.getText());
                } else {
                    Assert.fail("Unknown hyperlink!");
                }
                break;
            case MERGE:
                log.info(
                    "额外信息是超链接,而且覆盖了一个区间,在firstRowIndex:{},firstColumnIndex;{},lastRowIndex:{},lastColumnIndex:{}",
                    extra.getFirstRowIndex(), extra.getFirstColumnIndex(), extra.getLastRowIndex(),
                    extra.getLastColumnIndex());
                break;
            defaultextraRead
        }
    }
}
```

> 由于是流式读取，没法在读取到单元格数据的时候直接读取到额外信息，所以只能最后通知哪些单元格有哪些额外信息



## ExcelReaderSheetBuilder.sheet()

```java
public static ExcelReaderSheetBuilder readSheet(Integer sheetNo, String sheetName) {
    ExcelReaderSheetBuilder excelReaderSheetBuilder = new ExcelReaderSheetBuilder();
    if (sheetNo != null) {
        excelReaderSheetBuilder.sheetNo(sheetNo);
    }
    if (sheetName != null) {
        excelReaderSheetBuilder.sheetName(sheetName);
    }
    return excelReaderSheetBuilder;
}
```

该方法指定读取 Excel 文件中哪一张 Sheet 表。

* **sheetNo**：第几张 Sheet 表。默认为0，即第一张。
* **sheetName**：指定读取的 Sheet 表名。



> 注意：
>
> * 如果 sheetNo 和 sheetName 都为 null，则默认会读取第一张 Sheet 表。
> * 如果两个都传入，则会找符合 sheetNo  或 sheetName的所有表。
> * 同一张 Sheet 只会被读取一次，如果要重复读取同一张 Sheet，则应该多次调用 read 方法。



## @ExcelProperty

在读取过程中默认将 Sheet 表的列值依次映射到指定 Class 的字段。即第一列对应 Class 中第一个字段，依次类推。

可以通过 `@ExcelProperty` 定义字段映射 Sheet 表中的那一列或列名（默认将第一行数据作为列名）

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Activity implements Serializable {
    // 不用@ExcelProperty 表示则默认为第一列
    private Long id;

    @ExcelProperty("活动标题")
    private String title;

    @ExcelProperty(index = 2)
    private String intro;

    private LocalDateTime createAt;

    private LocalDateTime updateAt;
}
```

> 注意：
>
> 1. @ExcelProperty 一般建议只用 index 或者 name，不要同时使用。
> 2. 如果 Class 中出现多个字段的 index 相同，则报异常。
> 3. 如果 Class 中出现多个字段的 name 相同，则只有最后一个字段被赋值。
> 4. 如果 Class 中出现两个字段，分别使用 index 和 name，并最终指向同一列时，则只有使用 index 的字段赋值。
> 5. 如果 Sheet 中出现两个同名的列，则使用 name 只会读取第一列的值。



## @ExcelIgnore

前面说过，ExcelReader 会默认将列映射到实体类对应的字段（默认按照顺序映射）。

实体类有些字段不是映射 Sheet 表列时，如果没有忽略处理，则会自动映射，如果类型转换不兼容，还会报异常。

因此对于这些字段应该使用 `@ExcelIgnore` 标识忽略。

 

## 无对象映射

如果不需要将读取到的行数据信息映射到 entity 实体对象。可以指定类型为 `Map<Integer, String> data`，则会直接将解析到的行信息转成该 Map 类型。key 为列 index，val 为列值。



## 类型转换

Exceleasy 通过 DefaultConverterLoader 提供默认的 Converter 支持。

用户可以对单独的字段指定自定义的 Converter。

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Activity implements Serializable {
    @ExcelProperty(index = 0)
    private Long id;

    @ExcelProperty(value = "活动标题", converter = ActivityTitleConverter.class)
    private String title;

    @ExcelProperty("描述")
    private String intro;

    @NumberFormat("#.##") // 截取小数点后两位
    private String taskNum;

    @DateTimeFormat("yyyy年MM月dd日HH时mm分ss秒")
    private Date date;

    @ExcelIgnore
    private LocalDateTime createAt;

    @ExcelIgnore
    private LocalDateTime updateAt;
}
```

`@NumberFormat`：支持将数字类型的列截取指定数位转换成 String 类型。

`@DateTimeFormat`：按照自定义的时间格式读取列，并转换成 String 或者 Date 类型。

`@ExcelProperty` 可以指定自定义的 Converter 类。

自定义 Converter 需要实现 Converter，并指定映射类型。并通过复写 convertToJavaData() 和 convertToExcelData() 实现自定义读取和写入逻辑。

```java
public class ActivityTitleConverter implements Converter<String> {
    @Override
    public Class<?> supportJavaTypeKey() {
        return String.class;
    }

    @Override
    public CellDataTypeEnum supportExcelTypeKey() {
        return CellDataTypeEnum.STRING;
    }

    /**
     * 读取列值时会调用
     */
    @Override
    public String convertToJavaData(ReadConverterContext<?> context) {
        return "这是活动标题：" + context.getReadCellData().getStringValue();
    }

    /**
     * 写入列值时会调用
     */
    @Override
    public WriteCellData<?> convertToExcelData(WriteConverterContext<String> context) {
        return new WriteCellData<>(context.getValue());
    }

}
```



## 行头数

默认会将 Sheet 表第一行作为列名和实体类进行映射。

如果 Sheet 表存在多行头，则此时需要定义行头数。

通过 `ExcelReaderSheetBuilder.headRowNumber(Integer headRowNumber)` 设置行头数。

默认行头数为1，如果设置行头数超过1，且一列存在多行列名，则默认以最后一行名字作为该列的列名。



## 复杂头

如果行头存在多行，如：

![image-20220722121505666](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20220722121505666.png)

则读取和写入时需要指定多个行名：

```java
@ExcelProperty({"主标题","活动标题"})
private String title;

@ExcelProperty({"主标题","描述"})
private String intro;
```





## 额外信息

额外信息指批注、超链接、合并单元格信息。

可以通过 `ExcelReaderBuilder.extraRead(CellExtraTypeEnum extraType)` 指定需要读取的额外信息。

额外信息默认都不读取。

CellExtraTypeEnum 额外信息类型如下：

* COMMENT：批注。

 * Hyperlink：超链接。
 * MERGE：合并单元格信息。



设置了需要读取的额外信息后，发现额外信息会触发

 `ReadListener.extra(CellExtra extra, AnalysisContext context)` 方法。



## 读取表

前面构建好 `ExcelReaderSheetBuilder` 定义如何读取表。

最后可以通过调用以下方法触发读取 Sheet 表操作。 

* `ExcelReaderSheetBuilder.sheet().doRead()`：读取一张表（有可能为两张表，可参考上方）。
* `ExcelReader.read(List<ReadSheet> readSheetList)`：读取多张表。
* `ExcelReaderSheetBuilder.doReadAll()`：读取所有 Sheet 表。
* `ExcelReaderSheetBuilder.doReadAllSync()`：同步读取所有 Sheet 表。

实际上底层都是调用：

```java
excelAnalyser.analysis(List<ReadSheet> readSheetList, Boolean readAll);
```

如果 `readAll` 为 true，则忽略前面的 `readSheetList`，直接读取所有 Sheet 表；反之则读取指定的 `readSheetList` 表。



> `ExcelReaderSheetBuilder.doReadAllSync()` 和 `ExcelReaderSheetBuilder.doReadAll()` 区别在于：
>
> 前者使用的是内置的 SyncReadListener，该 ReadListener 会将读取到的所有数据存到 List 中，并在读取整张 Sheet 表后返回。
>
> 一般不建议使用，因为如果数据量大，则会大致 List 过大，竟而导致 OOM 异常。



## 案例实践

**ReadListener：**

```java
/**
 * 基本的缓存读取Listener
 * @author CavieLee
 * @since 2022/05/17
 */
@Slf4j
public abstract class BasePageReadListener<T> implements ReadListener<T> {

    /**
     * 缓存数，不建议过大，过大可能导致OOM
     */
    private static final int BATCH_COUNT = 100;
    /**
     * 缓存数据List
     */
    public List<T> cachedDataList;

    public BasePageReadListener() {
        cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);
    }

    public BasePageReadListener(int batchCount) {
        cachedDataList = ListUtils.newArrayListWithExpectedSize(batchCount);
    }

    /**
     * 在转换异常、获取其他异常下会调用本接口。抛出异常则停止读取。如果这里不抛出异常则继续读取下一行。
     */
    @Override
    public void onException(Exception exception, AnalysisContext context) {
        log.error("解析失败，但是继续解析下一行:{}", exception.getMessage());
        if (exception instanceof ExcelDataConvertException) {
            ExcelDataConvertException excelDataConvertException = (ExcelDataConvertException)exception;
            log.error("第{}行，第{}列解析异常，数据为:{}", excelDataConvertException.getRowIndex(),
                    excelDataConvertException.getColumnIndex(), excelDataConvertException.getCellData());
        }
    }

    /**
     * 每解析一行行头就会触发一次
     */
    @Override
    public void invokeHead(Map<Integer, ReadCellData<?>> headMap, AnalysisContext context) {
        log.info("解析到一条头数据:{}", JSON.toJSONString(headMap));
        // 如果想转成 Map<Integer,String>
        // 方案1： 不要implements ReadListener 而是 extends AnalysisEventListener
        // 方案2： 调用 ConverterUtils.convertToStringMap(headMap, context) 自动会转换
    }

    /**
     * 每读取一条数据就会触发一次
     * 处理逻辑：先添加到 cachedDataList缓存
     * 缓存满了就调用自定义逻辑saveData(),并清空缓存
     */
    @Override
    public void invoke(T data, AnalysisContext context) {
        log.info("解析到一条数据:{}", JSON.toJSONString(data));
        boolean needDeal = checkData(data);
        if (!needDeal) {
            return;
        }
        cachedDataList.add(data);
        // 达到缓存上限，需要处理当前缓存数据，并清空缓存（防止缓存导致OOM）
        if (cachedDataList.size() >= BATCH_COUNT) {
            processData();
            // 清空缓存
            cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);
        }
    }

    /**
     * 检查读取到的data是否处理
     */
    public Boolean checkData(T data) {
        return true;
    }

    /**
     * 处理缓存数据
     */
    public void processData() {
    }

    /**
     * 单个 Sheet 表读取完
     * 将 cachedDataList 缓存中的数据存储到数据库
     */
    @Override
    public void doAfterAllAnalysed(AnalysisContext context) {
        // 这里也要保存数据，确保最后遗留的数据也存储到数据库
        processData();
        cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);
        log.info("所有数据解析完成！");
    }

    @Override
    public void extra(CellExtra extra, AnalysisContext context) {
        log.info("读取到了一条额外信息:{}", JSON.toJSONString(extra));
        switch (extra.getType()) {
            case COMMENT:
                log.info("额外信息是批注,在rowIndex:{},columnIndex;{},内容是:{}", extra.getRowIndex(), extra.getColumnIndex(),
                        extra.getText());
                break;
            case HYPERLINK:
                if ("Sheet1!A1".equals(extra.getText())) {
                    log.info("额外信息是超链接,在rowIndex:{},columnIndex;{},内容是:{}", extra.getRowIndex(),
                            extra.getColumnIndex(), extra.getText());
                } else if ("Sheet2!A1".equals(extra.getText())) {
                    log.info(
                            "额外信息是超链接,而且覆盖了一个区间,在firstRowIndex:{},firstColumnIndex;{},lastRowIndex:{},lastColumnIndex:{},"
                                    + "内容是:{}",
                            extra.getFirstRowIndex(), extra.getFirstColumnIndex(), extra.getLastRowIndex(),
                            extra.getLastColumnIndex(), extra.getText());
                } else {
                    Assert.fail("Unknown hyperlink!");
                }
                break;
            case MERGE:
                log.info(
                        "额外信息是超链接,而且覆盖了一个区间,在firstRowIndex:{},firstColumnIndex;{},lastRowIndex:{},lastColumnIndex:{}",
                        extra.getFirstRowIndex(), extra.getFirstColumnIndex(), extra.getLastRowIndex(),
                        extra.getLastColumnIndex());
                break;
            default:
        }
    }
}

```

```java
/**
 * @author CavieLee
 * @since 2022/05/17
 */
@Slf4j
public class ActivityReadListener extends BasePageReadListener<Activity> {

    /**
     * 缓存数，不建议过大，过大可能导致OOM
     */
    private static final int BATCH_COUNT = 100;

    /**
     * 可以是service可以是mapper，用于处理数据
     */
    private ActivityDOMapper activityDOMapper;

    /**
     * 无法被 Spring 管理，因为每一次读取都需要一个新的 Listener
     */
    private ActivityReadListener() {
    }

    /**
     * 如果依赖 Spring 的 Bean，则应该将这些 Bean 通过构造参数传入
     */
    public ActivityReadListener(ActivityDOMapper activityDOMapper) {
        super(BATCH_COUNT);
        this.activityDOMapper = activityDOMapper;
    }

    /**
     * 检查读取到的data是否处理
     */
    @Override
    public Boolean checkData(Activity data) {
        return data.getId() != null;
    }

    /**
     * 处理缓存数据（存储数据库）
     */
    @Override
    public void processData() {
        log.info("{}条数据，开始存储数据库！", cachedDataList.size());
        List<ActivityDO> doList = cachedDataList.stream().map(ActivityDO.CONVERTOR::dtoToDO).collect(Collectors.toList());
        activityDOMapper.insertBatchSomeColumn(doList);
        log.info("存储数据库成功！");
    }
}
```

实体 Class：

```java
/**
 * 活动信息
 * @author CavieLee
 * @since 2022/04/15
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Activity implements Serializable {
    @ExcelProperty(index = 0)
    private Long id;

    @ExcelProperty(value = "活动标题", converter = ActivityTitleConverter.class)
    private String title;

    @ExcelProperty("描述")
    private String intro;

    @ExcelIgnore
    private LocalDateTime createAt;

    @ExcelIgnore
    private LocalDateTime updateAt;
}
```

工具类：

```java
/**
 * @author CavieLee
 * @since 2022/05/16
 */
@Slf4j
public class ReadUtils {
    /**
     * EasyExcel.read(String pathName, Class head, ReadListener readListener).sheet().doRead();
     * pathName: excel 文件绝对路径
     * head: 每一行数据映射到那个entity实体类
     * readListener: 读取监听器，用于处理读取事件、异常事件、读取完成事件
     * 默认读取第一张 Sheet 表读取完后就会自动关闭。
     */
    public void read(String pathName, Class head, ReadListener readListener) {
        read(pathName, head, readListener, null, null, 1);
    }

    public void read(String pathName, Class head, ReadListener readListener, Integer sheetNo) {
        read(pathName, head, readListener, sheetNo, null, 1);
    }

    public void read(String pathName, Class head, ReadListener readListener, String sheetName) {
        read(pathName, head, readListener, null, sheetName, 1);
    }

    public void readWithHeadRowNumber(String pathName, Class head, ReadListener readListener, Integer headRowNumber) {
        read(pathName, head, readListener, null, null, headRowNumber);
    }

    /**
     * sheetNo: 指定读取第几个Sheet表，默认读取第一个。
     * sheetName: 读取指定Sheet表名
     * headRowNumber：默认第一行为列名，可以自定义。如果存在多行head列，则该最后一行的名为列名
     */
    private void read(String pathName, Class head, ReadListener readListener,
                     Integer sheetNo, String sheetName, Integer headRowNumber) {
        EasyExcel.read(pathName, head, readListener).sheet(sheetNo, sheetName).headRowNumber(headRowNumber).doRead();
    }

    /**
     * 读取所有 Sheet 表
     */
    public void readAll(String pathName, Class head, ReadListener readListener) {
        EasyExcel.read(pathName, head, readListener).doReadAll();
    }

    /**
     * readSheetList: 指定读取的 Sheet 表
     */
    public void read(String pathName, List<ReadSheet> readSheetList) {
        ExcelReader excelReader = EasyExcel.read(pathName).build();
        excelReader.read(readSheetList);
        // 关闭excelReader，读的时候会创建临时文件，防止临时文件过大导致磁盘满了
        excelReader.finish();
    }

    /**
     * 读取完所有表，并将所有数据存在 List 中返回。一般不推荐使用，因为数据量大时，消耗大量内存，可能会导致 OOM。
     */
    public <T> List<T> readSync(String pathName, Class head) {
        return EasyExcel.read(pathName).head(head).doReadAllSync();
    }
}
```

```java
public class FileUtil {
    public static InputStream getResourcesFileInputStream(String fileName) {
        return Thread.currentThread().getContextClassLoader().getResourceAsStream("" + fileName);
    }

    public static String getPath(String resourcePath) {
        URL resource = FileUtil.class.getResource(resourcePath);
        return resource == null ? null : resource.getPath();
    }
}
```

测试类：

```java
/**
 * @author CavieLee
 * @since 2022/05/17
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class ReadTest {
    @Autowired
    private ActivityDOMapper activityDOMapper;

    /**
     * 使用自带的 PageReadListener 读取 Sheet 表
     */
    @Test
    public void readByPageReadListenerTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        // 写法1：
        // 使用自带的 PageReadListener，需要传入一个Consumer，定义如何处理读取到的数据
        // PageReadListener自带一个100条数据大小的缓存List
        // 每次读取一条数据都会存入缓存list，直到缓存满了（100条）就会丢给Consumer处理。
        readUtils.read(pathName, Activity.class, new PageReadListener<Activity>(dataList -> {
            for (Activity activity : dataList) {
                System.out.printf("读取到一条数据%s%n", JSON.toJSONString(activity));
            }
        }));
    }

    /**
     * 使用匿名内部类创建ReadListener读取 Sheet 表
     */
    @Test
    public void readByInnerListenerTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        // 写法2：匿名内部类实现ReadListener
        readUtils.read(pathName, Activity.class, new ReadListener<Activity>() {
            /**
             * 单次缓存的数据量
             */
            public static final int BATCH_COUNT = 100;
            /**
             *临时存储
             */
            private List<Activity> cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);

            /**
             * 每读取一条数据就会触发一次
             * 处理逻辑：先添加到 cachedDataList缓存
             * 缓存满了就调用自定义逻辑saveData(),并清空缓存
             */
            @Override
            public void invoke(Activity data, AnalysisContext context) {
                cachedDataList.add(data);
                if (cachedDataList.size() >= BATCH_COUNT) {
                    saveData();
                    // 存储完成清理 list
                    cachedDataList = ListUtils.newArrayListWithExpectedSize(BATCH_COUNT);
                }
            }

            /**
             * 读取完后触发
             * 由于可能残留缓存，因此最后应当将缓存剩余的数据处理
             */
            @Override
            public void doAfterAllAnalysed(AnalysisContext context) {
                saveData();
            }

            /**
             * 将cachedDataList缓存中的数据存储到数据库
             */
            private void saveData() {
                System.out.printf("%s条数据，开始存储数据库！%n", cachedDataList.size());
                System.out.println("存储数据库成功！");
            }
        });
    }

    /**
     * 使用自定义ReadListener读取 Sheet 表
     */
    @Test
    public void readByCustomerReadListenerTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        // 写法3：
        // 传入自定义 ReadListener
        readUtils.read(pathName, Activity.class, new ActivityReadListener(activityDOMapper));
    }

    /**
     * 读取指定的 Sheet 表
     */
    @Test
    public void readCustomerSheetTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        // 写法4：
        // 传入自定义 ReadListener，并指定读取的sheet信息
        readUtils.read(pathName, Activity.class, new ActivityReadListener(activityDOMapper), "活动1");
        readUtils.read(pathName, Activity.class, new ActivityReadListener(activityDOMapper), 1);
    }

    /**
     * 读取所有的 Sheet 表
     */
    @Test
    public void readAllTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        readUtils.readAll(pathName, Activity.class, new ActivityReadListener(activityDOMapper));
    }

    /**
     * 读取指定的多张 Sheet 表
     */
    @Test
    public void readCustomSheetTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        List<ReadSheet> readSheetList = new ArrayList<>();
        ReadSheet readSheet1 =
                EasyExcel.readSheet(0).head(Activity.class).registerReadListener(new ActivityReadListener(activityDOMapper)).build();
        ReadSheet readSheet2 =
                EasyExcel.readSheet(1).head(Activity.class).registerReadListener(new ActivityReadListener(activityDOMapper)).build();
        readSheetList.add(readSheet1);
        readSheetList.add(readSheet2);

        readUtils.read(pathName, readSheetList);
    }

    /**
     * 读取 Sheet 时使用自定义 Convertor
     */
    @Test
    public void readConvertorTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        readUtils.read(pathName, Activity.class, new ActivityReadListener(activityDOMapper));
    }

    /**
     * 自定义head行数
     */
    @Test
    public void readHeadRowNumberTest() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        readUtils.readWithHeadRowNumber(pathName, Activity.class, new ActivityReadListener(activityDOMapper), 2);
    }

    /**
     * 同步的返回，不推荐使用，如果数据量大会把数据放到内存里面
     */
    @Test
    public void synchronousRead() {
        String resourcePath = "/data/activity.xlsx";
        String pathName = FileUtil.getPath(resourcePath);

        ReadUtils readUtils = new ReadUtils();
        List<Activity> dataList = readUtils.readSync(pathName, Activity.class);

        for (Activity data : dataList) {
            // 返回每条数据的键值对 表示所在的列 和所在列的值
            System.out.printf("读取到数据:%s%n", JSON.toJSONString(data));
        }
    }
}
```



## WEB 读取

通过网络上传 excel 文件，通过 file.inputStream 读取。

```java
@PostMapping("upload")
public String upload(MultipartFile file) throws IOException {
    ReadUtils readUtils = new ReadUtils();
    readUtils.read(file.getInputStream(), Activity.class, new ActivityReadListener(activityDOMapper));
    return "success";
}
```

# ![image-20220722113422064](C:\Users\63190\AppData\Roaming\Typora\typora-user-images\image-20220722113422064.png)



# 写出

写出即将对象列表数据根据对象字段与行头信息映射关系转成excel对应的每一行数据。

## EasyExcel.write()

```java
public static ExcelWriterBuilder write(String pathName, Class head) {
    ExcelWriterBuilder excelWriterBuilder = new ExcelWriterBuilder();
    excelWriterBuilder.file(pathName);
    if (head != null) {
        excelWriterBuilder.head(head);
    }
    return excelWriterBuilder;
}
```

该方法需要传入两个个参数，用于定义写出信息：

* **pathName**：指定 Excel 文件的绝对路径。
* **head**：Sheet 表每一行数据映射到那个entity实体类



## ExcelWriteSheetBuilder.sheet()

```java
public ExcelWriterSheetBuilder sheet(String sheetName) {
    return sheet(null, sheetName);
}
public ExcelWriterSheetBuilder sheet(Integer sheetNo, String sheetName) {
    ExcelWriter excelWriter = build();
    ExcelWriterSheetBuilder excelWriterSheetBuilder = new ExcelWriterSheetBuilder(excelWriter);
    if (sheetNo != null) {
        excelWriterSheetBuilder.sheetNo(sheetNo);
    }
    if (sheetName != null) {
        excelWriterSheetBuilder.sheetName(sheetName);
    }
    return excelWriterSheetBuilder;
}
```

和上面读取一样，用于指定写入的表和表明信息。



## 指定或忽略列的写出

`ExcelWriterBuilder` 继承 `AbstractExcelWriterParameterBuilder`，`AbstractExcelWriterParameterBuilder` 提供两个方法：

* `excludeColumnFiledNames(Collection<String> excludeColumnFiledNames)`：指定忽略的列名，被忽略的列名（字段）不会被写入到表中。
* `excludeColumnIndexes(Collection<Integer> excludeColumnIndexes)`：指定忽略index的列，被忽略index的列（字段）不会被写入到表中。
* `includeColumnFiledNames(Collection<String> includeColumnFiledNames)`：指定写入的列名，当列名（字段）满足指定的列名才会被写入到表中。
* `includeColumnIndexes(Collection<Integer> includeColumnIndexes)`：指定写入index的列，被指定idex的列（字段）才会被写入到表中。



> 列和字段映射关系可以参考上面的 @ExcelProperty 。
>
> 注：如果使用 `@ExcelProperty(index = x)`指定字段写入的index，则有可能导致写出时产生空列。
>
> 例子：如果对象有两个字段，而其中一个字段 `@ExcelProperty(index = 2)` ，则会导致第二列是空列。
>
> 为了避免空列的产生，应该使用 `@ExcelProperty(order= x)`，如果产生空列，order 会避免空列，而是自动往后补全。



## 列宽、行高

```java
@Getter
@Setter
@EqualsAndHashCode
@ContentRowHeight(10) // 全局内容行高
@HeadRowHeight(20) // 全局行头行高
@ColumnWidth(25) // 全局列宽
public class WidthAndHeightData {
    @ExcelProperty("字符串标题")
    private String string;
    @ExcelProperty("日期标题")
    private Date date;
    /**
     * 宽度为50
     */
    @ColumnWidth(50)
    @ExcelProperty("数字标题")
    private Double doubleData;
}
```



## 合并单元格

```java
@Getter
@Setter
@EqualsAndHashCode
// 将第6-7行的2-3列合并成一个单元格
// @OnceAbsoluteMerge(firstRowIndex = 5, lastRowIndex = 6, firstColumnIndex = 1, lastColumnIndex = 2)
public class DemoMergeData {
    // 这一列 每隔2行 合并单元格
    @ContentLoopMerge(eachRow = 2)
    @ExcelProperty("字符串标题")
    private String string;
    @ExcelProperty("日期标题")
    private Date date;
    @ExcelProperty("数字标题")
    private Double doubleData;
}
```



## 导出图片

```java
/**
 * @author CavieLee
 * @since 2022/10/18
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@ContentRowHeight(20) // 全局内容行高
@HeadRowHeight(20) // 全局行头行高
@ColumnWidth(50) // 全局列宽
public class ImageDemoData {
    // 根据 File 对象写入图片
    @ExcelProperty("图片1")
    private File fileImage;

    // 根据 InputStream 对象写入图片
    @ExcelProperty("图片2")
    private InputStream inputStreamImage;

    // 根据指定的本地文件path生成File对象写入图片
    // 一定要指定Converter为StringImageConverter，否则String类型默认使用字符串转换器
    @ExcelProperty(value = "图片3", converter = StringImageConverter.class)
    private String stringImage;

    // 写入网上图片
    // @since 2.1.1
    @ExcelProperty("图片4")
    private URL urlImage;

    // 根据文件导出 并设置导出的位置。
    // @since 3.0.0-beta1
    private WriteCellData<Void> writeCellDataFileImage;
}
```



```java
public static void main(String[] args) throws IOException {
    List<ImageDemoData> dataList = new ArrayList<>();
    ImageDemoData data = new ImageDemoData();

    String imagePath = "C:\\Users\\Desktop\\1.jpg";
    File file = new File(imagePath);
    data.setFileImage(file);

    InputStream fileInputStream = new FileInputStream(file);
    data.setInputStreamImage(fileInputStream);

    data.setStringImage(imagePath);

    data.setUrlImage(new URL("https://raw.githubusercontent.com/alibaba/easyexcel/master/src/test/resources/converter/img.jpg"));

    dataList.add(data);

    // 这里演示
    // 需要额外放入文字
    // 而且需要放入2个图片
    // 第一个图片靠左
    // 第二个靠右 而且要额外的占用他后面的单元格
    WriteCellData<Void> writeCellData = new WriteCellData<>();
    data.setWriteCellDataFileImage(writeCellData);
    // 这里可以设置为 EMPTY 则代表不需要其他数据了
    writeCellData.setType(CellDataTypeEnum.STRING);
    writeCellData.setStringValue("额外的放一些文字");

    // 可以放入多个图片
    List<ImageData> imageDataList = new ArrayList<>();
    ImageData imageData = new ImageData();
    imageDataList.add(imageData);
    writeCellData.setImageDataList(imageDataList);
    // 放入2进制图片
    imageData.setImage(FileUtils.readFileToByteArray(new File(imagePath)));
    // 图片类型
    imageData.setImageType(ImageData.ImageType.PICTURE_TYPE_PNG);
    // 上 右 下 左 需要留空
    // 这个类似于 css 的 margin
    // 这里实测 不能设置太大 超过单元格原始大小后 打开会提示修复。暂时未找到很好的解法。
    imageData.setTop(5);
    imageData.setRight(40);
    imageData.setBottom(5);
    imageData.setLeft(5);

    // 放入第二个图片
    imageData = new ImageData();
    imageDataList.add(imageData);
    writeCellData.setImageDataList(imageDataList);
    imageData.setImage(FileUtils.readFileToByteArray(new File(imagePath)));
    imageData.setImageType(ImageData.ImageType.PICTURE_TYPE_PNG);
    imageData.setTop(5);
    imageData.setRight(5);
    imageData.setBottom(5);
    imageData.setLeft(50);
    // 设置图片的位置，假设现在目标是覆盖当前单元格和当前单元格右边的单元格
    // 起点相对于当前单元格为0 当然可以不写
    imageData.setRelativeFirstRowIndex(0);
    imageData.setRelativeFirstColumnIndex(0);
    imageData.setRelativeLastRowIndex(0);
    // 前面3个可以不写  下面这个需要写 也就是 结尾 需要相对当前单元格 往右移动一格
    // 也就是说 这个图片会覆盖当前单元格和 后面的那一格
    imageData.setRelativeLastColumnIndex(1);

    EasyExcel.write("C:\\Users\\Desktop\\图片处理结果.xlsx", ImageDemoData.class)
        .sheet(0, "图片处理结果")
        .doWrite(dataList);
}
```





## 写到文件

```java
/**
 * 导出Excel工具
 * @author CavieLee
 * @since 2022/05/19
 */
@Slf4j
public class WriteUtils {

    public void write(String pathName, Class head, Collection<?> data) {
        write(pathName, head, data, null, null);
    }

    public void write(String pathName, Class head, Collection<?> data, Integer sheetNo) {
        write(pathName, head, data, sheetNo, null);
    }

    public void write(String pathName, Class head, Collection<?> data, String sheetName) {
        write(pathName, head, data, null, sheetName);
    }

    /**
     * sheetNo: 指定写到第几个Sheet表，默认写到第一个。
     * sheetName: 指定写出的Sheet表名
     */
    private void write(String pathName, Class head, Collection<?> data,
                      Integer sheetNo, String sheetName) {
        EasyExcel.write(pathName, head).sheet(sheetNo, sheetName).doWrite(data);
    }

    /**
     * sheetNo: 指定写到第几个Sheet表，默认写到第一个。
     * sheetName: 指定写出的Sheet表名
     * excludeColumnFiledNames: 排除指定列名写入
     */
    public void writeWithExclude(String pathName, Class head, Collection<?> data,
                       Integer sheetNo, String sheetName, Collection<String> excludeColumnFiledNames) {
        EasyExcel.write(pathName, head).sheet(sheetNo, sheetName).excludeColumnFiledNames(excludeColumnFiledNames).doWrite(data);
    }

    /**
     * sheetNo: 指定写到第几个Sheet表，默认写到第一个。
     * sheetName: 指定写出的Sheet表名
     * includeColumnFiledNames: 指定列名写入
     */
    public void writeWithInclude(String pathName, Class head, Collection<?> data,
                       Integer sheetNo, String sheetName, Collection<String> includeColumnFiledNames) {
        EasyExcel.write(pathName, head).sheet(sheetNo, sheetName).includeColumnFiledNames(includeColumnFiledNames).doWrite(data);
    }

    /**
     * writeSheet: 指定写出的 Sheet 表
     */
    public void write(String pathName, Class head, Collection<?> data, WriteSheet writeSheet) {
        ExcelWriter excelWriter = EasyExcel.write(pathName, head).build();
        excelWriter.write(data, writeSheet);
        // 关闭excelWriter
        excelWriter.finish();
    }
}
```

 

## 写到网络请求

```java
@ApiOperation("每周报表生成")
@GetMapping("/week")
public void createWeekendReport(HttpServletResponse response) {
    String dateStr = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
    String fileName = URLEncoder.encode("日报" + dateStr, StandardCharsets.UTF_8).replaceAll("\\+", "%20");

    response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    response.setCharacterEncoding("utf-8");
    response.setHeader("Content-disposition", "attachment;filename*=utf-8''" + fileName + ".xlsx");

    ExcelWriter excelWriter = null;
    try {
        // 1. 由于写到web中，因此需要获取HttpServletResponse的outputStream
        // 2. 由于要同时写到同一个Excel文件的多张表，每个表的head不一样，因此需要获取 ExcelWriter
        excelWriter = EasyExcel.write(response.getOutputStream()).build();
        // 内容
        List<ContentReportData> dailyArticleReport = articleIssueService.createDailyReport();
        // 指定 head 和写入到那张表
        excelWriter.write(dailyArticleReport, EasyExcel.writerSheet(0, "文章每日报表")
                          .head(ContentReportData.class).build());

        List<CommentReportData> dailyCommentReport = commentService.createDailyReport();
            excelWriter.write(dailyCommentReport, EasyExcel.writerSheet(1, "评论每日报表")
                    .head(CommentReportData.class).build());
    } catch (Exception e) {
        throw new BaseError("生成日报Excel失败");
    } finally {
        // 关闭流
        if (excelWriter != null) {
            excelWriter.finish();
        }
    }
}
```



## 写到流中

写入流中需要注意：当关闭 excelWriter 时，才会将数据写入到指定的 OutputStream。

```java
ExcelWriter excelWriter = null;
ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
try {
    excelWriter = EasyExcel.write(byteArrayOutputStream).build();
    // 内容
    List<ContentReportData> dailyArticleReport = articleIssueService.createWeekendReport();
    excelWriter.write(dailyArticleReport, EasyExcel.writerSheet(0, "文章每周报表")
                      .head(ContentReportData.class).build());
} catch (Exception e) {
    throw new BaseError("生成周报Excel失败");
} finally {
    // 关闭流
    if (excelWriter != null) {
        excelWriter.finish();
    }
}
// 关闭流后才会写入到OutputStream
return byteArrayOutputStream.toByteArray();
```



## 写出到企业微信机器人

1. 通过定时任务调用接口，触发生成 Excel 文件并转成 byte[]。

2. 将 Excel 文件通过网络上传到企业微信机器人并获得文件id。
3. 触发企业微信机器人发送指定文件id的文件。

```java
private String uploadExcelDate(String fileName, byte[] excelDate) {
    MediaType contentType = MediaType.parse("application/form-data");
    RequestBody uploadRequestBody = new MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart("file", fileName + ".xlsx", RequestBody.create(excelDate, contentType))
        .build();

    String uploadUrl = wxRobotConfig.getWxUploadUrl() + "?key=" + wxRobotConfig.getWxRobotKey() + "&type=file";
    Request uploadrRequest = new Request.Builder()
        .url(uploadUrl)
        .post(uploadRequestBody)
        .build();

    try {
        var uploadResponse = okHttpClient.newCall(uploadrRequest).execute();
        if (!uploadResponse.isSuccessful()) {
            log.error("excel upload wx fail. code:{}, msg:{}", uploadResponse.code(), uploadResponse.message());
            throw new BaseError("excel上传企业微信失败");
        } else {
            if (uploadResponse.body() != null) {
                WeixinUploadResponse weixinUploadResponse = JSON.parseObject(uploadResponse.body().string(), WeixinUploadResponse.class);
                if (weixinUploadResponse.getErrcode() != 0) {
                    log.error("excel upload wx fail. code:{}, msg:{}", weixinUploadResponse.getErrcode(), weixinUploadResponse.getErrmsg());
                    throw new BaseError("excel上传企业微信失败");
                }
                return weixinUploadResponse.getMedia_id();
            }
        }
    } catch (Exception e) {
        log.error("excel upload wx fail.", e);
        throw new BaseError("excel上传企业微信失败");
    }
    return null;
}

private void callRobotSendFile(String mediaId) {
    WeixinSendFileRequest sendFileObject = WeixinSendFileRequest.builder()
        .msgtype("file")
        .file(WeixinSendFileRequest.WeixinFile.builder()
              .media_id(mediaId)
              .build())
        .build();
    String sendFileStr = JSON.toJSONString(sendFileObject);

    MediaType mediaType = MediaType.parse("application/json");
    RequestBody sendRequestBody = RequestBody.create(sendFileStr, mediaType);

    String callUrl = wxRobotConfig.getWxCallUrl() + "?key=" + wxRobotConfig.getWxRobotKey();

    Request sendFileRequest = new Request.Builder()
        .url(callUrl)
        .post(sendRequestBody)
        .build();

    try {
        var sendFileResponse = okHttpClient.newCall(sendFileRequest).execute();
        if (!sendFileResponse.isSuccessful()) {
            log.error("excel call ex fail. code:{}, msg:{}", sendFileResponse.code(), sendFileResponse.message());
            throw new BaseError("excel发送企业微信失败");
        }
    } catch (IOException e) {
        log.error("excel call ex fail.", e);
        throw new BaseError("excel发送企业微信失败");
    }
}
```

