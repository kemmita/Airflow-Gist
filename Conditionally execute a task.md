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

def _training_model(ti):
    accuracy = uniform(0.1, 10.0)
    print(f'model\'s accuracy: {accuracy}')
    ti.xcom_push(key='model_accuracy', value=accuracy)

def _choose_best_model(ti):
    accuracies = ti.xcom_pull(key='model_accuracy',task_ids=[
        'processing_tasks.training_model_a',
        'processing_tasks.training_model_b',
        'processing_tasks.training_model_c'
    ])
    ti.xcom_push(key='max_accuracy',value=max(accuracies))

# If the max accuracy is > 18 we will run task name accurate else inaccurate.
def _is_accurate(ti):
    max_accuracy = ti.xcom_pull(key='max_accuracy',task_ids=['choose_model'])
    if max(max_accuracy) > 18:
        return('accurate')
    return('inaccurate')        

with DAG('xcom_dag', schedule_interval='@daily', default_args=default_args, catchup=False) as dag:

    downloading_data = BashOperator(
        task_id='downloading_data',
        bash_command='sleep 3',
        do_xcom_push=False
    )

    with TaskGroup('processing_tasks') as processing_tasks:
        training_model_a = PythonOperator(
            task_id='training_model_a',
            python_callable=_training_model
        )

        training_model_b = PythonOperator(
            task_id='training_model_b',
            python_callable=_training_model
        )

        training_model_c = PythonOperator(
            task_id='training_model_c',
            python_callable=_training_model
        )

    choose_model = PythonOperator(
        task_id='choose_model',
        python_callable=_choose_best_model
    )
    
    # We will need to use a BranchPythonOperator that will call a python method that will return the name of the task we want to execute.
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
                                                            # Below we specfiy the branch operator and wrap the tasks to be ran conditionaly in a parallel block
    downloading_data >> processing_tasks >> choose_model >> is_accurate >> [accurate, inaccurate]
```
