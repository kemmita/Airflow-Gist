```py
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.dummy import DummyOperator
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.utils.task_group import TaskGroup

from random import uniform
from datetime import datetime

default_args = {
    'start_date': datetime(2020, 1, 1)
}

def _is_accurate(ti):
    max_accuracy = ti.xcom_pull(key='max_accuracy',task_ids=['choose_model'])
    if max(max_accuracy) > 18:
        return('accurate')
    return('inaccurate')        

with DAG('xcom_dag', schedule_interval='@daily', default_args=default_args, catchup=False) as dag:

    is_accurate = BranchPythonOperator(
        task_id='is_accurate',
        python_callable=_is_accurate
    )

    accurate = DummyOperator(
        task_id='accurate'
    )

    inaccurate = DummyOperator(
        task_id='inaccurate'
    )
    
    # Below on line 43, you will see that storing is to take place after accurate or inaccurate and those are conditional tasks so only one will be executed
    # In order for stroing to execute it must specify a trigger rule,accurate or inaccurate will be skipped based on a condition, storing will watch for that and then execute.
    storing = DummyOperator(
        task_id='storing',
        trigger_rule='none_failed_or_skipped'
    )

    downloading_data >> processing_tasks >> choose_model 
    choose_model >> is_accurate >> [accurate, inaccurate] >> storing
```
