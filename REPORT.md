# 金融数据分析报告：API调用与SQLite数据库管理

---

## 一、项目背景与目标

本项目旨在掌握金融数据的获取、存储与分析技能：
- **数据获取**：通过 FRED API 和 baostock API 获取宏观数据与A股数据
- **数据存储**：使用 SQLite 建立本地数据库，实现数据持久化
- **数据分析**：通过 SQL 查询完成多维度金融分析

---

## 二、数据获取

### 2.1 FRED 宏观经济数据

通过 FRED API 获取6个关键经济指标（2000-01 至 2026-03，月度频率）：

| Series ID | 指标 | 均值 | 说明 |
|-----------|------|------|------|
| DGS10 | 10年期国债收益率 | ~2.8% | 长期利率基准 |
| DGS2 | 2年期国债收益率 | ~1.9% | 短期利率基准 |
| FEDFUNDS | 联邦基金利率 | ~1.5% | 货币政策工具 |
| CPIAUCSL | CPI未季调 | ~210 | 通胀指标 |
| UNRATE | 失业率 | ~6.1% | 就业状况 |
| DEXCHUS | USD/CNY汇率 | ~7.0 | 汇率水平 |

**数据处理**：
- 统一频率为月度（`frequency="m"`）
- 缺失值采用前向填充（ffill）+ 后向填充（bfill）

### 2.2 A股行情数据

通过 baostock 获取：
- 沪深300指数（sh.000300）
- 10只自选股票：贵州茅台、招商银行、中国平安、中信证券、中国中免、比亚迪、宁德时代、海光信息、海康威视、恒瑞医药

---

## 三、数据库设计

### 3.1 表结构

```sql
-- 宏观数据表
CREATE TABLE macro_data (
    date TEXT NOT NULL,
    series_id TEXT NOT NULL,
    value REAL,
    PRIMARY KEY (date, series_id)
);

-- 股票行情表
CREATE TABLE stock_price (
    code TEXT NOT NULL,
    date TEXT NOT NULL,
    open REAL, high REAL, low REAL, close REAL,
    volume REAL, amount REAL, adjustflag TEXT,
    PRIMARY KEY (code, date)
);

-- 股票信息表
CREATE TABLE stock_info (
    code TEXT PRIMARY KEY,
    code_name TEXT,
    ipoDate TEXT
);
```

### 3.2 数据量

| 表名 | 记录数 | 说明 |
|------|--------|------|
| macro_data | ~1884 | 6序列 × 314月 |
| stock_price | ~10万+ | 多股票多年度日度数据 |
| stock_info | ~11 | 沪深300 + 10只自选股 |

---

## 四、分析任务与结果

### 4.1 必做1：收益率曲线利差分析

**分析方法**：计算10Y-2Y国债利差，识别经济衰退领先指标

**主要发现**：
- 利差均值：**1.06%**
- 2000年初短暂倒挂（-0.41%），对应科技股泡沫调整
- 2006-2007年倒挂，领先2007年12月金融危机
- 2019年再次倒挂，领先2020年新冠冲击

**经济意义**：利差倒挂是经济衰退的可靠领先指标，领先时间约6-18个月

### 4.2 必做2：股票年度统计

**分析方法**：按年度聚合计算平均收盘价和总成交量

**主要发现**：
- 沪深300指数年均点位在3000-5000区间波动
- 2015年成交量显著放大（牛市期间）
- 宁德时代等成长股成交量逐年放大

### 4.3 必做3：上市时间筛选

**分析方法**：筛选上市超过10年的股票

**主要发现**：通过SQL的`julianday`函数计算上市时长，识别成熟型企业

### 4.4 自定义1：菲利普斯曲线验证

**分析方法**：对比CPI同比变化率与失业率的相关性

**主要发现**：
- 两个指标存在一定负相关性
- 2022年出现例外：高通胀与低失业率并存（通胀型复苏）

### 4.5 自定义2：美联储加息周期与汇率

**分析方法**：识别加息周期，计算期间USD/CNY汇率变动

**主要发现**：
- 加息周期中USD/CNY趋于上行（人民币贬值）
- 降息周期中USD/CNY趋于下行（人民币升值）
- 2015-2018加息周期：人民币从6.1贬值至6.9

---

## 五、代码实现亮点

### 5.1 通用API请求函数

```python
def fetch_fred_data(series_id, start_date, end_date, frequency="m"):
    """支持任意序列ID、时间范围和频率的FRED API请求"""
    params = {
        "series_id": series_id,
        "api_key": API_KEY,
        "file_type": "json",
        "frequency": frequency
    }
    response = requests.get(BASE_URL, params=params)
    return pd.Series(data)
```

### 5.2 批量下载与重试机制

```python
def batch_fetch_stocks(stock_codes, max_retries=3):
    """支持失败重试的批量下载"""
    for attempt in range(max_retries):
        df = fetch_stock_data(code)
        if df is not None:
            break
        time.sleep(1)  # 重试前等待
```

### 5.3 增量更新策略

```python
def update_if_newer(conn, table, date_column):
    """检测最新日期，只下载增量数据"""
    latest = pd.read_sql_query(f"SELECT MAX({date_column}) FROM {table}", conn)
    # 比较后决定下载范围
```

---

## 六、结论与改进

### 6.1 主要结论

1. **数据获取**：成功通过双API获取宏观+A股数据，统一为月度频率
2. **数据库设计**：合理的表结构设计，支持高效SQL查询
3. **分析应用**：
   - 验证了收益率曲线作为衰退领先指标的有效性
   - 展示了菲利普斯曲线在当代的局限性
   - 量化了美联储政策对人民币汇率的影响

### 6.2 改进方向

1. **数据丰富**：加入更多宏观指标（GDP、M2、PPI等）
2. **行业分类**：baostock缺乏行业字段，可通过tushare补充
3. **自动更新**：实现定时任务自动更新数据库
4. **数据质量检验**：检测异常值、缺失交易日等

---

## 七、参考资料

- FRED API文档：https://fred.stlouisfed.org/docs/api/fred/
- baostock文档：http://baostock.com/baostock/index.php
- SQLite语法：https://www.sqlite.org/lang.html
