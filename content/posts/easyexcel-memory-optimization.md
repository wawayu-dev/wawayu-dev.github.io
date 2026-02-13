---
title: "EasyExcel 百万级数据导出的内存优化实战"
date: 2025-05-22
tags: ["Java", "性能优化", "问题排查", "最佳实践"]
categories: ["技术"]
description: "深入讲解如何使用 EasyExcel 实现百万级数据导出，从内存溢出到性能优化，提供完整的生产级解决方案"
draft: false
---

在企业级应用中，大数据量导出是一个常见但棘手的问题。传统的 Apache POI 在处理大文件时容易出现内存溢出，而阿里开源的 EasyExcel 通过优化内存占用，可以轻松处理百万级数据导出。

本文将从实际问题出发，深入讲解如何使用 EasyExcel 实现高性能的大数据量导出，并分享生产环境中的优化经验。

### 本文亮点
- [x] 理解传统 POI 的内存问题
- [x] 掌握 EasyExcel 的核心原理
- [x] 学会百万级数据导出的最佳实践
- [x] 了解异步导出与进度反馈方案
- [x] 掌握生产环境的性能调优技巧

---

## 传统 POI 的内存问题

### 问题场景

某电商系统需要导出订单数据，数据量达到 100 万条。使用传统的 Apache POI 实现：

```java
// 传统 POI 实现（有问题的代码）
public void exportOrders() {
    Workbook workbook = new XSSFWorkbook();
    Sheet sheet = workbook.createSheet("订单数据");
    
    // 查询 100 万条数据
    List<Order> orders = orderMapper.selectAll();
    
    // 写入 Excel
    for (int i = 0; i < orders.size(); i++) {
        Row row = sheet.createRow(i);
        Order order = orders.get(i);
        row.createCell(0).setCellValue(order.getOrderNo());
        row.createCell(1).setCellValue(order.getUserName());
        // ... 更多字段
    }
    
    // 输出文件
    FileOutputStream fos = new FileOutputStream("orders.xlsx");
    workbook.write(fos);
    fos.close();
}
```

### 内存分析

这段代码存在严重的内存问题：

1. **一次性加载所有数据**：100 万条数据全部加载到内存
2. **POI 对象占用大**：每个 Cell、Row、Sheet 都是 Java 对象
3. **内存无法释放**：数据写入前必须全部保存在内存中

**内存占用估算**：
- 每条订单数据：约 1KB
- 100 万条数据：约 1GB
- POI 对象开销：约 3-5 倍
- **总内存占用：3-5GB**

> **架构思考：** 传统 POI 采用 DOM 模式，需要在内存中构建完整的文档对象模型。这种方式虽然灵活，但在处理大文件时会导致内存溢出。EasyExcel 采用 SAX 模式，边读边写，大大降低了内存占用。

---

## EasyExcel 核心原理

### 内存优化策略

EasyExcel 通过以下方式优化内存：

1. **流式写入**：不在内存中保存完整的 Excel 对象
2. **分批处理**：支持分批查询和写入
3. **临时文件**：使用临时文件缓存数据
4. **对象复用**：复用 Java 对象，减少 GC 压力

### 内存占用对比

| 方案 | 10万条 | 50万条 | 100万条 |
|------|--------|--------|---------|
| Apache POI | 300MB | 1.5GB | OOM |
| EasyExcel | 20MB | 50MB | 100MB |

---

## EasyExcel 基础使用

### 项目依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>3.3.2</version>
</dependency>
```

### 定义数据模型

```java
@Data
public class OrderExportDTO {
    
    @ExcelProperty(value = "订单号", index = 0)
    private String orderNo;
    
    @ExcelProperty(value = "用户名", index = 1)
    private String userName;
    
    @ExcelProperty(value = "商品名称", index = 2)
    private String productName;
    
    @ExcelProperty(value = "订单金额", index = 3)
    @NumberFormat("#.##")
    private BigDecimal amount;
    
    @ExcelProperty(value = "订单状态", index = 4)
    @ExcelProperty(converter = OrderStatusConverter.class)
    private Integer status;
    
    @ExcelProperty(value = "创建时间", index = 5)
    @DateTimeFormat("yyyy-MM-dd HH:mm:ss")
    private Date createTime;
}
```

### 简单导出示例

```java
@Service
public class OrderExportService {
    
    @Autowired
    private OrderMapper orderMapper;
    
    /**
     * 简单导出（适用于小数据量）
     */
    public void simpleExport(HttpServletResponse response) throws IOException {
        // 查询数据
        List<OrderExportDTO> data = orderMapper.selectForExport();
        
        // 设置响应头
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setCharacterEncoding("utf-8");
        String fileName = URLEncoder.encode("订单数据", "UTF-8");
        response.setHeader("Content-disposition", "attachment;filename=" + fileName + ".xlsx");
        
        // 写入 Excel
        EasyExcel.write(response.getOutputStream(), OrderExportDTO.class)
                .sheet("订单数据")
                .doWrite(data);
    }
}
```

---

## 百万级数据导出方案

### 方案一：分页查询 + 流式写入

```java
@Service
@Slf4j
public class LargeDataExportService {
    
    @Autowired
    private OrderMapper orderMapper;
    
    private static final int PAGE_SIZE = 10000; // 每页 1 万条
    
    /**
     * 分页导出大数据量
     */
    public void exportLargeData(HttpServletResponse response) throws IOException {
        // 设置响应头
        setResponseHeader(response, "订单数据");
        
        // 使用 EasyExcel 的流式写入
        ExcelWriter excelWriter = null;
        try {
            excelWriter = EasyExcel.write(response.getOutputStream(), OrderExportDTO.class).build();
            WriteSheet writeSheet = EasyExcel.writerSheet("订单数据").build();
            
            // 分页查询并写入
            int pageNum = 1;
            while (true) {
                log.info("开始导出第 {} 页数据", pageNum);
                
                // 分页查询
                PageHelper.startPage(pageNum, PAGE_SIZE);
                List<OrderExportDTO> data = orderMapper.selectForExport();
                
                if (CollectionUtils.isEmpty(data)) {
                    break;
                }
                
                // 写入当前页数据
                excelWriter.write(data, writeSheet);
                
                log.info("第 {} 页数据导出完成，共 {} 条", pageNum, data.size());
                
                // 如果当前页数据不足一页，说明已经是最后一页
                if (data.size() < PAGE_SIZE) {
                    break;
                }
                
                pageNum++;
            }
            
            log.info("数据导出完成，共 {} 页", pageNum);
        } finally {
            if (excelWriter != null) {
                excelWriter.finish();
            }
        }
    }
    
    private void setResponseHeader(HttpServletResponse response, String fileName) throws UnsupportedEncodingException {
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        response.setCharacterEncoding("utf-8");
        String encodedFileName = URLEncoder.encode(fileName, "UTF-8");
        response.setHeader("Content-disposition", "attachment;filename=" + encodedFileName + ".xlsx");
    }
}
```

### 方案二：游标查询 + 流式写入

对于 MyBatis，可以使用游标（Cursor）进一步优化内存：

```java
/**
 * Mapper 接口
 */
public interface OrderMapper {
    /**
     * 使用游标查询（流式查询）
     */
    @Options(resultSetType = ResultSetType.FORWARD_ONLY, fetchSize = 1000)
    @Select("SELECT * FROM t_order WHERE status = #{status}")
    Cursor<OrderExportDTO> selectByCursor(@Param("status") Integer status);
}

/**
 * 使用游标导出
 */
public void exportWithCursor(HttpServletResponse response) throws IOException {
    setResponseHeader(response, "订单数据");
    
    ExcelWriter excelWriter = null;
    Cursor<OrderExportDTO> cursor = null;
    
    try {
        excelWriter = EasyExcel.write(response.getOutputStream(), OrderExportDTO.class).build();
        WriteSheet writeSheet = EasyExcel.writerSheet("订单数据").build();
        
        // 打开游标
        cursor = orderMapper.selectByCursor(1);
        
        List<OrderExportDTO> batch = new ArrayList<>(PAGE_SIZE);
        int count = 0;
        
        // 遍历游标
        for (OrderExportDTO order : cursor) {
            batch.add(order);
            count++;
            
            // 每 1 万条写入一次
            if (batch.size() >= PAGE_SIZE) {
                excelWriter.write(batch, writeSheet);
                log.info("已导出 {} 条数据", count);
                batch.clear();
            }
        }
        
        // 写入剩余数据
        if (!batch.isEmpty()) {
            excelWriter.write(batch, writeSheet);
            log.info("数据导出完成，共 {} 条", count);
        }
        
    } finally {
        if (cursor != null) {
            cursor.close();
        }
        if (excelWriter != null) {
            excelWriter.finish();
        }
    }
}
```

> **避坑提示：** 使用游标查询时，需要注意：1) 必须在同一个事务中完成；2) 不能使用分页插件；3) 需要设置 fetchSize 控制每次从数据库获取的记录数。

---

## 异步导出方案

对于超大数据量（百万级以上），同步导出会导致请求超时。推荐使用异步导出方案：

### 整体架构

```
用户发起导出请求
    ↓
创建导出任务（返回任务ID）
    ↓
异步线程执行导出
    ↓
导出完成后上传到 OSS
    ↓
通知用户下载
```

### 导出任务表

```sql
CREATE TABLE export_task (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    task_id VARCHAR(64) NOT NULL COMMENT '任务ID',
    file_name VARCHAR(255) NOT NULL COMMENT '文件名',
    status TINYINT NOT NULL COMMENT '状态：0-待处理，1-处理中，2-成功，3-失败',
    total_count INT COMMENT '总记录数',
    exported_count INT COMMENT '已导出记录数',
    file_url VARCHAR(512) COMMENT '文件下载地址',
    error_msg VARCHAR(1000) COMMENT '错误信息',
    create_by VARCHAR(64) COMMENT '创建人',
    create_time DATETIME NOT NULL,
    update_time DATETIME NOT NULL,
    INDEX idx_task_id (task_id),
    INDEX idx_create_by (create_by)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='导出任务表';
```

### 异步导出实现

```java
@Service
@Slf4j
public class AsyncExportService {
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Autowired
    private ExportTaskMapper exportTaskMapper;
    
    @Autowired
    private OssService ossService;
    
    @Autowired
    private ThreadPoolExecutor exportExecutor;
    
    /**
     * 创建导出任务
     */
    public String createExportTask(String userId) {
        // 生成任务 ID
        String taskId = UUID.randomUUID().toString().replace("-", "");
        
        // 创建任务记录
        ExportTask task = new ExportTask();
        task.setTaskId(taskId);
        task.setFileName("订单数据_" + DateUtil.format(new Date(), "yyyyMMddHHmmss") + ".xlsx");
        task.setStatus(0); // 待处理
        task.setCreateBy(userId);
        task.setCreateTime(new Date());
        task.setUpdateTime(new Date());
        exportTaskMapper.insert(task);
        
        // 提交异步任务
        exportExecutor.execute(() -> executeExport(taskId));
        
        return taskId;
    }
    
    /**
     * 执行导出
     */
    private void executeExport(String taskId) {
        ExportTask task = exportTaskMapper.selectByTaskId(taskId);
        
        try {
            // 更新状态为处理中
            task.setStatus(1);
            exportTaskMapper.updateById(task);
            
            // 创建临时文件
            String tempFilePath = "/tmp/" + task.getFileName();
            File tempFile = new File(tempFilePath);
            
            // 导出数据到临时文件
            exportToFile(tempFile, task);
            
            // 上传到 OSS
            String fileUrl = ossService.upload(tempFile, "export/" + task.getFileName());
            
            // 更新任务状态
            task.setStatus(2); // 成功
            task.setFileUrl(fileUrl);
            task.setUpdateTime(new Date());
            exportTaskMapper.updateById(task);
            
            // 删除临时文件
            tempFile.delete();
            
            // 发送通知（邮件、站内信等）
            sendNotification(task);
            
            log.info("导出任务 {} 完成", taskId);
            
        } catch (Exception e) {
            log.error("导出任务 {} 失败", taskId, e);
            
            // 更新任务状态为失败
            task.setStatus(3);
            task.setErrorMsg(e.getMessage());
            task.setUpdateTime(new Date());
            exportTaskMapper.updateById(task);
        }
    }
    
    /**
     * 导出数据到文件
     */
    private void exportToFile(File file, ExportTask task) throws IOException {
        ExcelWriter excelWriter = null;
        Cursor<OrderExportDTO> cursor = null;
        
        try {
            excelWriter = EasyExcel.write(file, OrderExportDTO.class).build();
            WriteSheet writeSheet = EasyExcel.writerSheet("订单数据").build();
            
            cursor = orderMapper.selectByCursor(1);
            
            List<OrderExportDTO> batch = new ArrayList<>(10000);
            int totalCount = 0;
            int exportedCount = 0;
            
            for (OrderExportDTO order : cursor) {
                batch.add(order);
                totalCount++;
                
                if (batch.size() >= 10000) {
                    excelWriter.write(batch, writeSheet);
                    exportedCount += batch.size();
                    batch.clear();
                    
                    // 更新进度
                    task.setTotalCount(totalCount);
                    task.setExportedCount(exportedCount);
                    task.setUpdateTime(new Date());
                    exportTaskMapper.updateById(task);
                    
                    log.info("任务 {} 已导出 {} 条数据", task.getTaskId(), exportedCount);
                }
            }
            
            // 写入剩余数据
            if (!batch.isEmpty()) {
                excelWriter.write(batch, writeSheet);
                exportedCount += batch.size();
            }
            
            // 更新最终计数
            task.setTotalCount(totalCount);
            task.setExportedCount(exportedCount);
            task.setUpdateTime(new Date());
            exportTaskMapper.updateById(task);
            
        } finally {
            if (cursor != null) {
                cursor.close();
            }
            if (excelWriter != null) {
                excelWriter.finish();
            }
        }
    }
    
    /**
     * 查询任务进度
     */
    public ExportTask queryProgress(String taskId) {
        return exportTaskMapper.selectByTaskId(taskId);
    }
    
    /**
     * 发送通知
     */
    private void sendNotification(ExportTask task) {
        // 发送邮件、站内信等
        log.info("发送导出完成通知给用户 {}", task.getCreateBy());
    }
}
```

### 前端轮询进度

```javascript
// 创建导出任务
async function createExportTask() {
    const response = await fetch('/api/export/create', {
        method: 'POST'
    });
    const data = await response.json();
    const taskId = data.taskId;
    
    // 开始轮询进度
    pollProgress(taskId);
}

// 轮询进度
function pollProgress(taskId) {
    const timer = setInterval(async () => {
        const response = await fetch(`/api/export/progress/${taskId}`);
        const task = await response.json();
        
        // 更新进度条
        const progress = task.totalCount > 0 
            ? (task.exportedCount / task.totalCount * 100).toFixed(2)
            : 0;
        updateProgressBar(progress);
        
        // 检查状态
        if (task.status === 2) {
            // 导出成功
            clearInterval(timer);
            showDownloadLink(task.fileUrl);
        } else if (task.status === 3) {
            // 导出失败
            clearInterval(timer);
            showError(task.errorMsg);
        }
    }, 2000); // 每 2 秒轮询一次
}
```

---

## 性能优化技巧

### 1. 数据库查询优化

```java
/**
 * 只查询需要导出的字段
 */
@Select("SELECT order_no, user_name, product_name, amount, status, create_time " +
        "FROM t_order WHERE status = #{status}")
List<OrderExportDTO> selectForExport(@Param("status") Integer status);

/**
 * 添加索引
 */
CREATE INDEX idx_status_create_time ON t_order(status, create_time);
```

### 2. 线程池配置

```java
@Configuration
public class ThreadPoolConfig {
    
    @Bean("exportExecutor")
    public ThreadPoolExecutor exportExecutor() {
        return new ThreadPoolExecutor(
            2,  // 核心线程数
            5,  // 最大线程数
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),
            new ThreadFactoryBuilder()
                .setNameFormat("export-thread-%d")
                .build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

### 3. 自定义样式

```java
/**
 * 自定义表头样式
 */
public class CustomCellWriteHandler implements CellWriteHandler {
    
    @Override
    public void afterCellDispose(WriteSheetHolder writeSheetHolder, 
                                 WriteTableHolder writeTableHolder,
                                 List<WriteCellData<?>> cellDataList, 
                                 Cell cell, 
                                 Head head, 
                                 Integer relativeRowIndex, 
                                 Boolean isHead) {
        if (isHead) {
            // 设置表头样式
            CellStyle cellStyle = cell.getSheet().getWorkbook().createCellStyle();
            cellStyle.setFillForegroundColor(IndexedColors.GREY_25_PERCENT.getIndex());
            cellStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);
            
            Font font = cell.getSheet().getWorkbook().createFont();
            font.setBold(true);
            font.setFontHeightInPoints((short) 12);
            cellStyle.setFont(font);
            
            cell.setCellStyle(cellStyle);
        }
    }
}

// 使用自定义样式
EasyExcel.write(file, OrderExportDTO.class)
        .registerWriteHandler(new CustomCellWriteHandler())
        .sheet("订单数据")
        .doWrite(data);
```

### 4. 数据转换器

```java
/**
 * 订单状态转换器
 */
public class OrderStatusConverter implements Converter<Integer> {
    
    private static final Map<Integer, String> STATUS_MAP = new HashMap<>();
    
    static {
        STATUS_MAP.put(0, "待支付");
        STATUS_MAP.put(1, "已支付");
        STATUS_MAP.put(2, "已发货");
        STATUS_MAP.put(3, "已完成");
        STATUS_MAP.put(4, "已取消");
    }
    
    @Override
    public Class<Integer> supportJavaTypeKey() {
        return Integer.class;
    }
    
    @Override
    public CellDataTypeEnum supportExcelTypeKey() {
        return CellDataTypeEnum.STRING;
    }
    
    @Override
    public WriteCellData<?> convertToExcelData(Integer value, ExcelContentProperty contentProperty,
                                                GlobalConfiguration globalConfiguration) {
        return new WriteCellData<>(STATUS_MAP.getOrDefault(value, "未知"));
    }
}
```

### 5. 内存监控

```java
@Component
@Slf4j
public class MemoryMonitor {
    
    @Scheduled(fixedRate = 5000)
    public void monitorMemory() {
        Runtime runtime = Runtime.getRuntime();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        long maxMemory = runtime.maxMemory();
        
        double usedPercent = (double) usedMemory / maxMemory * 100;
        
        if (usedPercent > 80) {
            log.warn("内存使用率过高：{:.2f}%，已使用：{}MB，最大：{}MB", 
                    usedPercent, 
                    usedMemory / 1024 / 1024,
                    maxMemory / 1024 / 1024);
        }
    }
}
```

> **避坑提示：** 在生产环境中，建议：1) 限制单次导出的最大记录数（如 100 万条）；2) 设置导出任务的超时时间；3) 定期清理过期的导出文件；4) 监控导出任务的执行情况。

---

## 总结与思考

本文深入讲解了如何使用 EasyExcel 实现百万级数据导出，从内存问题到性能优化，提供了完整的解决方案：

- **内存优化**：使用 EasyExcel 替代传统 POI，内存占用降低 90%
- **分批处理**：通过分页查询或游标查询，避免一次性加载所有数据
- **异步导出**：对于超大数据量，使用异步任务 + 进度反馈
- **性能调优**：数据库优化、线程池配置、样式定制等
- **生产实践**：任务管理、文件存储、通知机制等

在实际应用中，还需要考虑：

- **数据安全**：导出文件的访问权限控制
- **并发控制**：限制同时执行的导出任务数量
- **资源隔离**：导出任务使用独立的数据库连接池
- **监控告警**：导出失败率、平均耗时等指标

下次当你需要实现大数据量导出功能时，不妨试试 EasyExcel 这个强大的工具。