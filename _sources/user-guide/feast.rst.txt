=====
Feast
=====


Introduction to Feast
=====================

Feast (Feature Store) is a customizable operational data system that re-uses existing infrastructure to manage and serve machine learning features to models.

* **Feature sharing and reuse**: Engineering features is one of the most time consuming activities in building an end-to-end ML system, yet many teams continue to develop features in silos. This leads to a high amount of re-development and duplication of work across teams and projects.

* **Serving features at scale**: Models need data that can come from a variety of sources, including event streams, data lakes, warehouses, or notebooks. ML teams need to be able to store and serve all these data sources to their models in a performant and reliable way. The challenge is scalably producing massive datasets of features for model training, and providing access to real-time feature data at low latency and high throughput in serving.

* **Consistency between training and serving**: The separation between data scientists and engineering teams often lead to the re-development of feature transformations when moving from training to online serving. Inconsistencies that arise due to discrepancies between training and serving implementations frequently leads to a drop in model performance in production.


Prerequisites
=============

* You have already installed Kubeflow on your cluster (full or lite version).

* You have installed `MinIO <https://min.io/docs/minio/kubernetes/upstream/index.html>`_  operator.

* You can access the kubeflow dashboard.

* You Have created Kubeflow Notebook refer to :ref:`user-guide-notebooks`.


Getting started with Feast
==========================

This guide provides the necessary resources to install Feast alongside Kubeflow, describes the usage of Feast with Kubeflow components, and provides examples that users can follow to test their setup.

Grab the `code examples <https://github.com/AmyHoney/feast-example/blob/master/03_feature_repo_s3_offline_sqlite_online/s3_online_explore_date.ipynb>`_ to know feast and use it: 

* Setting up a feature repo using `MinIO <https://min.io/docs/minio/kubernetes/upstream/index.html>`_ S3 bucket about registry for offline features to train model.

* Make predictions using online features by `SQLite <https://docs.feast.dev/reference/online-stores/sqlite>`_.


----------------
Installing Feast
----------------

Before we get started, first install Feast and others' dependencies:

.. code-block:: shell

    pip install feast==0.29.0
    pip install scikit-learn
    pip install boto3
    pip install s3fs


------------------
Exploring the data
------------------

We've made some dummy data for this workshop in `infra/driver_stats.parquet <https://github.com/AmyHoney/feast-example/blob/master/01_feature_repo_local/infra/driver_stats.parquet>`_. Let's dive into what the data looks like.

.. code-block:: shell

    import pandas as pd
    pd.read_parquet("infra/driver_stats.parquet")


.. image:: ../_static/user-guide-feast-data.png

This is a set of time-series data with driver_id as the primary key (representing the driver entity) and event_timestamp as showing when the event happened.

-----------------------------------------------------------
Access MinIO and setup bucket for registry and file sources
-----------------------------------------------------------

Firstly you need to get MinIO access and secret key for authentication.

.. code-block:: shell

    # get the Minio secret name
    $ microk8s kubectl get secret -n admin | grep minio

    # get the access key for MinIO
    $ microk8s kubectl get secret <minio-secret-name> -n admin -o jsonpath='{.data.accesskey}' | base64 -d

    # get the secret key for MinIO
    $ microk8s kubectl get secret <minio-secret-name> -n admin -o jsonpath='{.data.secretkey}' | base64 -d

Secondly update the MinIO parameters on your environment, create a bucket for feast and upload data files to bucket.

.. code-block:: shell

    import os
    from urllib.parse import urlparse
    import boto3

    # Update these parameters about your environment
    os.environ["FEAST_S3_ENDPOINT_URL"] = "http://minio.kubeflow:9000"
    os.environ["AWS_ACCESS_KEY_ID"] = "minio"
    os.environ["AWS_SECRET_ACCESS_KEY"] = "RL4YSFL8TT39BBHZO7SKNYOG6Y5TDD"

    s3 = boto3.resource('s3',
                        endpoint_url=os.getenv("MLFLOW_S3_ENDPOINT_URL"),
                        verify=False)

    # Create a bucket
    bucket_name='featurestore'
    s3.create_bucket(Bucket=bucket_name)

    # Check if the newly bucket exists
    print(list(s3.buckets.all()))

    # Upload data file to the newly bucket
    bucket = s3.Bucket(bucket_name)
    bucket_path = "infra"
    bucket.upload_file("infra/driver_stats.parquet", os.path.join(bucket_path, "driver_stats.parquet"))

    # check files
    for obj in bucket.objects.filter(Prefix=bucket_path):
        print(obj.key)

----------------------
Setup the feature repo
----------------------

The first thing needs to do is setup a ``feature_store.yaml`` file, and it is the primary way to configure an overall Feast project. We have setup a sample feature reposity in `03_feature_repo_s3_offline_sqlite_online <https://github.com/AmyHoney/feast-example/tree/master/03_feature_repo_s3_offline_sqlite_online>`_

^^^^^^^^^^^^^^^^^^^^^^^^^^
Use your configured bucket
^^^^^^^^^^^^^^^^^^^^^^^^^^

**data_sources.py**

.. code-block:: shell

    from feast import FileSource
    import s3fs

    bucket_name = "featurestore"
    file_name = "driver_stats.parquet"
    s3_endpoint = "http://minio.kubeflow:9000"

    s3 = s3fs.S3FileSystem(key='minio',
                        secret='RL4YSFL8TT39BBHZO7SKNYOG6Y5TDD',
                        client_kwargs={'endpoint_url': s3_endpoint}, use_ssl=False)

    driver_stats = FileSource(
        name="driver_stats_source",
        path="s3://featurestore/infra/driver_stats.parquet",  # TODO: Replace with your bucket
        s3_endpoint_override="http://minio.kubeflow:9000", # TODO: Replace with your MinIO URL
        timestamp_field="event_timestamp",
        created_timestamp_column="created",
        description="A table describing the stats of a driver based on hourly logs",
        owner="test2@gmail.com",
    )

**feature_store.yaml**

.. code-block:: shell

  project: feast_demo_minio
  provider: local
  registry: s3://featurestore/infra/registry.pb # TODO: Replace with your bucket
  online_store:
    type: sqlite
    path: data/online_store.db
  offline_store:
    type: file
  entity_key_serialization_version: 2


A quick explanation of what's happening in this ``feature_store.yaml``:

================== =====================================================================================  ========================================================================================================
    Key              What it does                                                                          Example                                                                                                  
================== =====================================================================================  ========================================================================================================
`project`          Gives infrastructure isolation via namespacing (e.g. online stores + Feast objects).   any unique name within your organization (e.g. `feast_demo_minio`)                                         
`provider`         Defines registry location & sets defaults for offline / online stores                  `local`, `aws`, `gcp` (MinIO has S3 object and installed on Kubeflow)
`registry`         Defines the specific path for the registry (local, gcs, s3, etc)                       `s3://[YOUR BUCKET]/registry.pb`                                                                         
`online_store`     Configures online store (if needed for supporting real-time models)                    `null`, `sqlite`, `redis`, `dynamodb`, `datastore`, `postgres` (each have their own extra configs)        
`offline_store`    Configures offline store, which executes point in time joins                           `bigquery`, `snowflake.offline`,  `redshift`, `spark`, `trino`  (each have their own extra configs)      
================== =====================================================================================  ========================================================================================================


^^^^^^^^^^^^^^^^^^^
Run ``feast plan``
^^^^^^^^^^^^^^^^^^^

With the ``feature_store.yaml`` setup, you can now run ``feast plan`` to see what changes would happen with ``feast apply``.

.. code-block:: shell

    feast plan

Sample output:

.. code-block:: shell

    02/22/2023 02:48:14 AM botocore.credentials INFO: Found credentials in environment variables.
    Created entity driver
    Created feature view driver_hourly_stats
    Created feature service model_v1
    Created feature service model_v2

    Created sqlite table feast_demo_minio_driver_hourly_stats

^^^^^^^^^^^^^^^^^^^^
Run ``feast apply``
^^^^^^^^^^^^^^^^^^^^

This will parse the feature, data source, and feature service definitions and publish them to the registry. It may also setup some tables in the online store to materialize batch features.

.. code-block:: shell

    feast apply

    02/22/2023 02:48:14 AM botocore.credentials INFO: Found credentials in environment variables.
    Created entity driver
    Created feature view driver_hourly_stats
    Created feature service model_v1
    Created feature service model_v2

    Deploying infrastructure for feast_demo_minio_driver_hourly_stats


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Verify features are registered
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can now run Feast CLI commands to verify Feast knows about your features and data sources.

.. code-block:: shell

    feast feature-views list

    02/22/2023 02:48:43 AM botocore.credentials INFO: Found credentials in environment variables.
    NAME                 ENTITIES    TYPE
    driver_hourly_stats  {'driver'}  FeatureView


---------------------------------------------
Fetch offline features from S3 to train model
---------------------------------------------

``get_historical_features`` is an API by which you can retrieve features (by referencing features directly or via feature services). It will under the hood manage point-in-time joins and avoid data leakage to generate training datasets.

Fetch feature from `driver_orders.csv <https://github.com/AmyHoney/feast-example/blob/master/03_feature_repo_s3_offline_sqlite_online/driver_orders.csv>`_ data using ``get_historical_features`` API to train model.

.. code-block:: shell

  import feast
  from joblib import dump
  import pandas as pd
  from sklearn.linear_model import LinearRegression

  # Load driver order data, when orders give to entity_df, it shows 0 entries  
  orders = pd.read_csv("driver_orders.csv", sep="\t")
  orders["event_timestamp"] = pd.to_datetime(orders["event_timestamp"])
  print(orders)

  # Connect to your feature store provider
  store = FeatureStore(repo_path="./feature_repo_minio_online")

  # Because we're using the default FileOfflineStore, this executes on your machine
  training_df = store.get_historical_features(
      entity_df=orders, 
      features=store.get_feature_service("model_v2"),
  ).to_df()

  print("----- Feature schema -----\n")
  print(training_df.info())

  print()
  print("----- Example features -----\n")
  print(training_df.head())

  # Train model
  target = "trip_completed"

  reg = LinearRegression()
  train_X = training_df[training_df.columns.drop(target).drop("event_timestamp")]
  train_Y = training_df.loc[:, target]
  reg.fit(train_X[sorted(train_X)], train_Y)

  # Save model
  dump(reg, "driver_model.bin")

The output is like this, and the trained model file "driver_model.bin" is save as the current directory.

.. code-block:: shell

              event_timestamp  driver_id  trip_completed
  0 2021-04-16 20:29:28+00:00       1001               1
  1 2021-04-17 04:29:28+00:00       1002               0
  2 2021-04-17 12:29:28+00:00       1003               0
  3 2021-04-17 20:29:28+00:00       1001               1
  4 2021-04-18 04:29:28+00:00       1002               0
  5 2021-04-18 12:29:28+00:00       1003               0
  6 2021-04-18 20:29:28+00:00       1001               1
  7 2021-04-19 04:29:28+00:00       1002               0
  8 2021-04-19 12:29:28+00:00       1003               0
  9 2021-04-19 20:29:28+00:00       1004               1
  ----- Feature schema -----

  <class 'pandas.core.frame.DataFrame'>
  RangeIndex: 10 entries, 0 to 9
  Data columns (total 5 columns):
  #   Column           Non-Null Count  Dtype              
  ---  ------           --------------  -----              
  0   event_timestamp  10 non-null     datetime64[ns, UTC]
  1   driver_id        10 non-null     int64              
  2   trip_completed   10 non-null     int64              
  3   conv_rate        10 non-null     float32            
  4   acc_rate         10 non-null     float32            
  dtypes: datetime64[ns, UTC](1), float32(2), int64(2)
  memory usage: 448.0 bytes
  None

  ----- Example features -----

              event_timestamp  driver_id  trip_completed  conv_rate  acc_rate
  0 2021-04-16 20:29:28+00:00       1001               1   0.521149  0.751659
  1 2021-04-17 04:29:28+00:00       1002               0   0.089014  0.212637
  2 2021-04-17 12:29:28+00:00       1003               0   0.188855  0.344736
  3 2021-04-17 20:29:28+00:00       1001               1   0.521149  0.751659
  4 2021-04-18 04:29:28+00:00       1002               0   0.089014  0.212637

  ['driver_model.bin']


--------------------------------------------
Fetch online features from SQLite to predict
--------------------------------------------

First we materialize features (which generate the latest values for each entity key from batch sources) into the online store (sqlite).

.. code-block:: shell

    feast materialize-incremental $(date +%Y-%m-%d)

Now we can retrieve these materialized features from SQLite by directly using the SDK, load the trained model file before, to make prediction.

.. code-block:: shell

  import pandas as pd
  import feast
  from joblib import load


  class DriverRankingModel:
      def __init__(self):
          # Load model
          self.model = load("driver_model.bin")

          # Set up feature store
          self.fs = feast.FeatureStore(repo_path="./feature_repo_minio_online")

      def predict(self, driver_ids):
          # Read features from Feast
          driver_features = self.fs.get_online_features(
              entity_rows=[{"driver_id": driver_id} for driver_id in driver_ids],
              features=[
                  "driver_hourly_stats:conv_rate",
                  "driver_hourly_stats:acc_rate",
              ],
          )
          df = pd.DataFrame.from_dict(driver_features.to_dict())

          # Make prediction
          df["prediction"] = self.model.predict(df[sorted(df)])

          # Choose best driver
          best_driver_id = df["driver_id"].iloc[df["prediction"].argmax()]

          # return best driver
          return best_driver_id
     
  def make_drivers_prediction():
      drivers = [1001, 1002, 1003, 1004]
      model = DriverRankingModel()
      best_driver = model.predict(drivers)
      print(f"Prediction for best driver id: {best_driver}")
      
  make_drivers_prediction() 

The result output is ``Prediction for best driver id: 1003``

.. seealso::

    `Feast quickstart <https://docs.feast.dev/getting-started/quickstart>`__

    `Feature Store on Kubeflow <https://www.kubeflow.org/docs/external-add-ons/feature-store/>`__

    `Workshop: Learning Feast <https://github.com/feast-dev/feast-workshop>`__

    `Feast Driver Ranking Example <https://github.com/juskuz/feast-driver-ranking-demo-aitech>`__