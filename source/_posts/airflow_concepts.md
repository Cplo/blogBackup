---
title: Airflow概念
categories:
- Workflow
tags:
- airflow
- 翻译
---
### 触发器规则
虽然正常的工作流是在任务的所有直接上游任务都成功时触发任务，但Airflow允许进行更复杂的依赖关系设置。 

所有的operators都有一个***trigger_rule***参数，它定义了生成的任务被触发的规则。 trigger_rule参数的默认值是all_success，定义为“当所有直接上游任务成功执行完成时触发此任务”。这里描述的所有其他规则都基于直接的父任务，并且这些值可以在创建任务时传递给任何operator：

- all_success: 所有父任务被成功执行（默认）
- all_failed：所有父任务处于failed 或者 upstream_failed 状态
- all_done：所有的父任务都完成了他们的执行
- one_failed：当至少有一位父任务失败后会触发，不会等待所有父任务完成
- one_success：当至少有一位父任务成功后会触发，并不等待所有的父任务完成
- dummy：依赖关系仅用于显示，随意触发


请注意，这些可以与depends_on_past（boolean）一起使用，当设置为True时，如果任务的上一个schedule计划未成功，则会阻止任务被触发。

### 仅运行最新的schedule

标准工作流程行为针对特定日期/时间范围运行一系列任务。但是，某些工作流执行独立于运行时间的任务，但需要按计划运行，与标准cron作业非常相似。这些情况下，在暂停期间回填或运行未命中的作业只会浪费CPU周期。

对于这种情况，您可以使用LatestOnlyOperator跳过在DAG的不在最近一次计划运行期间运行的任务。如果现在的时间不在其执行时间和下一个计划的执行时间之间，LatestOnlyOperator将跳过所有即时下游任务及其本身。

必须注意跳过的任务和触发规则之间的交互。任务跳过将传递通过这些触发规则：all_success和all_failed，all_done，one_failed，one_success和dummy不传递。如果您想使用带有不会传递跳转的触发器规则的LatestOnlyOperator，则需要确保LatestOnlyOperator直接位于您要跳过的任务的上游。

通过使用触发规则可以混合以典型的日期/时间依赖模式运行的任务和使用LatestOnlyOperator的任务。

例如，请考虑以下dag：
```
#dags/latest_only_with_trigger.py
import datetime as dt

from airflow.models import DAG
from airflow.operators.dummy_operator import DummyOperator
from airflow.operators.latest_only_operator import LatestOnlyOperator
from airflow.utils.trigger_rule import TriggerRule


dag = DAG(
    dag_id='latest_only_with_trigger',
    schedule_interval=dt.timedelta(hours=4),
    start_date=dt.datetime(2016, 9, 20),
)

latest_only = LatestOnlyOperator(task_id='latest_only', dag=dag)

task1 = DummyOperator(task_id='task1', dag=dag)
task1.set_upstream(latest_only)

task2 = DummyOperator(task_id='task2', dag=dag)

task3 = DummyOperator(task_id='task3', dag=dag)
task3.set_upstream([task1, task2])

task4 = DummyOperator(task_id='task4', dag=dag,
                      trigger_rule=TriggerRule.ALL_DONE)
task4.set_upstream([task1, task2])
```

在这个dag的情况下，latest_only任务将把除最近运行以外的所有运行显示为跳过的状态。 task1直接位于latest_only的下游，并且也会跳过除最新版本之外的所有运行。 task2完全独立于latest_only，并将在所有预定时段运行。 task3位于task1和task2的下游，并且由于默认的trigger_rule为all_success，因此将从task1接收传递跳过。 task4位于task1和task2的下游，但由于其trigger_rule设置为all_done，只要task1被跳过（一个有效的完成状态）并且task2成功，它就会立即触发。

![image](
https://airflow.apache.org/_images/latest_only_with_trigger.png)

[原文链接](https://airflow.apache.org/concepts.html)
