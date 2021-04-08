```py
from airflow import DAG
from airflow.contrib.operators.spark_submit_operator import SparkSubmitOperator
from airflow.contrib.sensors.file_sensor import FileSensor
from airflow.operators.bash_operator import BashOperator 
from airflow.operators.hive_operator import HiveOperator 
from airflow.operators.python_operator import PythonOperator 
from airflow.sensors.http_sensor import HttpSensor
from datetime import datetime, timedelta
import json
import csv
import requests

default_args = {
   'start_date': datetime(2021,3,11),
   'owner': 'airflow',
   'depends_on_past': False,
   'email_on_failure': False,
   'email_on_retry': False,
   'email': 'test@test.com',
   'retries': 1,
   'retry_delay': timedelta(minutes=5)
}

def download_rates():
    with open('/usr/local/airflow/dags/files/forex_currencies.csv') as forex_currencies:
        reader = csv.DictReader(forex_currencies, delimiter=';')
        for row in reader:
            base = row['base']
            with_pairs = row['with_pairs'].split(' ')
            indata = requests.get('http://api.exchangeratesapi.io/latest?access_key=0093325565040a68334b11388d04f7d1&base=' + base).json()
            outdata = {'base': base, 'rates': {}, 'last_update': indata['date']}
            for pair in with_pairs:
                outdata['rates'][pair] = indata['rates'][pair]
            with open('/usr/local/airflow/dags/files/forex_rates.json', 'a') as outfile:
                json.dump(outdata, outfile)
                outfile.write('\n')

with DAG(dag_id='forex_dag_2',schedule_interval='@daily',default_args=default_args, catchup=False) as dag:
    is_forex_rates_available = HttpSensor(
        task_id='is_forex_rates_available',
        method='GET',
        http_conn_id='forex_api',
        endpoint='latest',
        request_params={'access_key':'0093325565040a68334b11388d04f7d1','symbols':'USD'},
        response_check=lambda response: "rates" in response.text,
        poke_interval=3,
        timeout=20
    )

    is_forex_file_available = FileSensor(
        task_id='is_forex_file_available',
        fs_conn_id='forex_path',
        filepath='forex_currencies.csv',
        poke_interval=3,
        timeout=20
    )

    download_forex_rates = PythonOperator(
        task_id='download_forex_rates',
        python_callable=download_rates
    )

    save_rates = BashOperator(
        task_id='save_rates',
        bash_command="""
            hdfs dfs -mkdir -p /forex && \
            hdfs dfs -put -f $AIRFLOW_HOME/dags/files/forex_rates.json /forex
            """
    )

    creating_forex_rates_table = HiveOperator(
        task_id="creating_forex_rates_table",
        hive_cli_conn_id="hive_conn",
        hql="""
            CREATE EXTERNAL TABLE IF NOT EXISTS forex_rates(
                base STRING,
                usd DOUBLE,
                nzd DOUBLE,
                jpy DOUBLE,
                gbp DOUBLE,
                cad DOUBLE,
                last_update DATE
                )
            ROW FORMAT DELIMITED
            FIELDS TERMINATED BY ','
            STORED AS TEXTFILE
        """
    )

    forex_processing = SparkSubmitOperator(
        task_id="forex_processing",
        conn_id="spark_wrap",
        application="/usr/local/airflow/dags/scripts/forex_processing.py",
        verbose=False
    )

    is_forex_rates_available >> is_forex_file_available >> download_forex_rates >> save_rates >> creating_forex_rates_table >> forex_processing
```
