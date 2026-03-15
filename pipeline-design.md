
## Task 1
### 1.1

graph TD
    %% 1. Data Sources
    subgraph Sources [Data Sources]
        A[(Historical Batch Files<br/>CSV/Excel)]
        B[Live Transaction Stream<br/>JSON API/Kafka]
    end

    %% 2. Ingestion Layer
    subgraph Ingestion [Ingestion Layer]
        C[Batch Extractor<br/>PyArrow/S3 Sync]
        D[Stream Consumer<br/>FastAPI/Python]
    end

    %% 3. Landing Zone
    subgraph Landing [Landing Zone - Raw Storage]
        E[(Raw S3 Bucket / Folder<br/>Immutable Objects)]
    end

    %% 4. Validation & Quarantine
    subgraph Quality_Control [Validation Stage]
        F{Validation Engine<br/>Great Expectations/Pydantic}
        G[Quarantine / Dead Letter Area<br/>Rejected Records]
    end

    %% 5. Transformation
    subgraph Transformation [Transformation Stage]
        H[Data Transformer<br/>Pandas/Polars/Spark]
    end

    %% 6. Storage & Consumers
    subgraph Output [Storage Layers & Access]
        I[(Production Data Lake<br/>Parquet/Snappy)]
        J[Consumer Interfaces<br/>BI Tools / SQL / ML Models]
    end

    %% 7. Monitoring (Across all layers)
    subgraph Observability [Monitoring & Alerting]
        K[Logs / Metrics / Slack Alerts]
    end

    %% Connections
    A --> C
    B --> D
    C --> E
    D --> E
    E --> F
    F -- "Failed (Invalid Schema/Nulls)" --> G
    F -- "Passed" --> H
    H --> I
    I --> J

    %% Monitoring Links (Dashed)
    C -.-> K
    D -.-> K
    F -.-> K
    H -.-> K
    I -.-> K



### 1.2

Data Sources - Both past sales documents (Batch) and new sales messages (Stream) are located here. There is no input, output is raw CSV, Excel, or JSON data. File system (local folder) or cloud storage (S3) can be used as technology.

Ingestion Layer - finds, reads data in the sources and brings it into the system (Landing Zone). It takes files from sources as inpus and brings the same files into Landing Zone. Technology can be Pyhton scripts(pandas.read_csv or requests libraries). 

Landing Zone - If a problem(error) happens in pipeline, we can return back here and can reprocess data again. Input is Raw data that comes from ingestion, and output is transferred to Validation stage. Files are groupped by date as raw storage.

Validation stage - Here missing ids, negative prices or wrong date formats are discovered. It takes raw data form Landing zone and passes valid data to Transformation, but invalid data to Quarantine. Quarantine - it is a space where invaild data is collected

Transformation stage - It prepares data for analysis. For example, it merges rows, creates new columns, corrects date format. Input is valid data that comes from Validation stage, it produces cleaned and structured data. Technology can be pandas or faster PyArrow

Storage layers - Here data is clean and fast. It takes clean data from Tranformation and stores in Parquet format. This columnar format is much faster to read than CSV.

Consumer Interfaces - it is stage where Analytics and programmers make decisions by taking cleand data and reading. Input is data that is taken from Storage and it produces reports and ML predictions. For this BI tools and SQL query engines acan be used.

Monitoring - It tracks performance of every other comonent. If a problem happens in any part of pipeline, it alert us and prodeuces dashboard. 




## Task 2
### 2.1

-- Schema validations : 
InvoiceNo : must be non-empty string with 6 characters,
StockCode : must be non-empty string with 5 or 6(with letter) characters, 
Description : string, can be null, but must be cast to string if present,
Quantity : must be whole number(integer), 
InvoiceDate : DateTime, must follow YYYY-MM-DD HH:MM:SS format.
UnitPrice :float, must be a numeric decimal.
CustomerID : string/int, can not be empty,
Country : string, defaults to "Unknown" if null


-- Value range validations :
Quantity - must be > 0 for standard sales, 
           must me > 0 for non-cancellation records,
UnitPrice - must be > 0 for non-cancellation records,
InvoiceDate - must be between 2010-01-01 and the current system date.


-- Business rule validations :
  Return/Cancellation Logic:
     Rule: If InvoiceNo starts with 'C', the Quantity must be negative.Action: If a 'C' invoice has a positive quantity, it is a logical error -> Quarantine

  Standard Transaction Logic:
     Rule: If InvoiceNo is a standard numeric string (no 'C' prefix), the Quantity must be positive.
     Action: If a numeric invoice has non-positive quantity -> Quarantine.

  Revenue Check:
     Rule: UnitPrice must be >= 0 regardless of the invoice type.


### 2.2

-- Schema validations : A record is rejected entirely. If InvoiceNo or StockCode is missing, the record is unusable. It is moved to Quarantine immediately


-- Value range validations : A record is flagged or rejected. If UnitPrice is negative, the record is blocked to prevent financial skewing. It is sent to the Dead Letter Area.


-- Busness rule validations : A record is partially accepted. If Description is missing, we fill it with "N/A" and let it pass. If C-prefix logic fails, the row is rejected.




## Task 3
### 3.1

*Cleaning operations*

-- 1. Lowercasing and removing edges
   Input : Mixed-case Description
   Output : All lowercase Description
   Idempotency : Safe


*Derived columns*

-- 1.Line totals = UnitPrice * Quantity
   Input : UnitPrice, Quantity
   Output : Line totals for each sale
   Idempotency : Safe


-- 2. Cancellation Flags
   Input : Invice record that begins with "C"
   Output : "Flag" column that consists of 0 or 1
   


-- 3. Date Components
   Input : Date
   Output : Hour, Day of Week or Month


*Customer-level aggregations*

-- 1. Total Revenue
   Input : Quantity, UnitPrice and CustomerId
   Output : Total amount of money for each customer
   Idempotency : Not safe


-- 2. Order Count
   Input : InvoiceNo and CustomerID
   Output : Count of unique InvoiceNo
   Idempotency : Not safe


-- 3. Product Diversity
   Input : StockCode and CustomerID
   Output : Count of unique StockCode

-- 4. Recency
   Input : InvoiceDate
   Output : Count of days since the customer's last purchase
   Recency Idempotency: Not safe

   
*Feature engineering for the ML model*

-- 1. Purchase Frequency --
   Input : Order Count / Total Months in Window
   Output : Measures how often a customer returns
   Idempotency : Not safe


-- 2 .Favorite Shopping Hour --
   Input : Hour component From Date column with the highest frequency
   Output : Shows hours that customers do shopping much
   Idempotency: Safe
   

-- 3. Monthly Spending Trend -- 
   Input : revenue over the last 3 months
   Output : Identifies if a customer's interest is growing or fading
   Idempotency : Not safe




### 3.2
Layer   |  Contents    |Format      |  Update Frequency |  Retention
        |              |            |                   |
Raw     |Original CSV  |   CSV      |       Daily       |   Permanent 
        |  files from  |            |                   |
        | source       |            |                   |
        |              |            |                   |
Clean   |  Validated,  |  Parquet   |       Daily       |    3–5 Years
        | type-cast,   |            |                   |
        | and filtered |            |                   |
        |    data      |            |                   |
        |              |            |                   |
Feature |  Customer    |Parquet/SQL |     Weekly        |  Current + 
        |aggregations  |            |      or           |     1 Year
        |and ML inputs |            |    Monthly        |


-- Raw Layer --
  We keep the data in its original format (CSV) because we never want to lose the "raw" evidence. If our cleaning code has a bug, we can always go back to these files. Only the Data Engineer reads it. This will grow linearly. Time-Travel: Supported by keeping files in folders labeled by date


-- Clean Layer --
   Parquet is "columnar." If we only want to calculate the sum of UnitPrice, the system doesn't have to read Description or Country, making it 10x faster than CSV. Data Analysts read it using SQL (DuckDB/BigQuery) or Power BI. Growth is medium over time. Parquet supports "schema evolution" and versioning, allowing analysts to see how the data looked last month


-- Feature Layer -- 
   We use Parquet or a SQL Table. This layer is "flat" and wide. Each row is a CustomerID with all their calculated stats. Data scientists read it using ML models. Growth is small over time. Time-travel is enabled in the Feature Layer to prevent data leakage. 



### 3.3
Tracking: We utilize a High-Water Mark based on InvoiceDate to identify new records.

Late Data: A 3-day look-back window is applied to catch delayed entries, with a Composite Key deduplication step to maintain data integrity.

Feature Refresh: Customer-level features are updated Daily via an overwrite of the active partition, with Monthly Snapshots preserved for model training.