Real-time Stock Market Data Pipeline

A real-time stock market data pipeline ingests live stock prices, processes them, stores them in a database, and makes them available for analytics and visualization.


---

Project Description

This project captures stock market data from an API, processes it using Apache Kafka and PySpark, stores it in Amazon Redshift, and visualizes trends using a BI tool.

Tech Stack:

Data Ingestion: Alpha Vantage / Yahoo Finance API

Streaming Platform: Apache Kafka

Processing: PySpark (Structured Streaming)

Storage: Amazon Redshift / PostgreSQL

Orchestration: Apache Airflow

Visualization: Tableau / Power BI



---

Features

✔ Real-time data ingestion – Fetch stock prices from an API every second/minute.
✔ Scalable architecture – Uses Kafka for handling high-frequency data.
✔ ETL processing – Cleans, transforms, and aggregates stock data.
✔ Data storage – Stores processed data in Amazon Redshift.
✔ Data visualization – Dashboards to monitor stock trends.
✔ Automation – Airflow schedules and monitors data pipeline jobs.


---

Installation Steps

1. Set Up Kafka & Zookeeper

# Start Zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties  

# Start Kafka Server
bin/kafka-server-start.sh config/server.properties  

# Create Kafka Topic for Stock Data
bin/kafka-topics.sh --create --topic stock_prices --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1

2. Install Dependencies

pip install kafka-python requests pyspark psycopg2 apache-airflow

3. Start Stock Data Producer

Create stock_data_producer.py:

from kafka import KafkaProducer
import requests
import json
import time

producer = KafkaProducer(bootstrap_servers='localhost:9092', value_serializer=lambda v: json.dumps(v).encode('utf-8'))
api_url = "https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=AAPL&interval=1min&apikey=YOUR_API_KEY"

while True:
    response = requests.get(api_url).json()
    stock_data = response.get("Time Series (1min)", {})
    for timestamp, data in stock_data.items():
        record = {"timestamp": timestamp, "stock": "AAPL", "price": float(data["1. open"])}
        producer.send('stock_prices', record)
    time.sleep(60)  # Fetch data every minute

4. Create PySpark Consumer & Transformation

Create stock_data_processor.py:

from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StringType, FloatType

spark = SparkSession.builder.appName("StockProcessor").getOrCreate()
schema = StructType().add("timestamp", StringType()).add("stock", StringType()).add("price", FloatType())

df = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "localhost:9092")\
    .option("subscribe", "stock_prices").load()

stock_df = df.selectExpr("CAST(value AS STRING)").select(from_json(col("value"), schema).alias("data")).select("data.*")

query = stock_df.writeStream.format("console").start()
query.awaitTermination()

5. Load Data into Redshift

Configure Airflow to run a DAG (stock_pipeline.py) to load data into Redshift:

from airflow import DAG
from airflow.providers.amazon.aws.transfers.postgres_to_s3 import PostgresToS3Operator
from airflow.providers.amazon.aws.operators.redshift import RedshiftSQLOperator
from datetime import datetime

dag = DAG('stock_pipeline', start_date=datetime(2025, 3, 21), schedule_interval='@hourly')

upload_to_redshift = RedshiftSQLOperator(
    task_id='load_stock_data',
    sql="COPY stock_data FROM 's3://your-bucket/stock_data.csv' IAM_ROLE 'your-redshift-role' CSV;",
    dag=dag
)

6. Run Airflow Scheduler

airflow scheduler
airflow webserver

7. Visualize Data in Tableau/Power BI

Connect to Redshift

Create a dashboard showing stock price trends



---

Done! 

This pipeline continuously fetches, processes, and stores stock data, enabling real-time analytics. Let me know if you need modifications!
