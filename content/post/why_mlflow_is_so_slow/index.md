---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Why MLflow is so slow"
subtitle: ""
summary: "and how to speed id up!"
authors: ["kirill-vasin"]
tags: []
categories: []
date: 2020-10-31
lastmod: 2020-10-31
featured: true
draft: false
toc: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
[MLflow]("https://www.mlflow.org/) — "open source platform for managing the end-to-end machine learning lifecycle". It offers a lot of cool features, and one of them is an ability to track parameters and metrics of your experiments, save them to a remote storage and get some visualizations via Graphic UI. In this post I will tell you how I figured out what is wrong with MLflow Tracking performance and how I managed to improve it. If you are only interested in the end result — jump straight to [Final solution](#final-solution) section! Moreover, you can find the code used here in a [google colab](https://colab.research.google.com/drive/1Y0g_m_4x4wp5i6fuS9D4Vu0RQ3eWQ2fR#scrollTo=HSVZUaptTE5H).

## Warning!
In this article I'm breaking some fail-safes set by developers of MLflow to achieve a desired performance. Do not use the code below in production if you do not fully understand what you are doing!

In addition, I should notice that most probably, you will not encounter the same problem if you are using a file store backend. Hopefully, MLflow will fix this issue in a future releases. There is an [open issue](https://github.com/mlflow/mlflow/issues/3010) on Github you can follow and contribute to. Right now, in October 2020, I'm using a release `1.11.0`.

## MLflow or MLslow
One day my colleagues and I decided it would be a brilliant idea to add MLflow to our ML Pipeline. We prepared a PostgreSQL database, installed MLflow and started logging results of our experiments with a handy `log_params` and `log_metrics` functions. 
```python
from time import time
import mlflow

mlflow.set_tracking_uri(DATABASE_URI)
experiment_name = "testing_mlflow"
run_name = "initial_run"
mlflow.set_experiment(experiment_name)
metrics = {f"metric_{i}": 1 / (i + 1) for i in range(12)}
params = {f"param_{i}": f"param_{i}" for i in range(3)}

start = time()
with mlflow.start_run(run_name=run_name):
    mlflow.log_metrics(metrics)
    mlflow.log_params(params)
print(f"Took {time()- start:.2f} seconds")
```
After conducting several experiments I noticed something is terribly wrong. The code inside `with` block took outrageous **53 seconds** to execute! So, I launched my investigation.

## Forgetful MLflow
As a law-abiding citizen I began with mlflow official documentation. Nothing was really helpful there, yet I've learned that MLflow uses SQLAlchemy to manage database backend. Changing environmental variable `MLFLOW_SQLALCHEMYSTORE_POOL_SIZE` mentioned in the docs to 10 (from default 5) provided me with a following output:
```
2020/10/31 20:39:39 INFO mlflow.store.db.utils:
Create SQLAlchemy engine with pool options {'pool_size': 10}
2020/10/31 20:39:47 INFO mlflow.store.db.utils:
Create SQLAlchemy engine with pool options {'pool_size': 10}
2020/10/31 20:39:56 INFO mlflow.store.db.utils:
Create SQLAlchemy engine with pool options {'pool_size': 10}
2020/10/31 20:40:26 INFO mlflow.store.db.utils:
Create SQLAlchemy engine with pool options {'pool_size': 10}
2020/10/31 20:40:37 INFO mlflow.store.db.utils:
Create SQLAlchemy engine with pool options {'pool_size': 10}
```
It seems that MLflow creates a new SQLAlchemy engine object each time you call MLflow in your code. Maybe that is why everything is so slow.
{{< figure src="forgetful_mlflow.png">}}

## Not so impressive enhancement
A brief look in the source code led me to an object `mlflow.tracking.client import MlflowClient` which is used underneath the main `mlflow` interface. Hence, I created a solution where only one instance of MlflowClient was used.
```python
from time import time

from mlflow.entities import Metric, Param
from mlflow.tracking.client import MlflowClient
from mlflow.tracking.context import (
    registry as context_registry,
)

mlflow_client = MlflowClient(tracking_uri=DATABASE_URI)
experiment_name = "client_run"
try:
    experiment_id = str(
        mlflow_client.get_experiment_by_name(
            experiment_name
        ).experiment_id
    )
except AttributeError:
    experiment_id = mlflow_client.create_experiment(
        experiment_name
    )

run_name = "naive_approach"
tags = context_registry.resolve_tags(
    {"mlflow.runName": run_name}
)
run = mlflow_client.create_run(
    experiment_id=experiment_id, tags=tags
)

metrics = {f"metric_{i}": 1 / (i + 1) for i in range(12)}
params = {f"param_{i}": f"param_{i}" for i in range(3)}
prep_metrics = [
    Metric(k, v, timestamp=int(time() * 1000), step=0)
    for k, v in metrics.items()
]
prep_params = [Param(k, str(v)) for k, v in metrics.items()]

start = time()
mlflow_client.log_batch(
    run_id=run.info.run_id,
    metrics=prep_metrics,
    params=prep_params,
)
mlflow_client.set_terminated(run.info.run_id)
print(f"Took {time()- start:.2f} seconds")
```
Let's look into a console output:
```
2020/10/31 21:48:07 INFO mlflow.store.db.utils:
Create SQLAlchemy engine with pool options {'pool_size': 3}
```
What a fantastic success! I managed to reduce a number of creations of `SQLAlchemy engine` objects just to 1! How much time this code took to execute?

**37 seconds**

Ok, maybe the success was not so fantastic. Seems I could use some profiling here.

## An ugly truth behing log_batch
With a help of cProfiler I managed to get a better understanding of what was going on.
```
ncalls: 164
tottime: 37.875
percall: 0.231
cumtime: 37.880
filename:lineno(function): {method 'execute' of
    'psycopg2.extensions.cursor' objects}

```
**164 calls to method 'execute'!!!** Wait, we are sending metrics and params to the database via method called `log_batch`. What the hell is going on? What is hidden behind `log_batch` method??? Let's take a look at [source code](https://github.com/mlflow/mlflow/blob/37496c3161a9f7561b4c7ca3054ae6bdb3154523/mlflow/store/tracking/sqlalchemy_store.py#L738) of a connector to SQLAlchemy from MLflow repository:
```python
def log_batch(self, run_id, metrics, params, tags):
  _validate_run_id(run_id)
  _validate_batch_log_data(metrics, params, tags)
  _validate_batch_log_limits(metrics, params, tags)
  with self.ManagedSessionMaker() as session:
      run = self._get_run(run_uuid=run_id, session=session)
      self._check_run_is_active(run)
  try:
      for param in params:
          self.log_param(run_id, param)
      for metric in metrics:
          self.log_metric(run_id, metric)
      for tag in tags:
          self.set_tag(run_id, tag)
  except MlflowException as e:
      raise e
  except Exception as e:
      raise MlflowException(e, INTERNAL_ERROR)
```
Lol, nice for-loops out there!
{{< figure src="mlflow_batch_request.jpg">}}

## Final solution
To avoid calling `execute` too often, I used SQLAlchemy session directly. 
```python
from time import time

from mlflow.store.tracking.dbmodels.models import (
    SqlLatestMetric,
    SqlParam,
)
from mlflow.tracking.client import MlflowClient
from mlflow.tracking.context import (
    registry as context_registry,
)

mlflow_client = MlflowClient(tracking_uri=DATABASE_URI)
experiment_name = "testing_mlflow"
try:
    experiment_id = str(
        mlflow_client.get_experiment_by_name(
            experiment_name
        ).experiment_id
    )
except AttributeError:
    experiment_id = mlflow_client.create_experiment(
        experiment_name
    )

run_name = "final_solution"
tags = context_registry.resolve_tags(
    {"mlflow.runName": run_name}
)
run = mlflow_client.create_run(
    experiment_id=experiment_id, tags=tags
)

metrics = {f"metric_{i}": 1 / (i + 1) for i in range(12)}
params = {f"param_{i}": f"param_{i}" for i in range(3)}
prep_metrics = [
    SqlLatestMetric(
        **{
            "key": key,
            "value": value,
            "run_uuid": run.info.run_id,
            "timestamp": int(time() * 1000),
            "step": 0,
            "is_nan": False,
        }
    )
    for key, value in metrics.items()
]
prep_params = [
    SqlParam(
        **{
            "key": key,
            "value": str(value),
            "run_uuid": run.info.run_id,
        }
    )
    for key, value in params.items()
]

start = time()
store = mlflow_client._tracking_client.store
with store.ManagedSessionMaker() as session:
    session.add_all(prep_params + prep_metrics)
mlflow_client.set_terminated(run.info.run_id)
print(f"Took {time()- start:.2f} seconds")
```
Final time: **3.71 seconds** which seem more reasonable. Yet, the code above does not optimize a creation of experiment and run entities, so you would get the more considerable boost in performance the bigger the size of metrics and params dicts.

Just in case I will duplicate a warning: **code above should not be used in production if you do not know what you are doing**. I purposefully turned off some of the MLflow validators and transformations like handling NaNs among metrics values to make this post more readable.

If you like the content above, please, consider subscribing to my Telegram channel: [@FuriousAI](https://t.me/FuriousAI). Also, I would be happy to receive any feedback: fill free to drop a message to me via [LinkedIn](https://www.linkedin.com/in/vasinkd/), [Telegram](https://t.me/vasinkd) or [email](mailto:vasin.kd@gmail.com).