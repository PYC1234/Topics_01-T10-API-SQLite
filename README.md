# 金融数据分析：API调用与SQLite数据库管理

## 项目概述

本项目完成金融数据的获取、存储与SQL分析实战任务：
- 通过 FRED API 获取美国宏观经济数据
- 通过 baostock API 获取 A 股行情数据
- 使用 SQLite 建立本地数据库实现数据持久化
- 使用 SQL 完成多维度数据分析

## 项目结构

```
topic10/
├── README.md                        # 本文件
├── 01_api_download.ipynb            # FRED + A股 API数据获取
├── 02_database_setup.ipynb          # SQLite数据库建立与写入
├── 03_sql_analysis.ipynb            # SQL查询与可视化分析
├── fin_data.db                      # SQLite数据库文件
└── output/                          # CSV缓存文件
    ├── macro_data.csv
    ├── hs300_daily.csv
    ├── stock_daily.csv
    └── stock_info.csv
```

## 数据说明

### FRED 宏观数据（2000-01 至 2026-03）

| Series ID | 指标名称 |
|-----------|---------|
| DGS10 | 美国10年期国债收益率 |
| DGS2 | 美国2年期国债收益率 |
| FEDFUNDS | 联邦基金利率 |
| CPIAUCSL | 美国CPI（未季调） |
| UNRATE | 美国失业率 |
| DEXCHUS | 人民币/美元汇率 |

### A股数据（2010-至今）

- 沪深300指数（sh.000300）
- 10只自选股票

## 数据库结构

### macro_data 表
```sql
date TEXT, series_id TEXT, value REAL
PRIMARY KEY (date, series_id)
```

### stock_price 表
```sql
code TEXT, date TEXT, open REAL, high REAL, low REAL,
close REAL, volume REAL, amount REAL, adjustflag TEXT
PRIMARY KEY (code, date)
```

### stock_info 表
```sql
code TEXT PRIMARY KEY, code_name TEXT, ipoDate TEXT
```

## SQL查询示例

### 1. 收益率曲线利差
```sql
SELECT date,
       MAX(CASE WHEN series_id='DGS10' THEN value END) -
       MAX(CASE WHEN series_id='DGS2'  THEN value END) AS spread_10_2
FROM macro_data
GROUP BY date ORDER BY date;
```

### 2. 股票年度统计
```sql
SELECT code, substr(date,1,4) AS year,
       AVG(close) AS avg_close, SUM(volume) AS total_volume
FROM stock_price
GROUP BY code, year;
```

### 3. 上市超过10年的股票
```sql
SELECT code, code_name, ipoDate,
       ROUND((julianday('now') - julianday(ipoDate)) / 365.0, 1) AS listing_years
FROM stock_info
WHERE (julianday('now') - julianday(ipoDate)) / 365.0 > 10;
```

## 环境配置

1. 安装依赖：
```bash
pip install requests pandas matplotlib numpy baostock python-dotenv
```

2. 配置API Key：
   - 在 `.env` 文件中设置 `FRED_API_KEY=your_key`
   - FRED API Key 申请：https://fred.stlouisfed.org/docs/api/api_key.html

## 运行顺序

1. `01_api_download.ipynb` - 获取数据并保存至output目录
2. `02_database_setup.ipynb` - 建立数据库并写入数据
3. `03_sql_analysis.ipynb` - 执行SQL查询与分析

## 注意事项

- `fin_data.db` 文件较大，已加入 `.gitignore`
- 重新生成数据库需依次运行 01 和 02 notebook
- baostock 无需注册，直接使用
