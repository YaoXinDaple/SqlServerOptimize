# SqlServerOptimize

## 索引设计建议
- 优先选择唯一性索引
- 为常作为查询条件的字段建立索引
- 经常需要排序、分组和联合操作的字段建立索引
- 尽量使用数据量少的索引，如果字段数据大，索引表体积相对较大，当发生索引扫描时影响比较严重控制索引数量，索引的字段发生修改时需要更新索引，过多索引会影响表性能
- 删除很少使用的索引，可以通过报表查看索引使用情况
- 复合索引区分度大的字段放在前面
- 区分度小的字段不建议创建索引，比如性别
- 命名建议用索引的字段名作为名称，方便在查询计划中分辨，不建议添加“”之类的前缀，SQLServer索引名称允许跨表重名的
- 查询语句中经常需要对索引中的某一列或者某及列进行排序时，可以在创建索引时，指定该列的索引顺序，查询时可以优化Sort命令的耗时

## 索引的进一步优化使用
  - 对索引设置**条件筛选**
     例：后台任务表 `Status` 有 `已执行`、`进行中`、`待处理` 三种状态，在业务处理中只会对`进行中`和`待处理`两种状态的任务数据进行处理，可以通过对该字段设置条件筛选，减少不必要的索引空间占用，优化查询效率。
```csharp
//EFCore配置方式
b.HasIndex(x => x.TaskStatus)
 .HasFilter($"{nameof(BackgroundTask.TaskStatus)} IN (0, 2)");
```
```sql
--SqlServer命令
CREATE INDEX IX_BackgroundTasks_TaskStatus ON BackgroundTasks(TaskStatus) WHERE TaskStatus IN (0,2);
```
- 索引**包含列**
  对于经常使用会在查询中返回的列，又不适合添加到索引中的情况，可以添加到索引包含列中
```csharp
//EFCore配置方式
b.HasIndex(x => x.TaskStatus)
 .HasFilter($"{nameof(BackgroundTask.TaskStatus)} IN (0, 2)")
 .IncludeProperties(
     nameof(BackgroundTask.TaskData),
     nameof(BackgroundTask.TaskIdentifier)
 );
```
```sql
--SqlServer命令
CREATE NONCLUSTERED INDEX IX_BackgroundTasks_TaskStatus
ON [LemonBackgroundTasks](TaskStatus)
INCLUDE (TaskData,TaskIdentifier);
```

## 两个列都有索引，查询时会走哪一个列的索引

同样的查询条件，如果值不一样，走的索引也会变，因为查询计划会根据**索引统计信息**来判断先走哪个索引效率更高，查询计划会优先使用区分度更高的索引

## 历史表 

用于查询某个表在指定时间或时间段的原始数据，适合审计/防篡改等需求
```sql
CREATE TABLE dbo.Employee
(
	EmployeeID INT NOT NULL PRIMARY KEY CLUSTERED,
	Name  NVARCHAR(100) NOT NULL,
	Position VARCHAR(100) NOT NULL,
	Department VARCHAR(100) NOT NULL,
	Address NVARCHAR(1024) NOT NULL,
	AnnualSalary DECIMAL(10,2)NOT NULL,
	ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START,
	ValidTo DATETIME2 GENERATED ALWAYS AS ROW END,PERIOD FOR SYSTEM_TIME(ValidFrom,ValidTo)
)
WITH(SYSTEM_VERSIONING =ON(HISTORY_TABLE = dbo.EmployeeHistory))

--Temporal queries*
--FOR SYSTEM TIME
--ALL,AS OF,BETWEEN..AND，FROM..TO,CONTAINED IN


SELECT * FROM dbo.Employee FOR SYSTEM_TIME AS OF '2021-09-01 T10:00:00.7230011'
```


## 冷热数据处理

热数据和冷数据放不同的文件组，将热数据所在文件组作为内存优化表

<details>
  <summary>点击展开查看详情</summary>
# SQL Server冷热数据分离与内存优化表配置指南

在SQL Server中实现冷热数据分离并将热数据设置为内存优化表是一种高效的数据库优化策略，可以显著提升系统性能并降低存储成本。以下是详细的实现方案：

## 一、冷热数据分离基础

### 1. 冷热数据概念

- **热数据**：频繁访问和修改的数据，需要高性能存储和快速响应
- **冷数据**：不常访问的历史数据，对访问速度要求较低但需要长期保存

### 2. 区分标准

- **时间维度**：例如将1年内的订单数据视为热数据，1年以上的视为冷数据
- **访问频率**：高频访问的数据视为热数据，低频访问的视为冷数据

## 二、冷热数据分离实现方案

### 1. 使用文件组技术分离冷热数据

#### 创建文件组

```sql
-- 创建数据库时定义文件组
CREATE DATABASE [SalesDB] ON PRIMARY
(
    NAME = N'SalesDB_Primary',
    FILENAME = N'C:\Data\SalesDB_Primary.mdf'
),
FILEGROUP [HOT_DATA]
(
    NAME = N'SalesDB_HotData',
    FILENAME = N'D:\FastStorage\SalesDB_HotData.ndf'
),
FILEGROUP [COLD_DATA]
(
    NAME = N'SalesDB_ColdData',
    FILENAME = N'E:\Archive\SalesDB_ColdData.ndf'
)
LOG ON
(
    NAME = N'SalesDB_log',
    FILENAME = N'C:\Logs\SalesDB_log.ldf'
);
```

#### 将表分配到不同文件组

```sql
-- 热数据表（当前年度订单）
CREATE TABLE dbo.Orders_Current
(
    OrderID INT IDENTITY PRIMARY KEY,
    OrderDate DATETIME NOT NULL,
    CustomerID INT NOT NULL,
    Amount DECIMAL(18,2)
) ON [HOT_DATA];

-- 冷数据表（历史订单）
CREATE TABLE dbo.Orders_Archive
(
    OrderID INT NOT NULL,
    OrderDate DATETIME NOT NULL,
    CustomerID INT NOT NULL,
    Amount DECIMAL(18,2)
) ON [COLD_DATA];
```

### 2. 使用表分区实现冷热分离

```sql
-- 创建分区函数（按年份分区）
CREATE PARTITION FUNCTION OrderDateRangePFN(DATETIME)
AS RANGE RIGHT FOR VALUES 
(
    '2023-01-01', '2024-01-01', '2025-01-01'
);

-- 创建分区方案
CREATE PARTITION SCHEME OrderDateRangePS
AS PARTITION OrderDateRangePFN
TO 
(
    [COLD_DATA],  -- 2023年之前
    [COLD_DATA],  -- 2023年
    [HOT_DATA],   -- 2024年
    [HOT_DATA]    -- 2025年及以后
);

-- 创建分区表
CREATE TABLE dbo.Orders
(
    OrderID INT IDENTITY,
    OrderDate DATETIME NOT NULL,
    CustomerID INT NOT NULL,
    Amount DECIMAL(18,2),
    CONSTRAINT PK_Orders PRIMARY KEY (OrderID, OrderDate)
) ON OrderDateRangePS(OrderDate);
```

## 三、将热数据设置为内存优化表

### 1. 添加内存优化文件组

```sql
-- 添加内存优化文件组
ALTER DATABASE SalesDB 
ADD FILEGROUP MemoryFG CONTAINS MEMORY_OPTIMIZED_DATA;

-- 添加内存优化数据文件
ALTER DATABASE SalesDB 
ADD FILE 
(
    NAME = 'SalesDB_InMemory', 
    FILENAME = 'C:\Data\SalesDB_InMemory'
) TO FILEGROUP MemoryFG;
```

### 2. 创建内存优化热数据表

```sql
-- 创建内存优化热数据表
CREATE TABLE dbo.Orders_Hot_InMemory
(
    OrderID INT NOT NULL PRIMARY KEY NONCLUSTERED,
    OrderDate DATETIME NOT NULL,
    CustomerID INT NOT NULL,
    Amount DECIMAL(18,2),
    INDEX IX_OrderDate HASH (OrderDate) WITH (BUCKET_COUNT = 10000)
) WITH 
(
    MEMORY_OPTIMIZED = ON, 
    DURABILITY = SCHEMA_AND_DATA
);
```

### 3. 配置资源调控器（可选）

```sql
-- 创建专用资源池
ALTER RESOURCE GOVERNOR DISABLE;
GO

CREATE RESOURCE POOL HotDataPool
WITH
(
    MAX_CPU_PERCENT = 50,
    MAX_MEMORY_PERCENT = 30
);
GO

-- 将数据库绑定到资源池
EXEC sys.sp_xtp_bind_db_resource_pool 
    @database_name = N'SalesDB', 
    @pool_name = N'HotDataPool';
GO

-- 重新启用资源调控器
ALTER RESOURCE GOVERNOR RECONFIGURE;
GO

-- 使绑定生效
ALTER DATABASE [SalesDB] SET OFFLINE;
ALTER DATABASE [SalesDB] SET ONLINE;
```

## 四、数据迁移与维护策略

### 1. 定期迁移冷数据

```sql
-- 将超过1年的订单迁移到冷数据表
INSERT INTO dbo.Orders_Archive
SELECT * FROM dbo.Orders_Current
WHERE OrderDate < DATEADD(YEAR, -1, GETDATE());

-- 从热数据表中删除已迁移数据
DELETE FROM dbo.Orders_Current
WHERE OrderDate < DATEADD(YEAR, -1, GETDATE());
```

### 2. 自动化冷热数据迁移

```sql
-- 创建归档控制表
CREATE TABLE dbo.ArchiveControl
(
    TableName VARCHAR(100) PRIMARY KEY,
    RetentionPeriod INT, -- 保留月数
    LastArchiveDate DATETIME,
    Status VARCHAR(20)
);

-- 创建归档存储过程
CREATE PROCEDURE dbo.sp_ArchiveColdData
AS
BEGIN
    DECLARE @TableName VARCHAR(100)
    DECLARE @RetentionPeriod INT
    
    -- 获取需要归档的表
    SELECT @TableName = TableName,
           @RetentionPeriod = RetentionPeriod
    FROM dbo.ArchiveControl
    WHERE Status = 'ACTIVE';
    
    -- 执行归档
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- 迁移数据到归档表
        EXEC('INSERT INTO ' + @TableName + '_Archive
              SELECT * FROM ' + @TableName + '
              WHERE OrderDate < DATEADD(MONTH, -' + @RetentionPeriod + ', GETDATE())');
              
        -- 删除原表数据
        EXEC('DELETE FROM ' + @TableName + '
              WHERE OrderDate < DATEADD(MONTH, -' + @RetentionPeriod + ', GETDATE())');
              
        -- 更新归档日期
        UPDATE dbo.ArchiveControl
        SET LastArchiveDate = GETDATE()
        WHERE TableName = @TableName;
        
        COMMIT;
    END TRY
    BEGIN CATCH
        ROLLBACK;
        -- 记录错误日志
    END CATCH;
END;
```

## 五、注意事项与最佳实践

1. **内存管理**：内存优化表会消耗大量内存，需合理规划服务器内存资源

2. **数据类型限制**：内存优化表不支持TEXT、NTEXT等数据类型，需注意转换

3. **持久性选择**：
   - `SCHEMA_AND_DATA`：同时持久化架构和数据（默认）
   - `SCHEMA_ONLY`：仅持久化架构，重启后数据丢失（适用于临时数据）

4. **索引策略**：内存优化表使用特殊的哈希或范围索引，需根据查询模式设计

5. **备份策略**：
   - 对热数据文件组和内存优化表更频繁备份
   - 对冷数据文件组减少备份频率

6. **性能监控**：定期使用SQL Server Profiler和性能监控工具监测系统性能

通过以上方案，您可以实现SQL Server中冷热数据的有效分离，并将热数据设置为内存优化表，从而显著提升系统性能并优化存储成本。
</details>
