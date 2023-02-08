==================
ML Metadata (MLMD)
==================

--------
Overview
--------

Metadata is the data that includes information about the context of other data and is generated in each phase of the ML lifecycle. 
From the data extraction to the model monitoring phase, all ML-related processes create specific metadata. For example, the 
initial part of the ML lifecycle includes metadata regarding the location, name, and version of the dataset; the model training 
phase includes metadata regarding hyperparameters and evaluation metrics.  

`ML Metadata (MLMD) <https://github.com/google/ml-metadata>`__ is a library for recording and retrieving metadata associated with 
ML developer and data scientist workflows. MLMD is an integral part of `TensorFlow Extended (TFX) <https://www.tensorflow.org/tfx>`__, 
but is designed so that it can be used independently.

Every run of a production ML pipeline generates metadata containing information about the various pipeline components, their 
executions (e.g. training runs), and resulting artifacts (e.g. trained models). In the event of unexpected pipeline behavior or 
errors, this metadata can be leveraged to analyze the lineage of pipeline components and debug issues. Think of this metadata as 
the equivalent of logs in software development.

MLMD helps you understand and analyze all the interconnected parts of your ML pipeline instead of analyzing them in isolation and 
can help you answer questions about your ML pipeline such as:

* Which dataset did the model train on?
* What were the hyperparameters used to train the model?
* Which pipeline run created the model?
* Which training run led to this model?
* When was the failed model pushed?

Below graphic shows a high-level overview of the various components that are part of MLMD.

    .. image:: ../_static/user-guide-mlmd-overview.png

--------------
Metadata Store
--------------

^^^^^^^^^^^^^^^^^^^^^^^
What is Metadata Store?
^^^^^^^^^^^^^^^^^^^^^^^

An ML metadata store is a centralized structure for storing the metadata of ML models. MLMD registers the following types of 
metadata in the Metadata Store database:

* Metadata about the artifacts generated through the components/steps of your ML pipelines
* Metadata about the executions of these components/steps
* Metadata about pipelines and associated lineage information

The Metadata Store provides APIs to record and retrieve metadata to and from the storage backend. The storage backend is pluggable 
and can be extended. MLMD provides reference implementations for SQLite (which supports in-memory and disk) and MySQL out of the 
box.

.. _Storage backends and store connection configuration:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Storage backends and store connection configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The ``MetadataStore`` object receives a connection configuration that corresponds to the storage backend used.

""""""""""""""
Fake Database
""""""""""""""

Fake Database provides an in-memory DB (using SQLite) for fast experimentation and local runs. The database is deleted when 
the store object is destroyed.

.. code-block::python

    from ml_metadata import metadata_store
    from ml_metadata.proto import metadata_store_pb2

    connection_config = metadata_store_pb2.ConnectionConfig()
    connection_config.fake_database.SetInParent() # Sets an empty fake database proto.
    store = metadata_store.MetadataStore(connection_config)

""""""
SQLite
""""""

SQLite reads and writes files from disk.

.. code-block:: python

    from ml_metadata import metadata_store
    from ml_metadata.proto import metadata_store_pb2

    connection_config = metadata_store_pb2.ConnectionConfig()
    connection_config.sqlite.filename_uri = '...'
    connection_config.sqlite.connection_mode = 3 # READWRITE_OPENCREATE
    store = metadata_store.MetadataStore(connection_config)

""""""
MySQL
""""""

MySQL connects to a MySQL server.

.. code-block:: python

    from ml_metadata import metadata_store
    from ml_metadata.proto import metadata_store_pb2

    connection_config = metadata_store_pb2.ConnectionConfig()
    connection_config.mysql.host = '...'
    connection_config.mysql.port = '...'
    connection_config.mysql.database = '...'
    connection_config.mysql.user = '...'
    connection_config.mysql.password = '...'
    store = metadata_store.MetadataStore(connection_config)

Similarly, when using a MySQL instance with Google CloudSQL (`quickstart <https://cloud.google.com/sql/docs/mysql/quickstart>`__, 
`connect-overview <https://cloud.google.com/sql/docs/mysql/connect-overview>`__), one could also use SSL option if applicable.

.. code-block:: python

    connection_config.mysql.ssl_options.key = '...'
    connection_config.mysql.ssl_options.cert = '...'
    connection_config.mysql.ssl_options.ca = '...'
    connection_config.mysql.ssl_options.capath = '...'
    connection_config.mysql.ssl_options.cipher = '...'
    connection_config.mysql.ssl_options.verify_server_cert = '...'
    store = metadata_store.MetadataStore(connection_config)

----------
Data model
----------

The Metadata Store uses the following data model to record and retrieve metadata from the storage backend.

* ``ArtifactType`` describes an artifact's type and its properties that are stored in the metadata store. You can register these types on-the-fly with the metadata store in code, or you can load them in the store from a serialized format. Once you register a type, its definition is available throughout the lifetime of the store.
* An ``Artifact`` describes a specific instance of an ``ArtifactType``, and its properties that are written to the metadata store. 
* An ``ExecutionType`` describes a type of component or step in a workflow, and its runtime parameters.
* An ``Execution`` is a record of a component run or a step in an ML workflow and the runtime parameters. An ``Execution`` can be thought of as an instance of an ``ExecutionType``. Executions are recorded when you run an ML pipeline or step.
* An ``Event`` is a record of the relationship between ``Artifact`` and ``Execution``. When an execution happens, events record every artifact that was used by the execution, and every artifact that was produced. These records allow for lineage tracking throughout a workflow. By looking at all ``Event``s, MLMD knows what ``Execution``s happened and what ``Artifact``s were created as a result. MLMD can then recurse back from any ``Artifact`` to all of its upstream inputs.
* A ``ContextType`` describes a type of conceptual group of ``Artifact`` and ``Execution`` in a workflow, and its structural properties. For example: projects, pipeline runs, experiments, owners, etc.
* A ``Context`` is an instance of a ``ContextType``. It captures the shared information within the group. For example: project name, changelist commit id, experiment annotations, etc. It has a user-defined **unique** name within its ``ContextType``.
* An ``Attribution`` is a record of the relationship between ``Artifact`` and ``Context``.
* An ``Association`` is a record of the relationship between ``Execution`` and ``Context``.

You may understand an ``Artifact`` as a concrete object that is produced by an ``Execution``. For example, a model is an 
``Artifact`` that is produced from an ``Execution`` of training step. The ``Context`` can be then considered as the whole run that 
encapsulates the experiment. And whenever a step (i.e. ``Execution``) produces an ``Artifact``, we can say that it emits an ``Event``, which binds an 
``Artifact`` and an ``Execution`` together. Finally, an ``Attribution`` connects an ``Artifact`` with a ``Context``, and an 
``Association`` connects an ``Execution`` with a ``Context``.

To summarize, 
* An ``Artifact`` (a model) is produced by an ``Execution`` (a workflow step) and they are bonded together by an ``Event``;
* An ``Execution`` (a workflow step) is part of a ``Context`` (an experiment) and they are bonded together by an ``Association``; 
* A ``Context`` (an experiment) generats an ``Artifact`` (a model) and they are bonded together by an ``Attribution``.

-------------
Functionality
-------------

Tracking the inputs and outputs of all components/steps in an ML workflow and their lineage allows ML platforms to enable several 
important features. The following list provides a non-exhaustive overview of some of the major benefits.

* List all Artifacts of a specific type, such as all models that have been trained.
* Show a directed acylic graph (DAG) of all related executions and their input and output artifacts of a context, such as a visualization of the workflow of an experiment for debugging and discovery.
* Recurse back through all events to see how an artifact was created.
* Record and query context of workflow runs, such as tracking the owner and changelist used for a workflow run and grouping the lineage by experiments.

-------------------------------
Integrate MLMD with ML workflow
-------------------------------

.. _Overall workflow:

^^^^^^^^^^^^^^^^
Overall workflow
^^^^^^^^^^^^^^^^

Now, it's time for us to strenthen the understanding of MLMD usage by going through an example. We here provide you with a typical 
ML workflow and and will show you how to use the low-level MLMD APIs to track the execution of a training task.

Following graph shows the overall workflow.

    .. image:: ../_static/user-guide-mlmd-workflow.png

This exmaple simply represents a workflow that uses a *dataset* (``Artifact``) to *train* (``Execution``) a *model* (``Artifact``).

^^^^^^^^^^^^^^^^^^^^^^
Step 0.1: Install MLMD
^^^^^^^^^^^^^^^^^^^^^^

We start from install MLMD library

""""""""""""""""""""
Installing from PyPI
""""""""""""""""""""

The recommended way to install ML Metadata is to use the `PyPI package <https://pypi.org/project/ml-metadata/>`__:

.. code-block:: shell
    
    pip install ml-metadata

Then import the relevant packages:

.. code-block:: python

    from ml_metadata import metadata_store
    from ml_metadata.proto import metadata_store_pb2

""""""""""""""""""""""
Installing with Docker
""""""""""""""""""""""

This is the recommended way to build ML Metadata under Linux.

Please first install ``docker`` and ``docker-compose`` by following the directions: `docker <https://docs.docker.com/get-docker/>`__; 
`docker-compose <https://docs.docker.com/compose/install/>`__.

Then, run the following at the project root:

.. code-block:: shell

    DOCKER_SERVICE=manylinux-python${PY_VERSION}
    sudo docker-compose build ${DOCKER_SERVICE}
    sudo docker-compose run ${DOCKER_SERVICE}

where ``PY_VERSION`` is one of ``{37, 38, 39}``.

A wheel will be produced under ``dist/``, and installed as follows:

.. code-block:: shell

    pip install dist/*.whl

""""""""""""""""""""""
Installing from source
""""""""""""""""""""""

If you prefer to installing MLMD from source, refer to `its github <https://github.com/google/ml-metadata#installing-from-source>`__ 
for detailed instruction.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 0.2: Configure store connection
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After installing and importing MLMD, we need to configure the store connection following instructions in 
section :ref:`Storage backends and store connection configuration`.

In this guide, we would install and import MLMD using PyPI, and use Fake Database for simplicity and convinience.

.. code-block:: text

    !pip install ml-metadata

.. code-block:: python

    from ml_metadata import metadata_store
    from ml_metadata.proto import metadata_store_pb2

    connection_config = metadata_store_pb2.ConnectionConfig()
    connection_config.fake_database.SetInParent() # Sets an empty fake database proto.
    store = metadata_store.MetadataStore(connection_config)

.. _Step 1:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 1: Register artifact types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We start from registering artifact types. You may see from the graph of the :ref:`Overall workflow`, we have two ``Artifact``, and 
therefore need to register two ``ArtifactType``.

Register ``ArtifactType`` for dataset:

.. code-block:: python

    data_type = metadata_store_pb2.ArtifactType()
    data_type.name = "DataSet"
    data_type.properties["day"] = metadata_store_pb2.INT
    data_type.properties["split"] = metadata_store_pb2.STRING
    data_type_id = store.put_artifact_type(data_type)

Register ``ArtifactType`` for saved model:

.. code-block:: python

    model_type = metadata_store_pb2.ArtifactType()
    model_type.name = "SavedModel"
    model_type.properties["version"] = metadata_store_pb2.INT
    model_type.properties["name"] = metadata_store_pb2.STRING
    model_type_id = store.put_artifact_type(model_type)

To see your registered ``ArtifactType``,

.. code-block:: python

    artifact_types = store.get_artifact_types()
    print(artifact_types)

and you should see outputs like below:

.. code-block:: text

    [id: 10
    name: "DataSet"
    properties {
    key: "day"
    value: INT
    }
    properties {
    key: "split"
    value: STRING
    }
    , id: 11
    name: "SavedModel"
    properties {
    key: "name"
    value: STRING
    }
    properties {
    key: "version"
    value: INT
    }
    ]

.. _Step 2:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 2: Register execution types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We then register execution types for all steps in the ML workflow. In this example, we will register an ``ExecutionType`` for 
trainer.

.. code-block:: python

    trainer_type = metadata_store_pb2.ExecutionType()
    trainer_type.name = "Trainer"
    trainer_type.properties["state"] = metadata_store_pb2.STRING
    trainer_type_id = store.put_execution_type(trainer_type)

You can query a registered ``ExecutionType`` with the returned ``id``:

.. code-block:: python

    [registered_type] = store.get_execution_types_by_id([trainer_type_id])
    print(registered_type)

And you should see output like below:

.. code-block:: text

    id: 12
    name: "Trainer"
    properties {
    key: "state"
    value: STRING
    }

.. _Step 3:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 3: Create dataset artifact
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Next, we create the dataset artifact using the Dataset ``ArtifactType`` we registered in :ref:`Step 1`.

.. code-block:: python

    data_artifact = metadata_store_pb2.Artifact()
    data_artifact.uri = 'path/to/data'
    data_artifact.properties["day"].int_value = 1
    data_artifact.properties["split"].string_value = 'train'

Use ``id`` to specify that this ``data_artifact`` is in ``Dataset`` artifact type:

.. code-block:: python

    data_artifact.type_id = data_type_id
    [data_artifact_id] = store.put_artifacts([data_artifact])

You can see your registered dataset artifact by querying all registered ``Artifact``.

.. code-block:: python

    artifacts = store.get_artifacts()
    print(artifacts)

You should see outputs like below:

.. code-block:: text

    [id: 1
    type_id: 10
    uri: "path/to/data"
    properties {
    key: "day"
    value {
        int_value: 1
    }
    }
    properties {
    key: "split"
    value {
        string_value: "train"
    }
    }
    create_time_since_epoch: 1675758739631
    last_update_time_since_epoch: 1675758739631
    ]

Plus, there are many ways to query the same ``Artifact``:

.. code-block:: python

    [stored_data_artifact] = store.get_artifacts_by_id([data_artifact_id])
    artifacts_with_uri = store.get_artifacts_by_uri(data_artifact.uri)
    artifacts_with_conditions = store.get_artifacts(
        list_options=mlmd.ListOptions(
            filter_query='uri LIKE "%/data" AND properties.day.int_value > 0'))

.. _Step 4:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 4: Create execution for trainer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We then create an ``Execution`` of the trainer run, using the Trainer ``ExecutionType`` we registered in :ref:`Step 2`.

.. code-block:: python

    trainer_run = metadata_store_pb2.Execution()

Similary, use ``id`` to specify that this ``Execution`` is in ``Trainer`` execution type.

.. code-block:: python

    trainer_run.type_id = trainer_type_id

And mark its status as "Running":

.. code-block:: python

    trainer_run.properties["state"].string_value = "RUNNING"
    [run_id] = store.put_executions([trainer_run])

You can see this execution by querying all registered ``Execution``.

.. code-block:: python

    executions = store.get_executions_by_id([run_id])
    print(executions)

And you should see output like below:

.. code-block:: text

    [id: 1
    type_id: 12
    properties {
    key: "state"
    value {
        string_value: "RUNNING"
    }
    }
    create_time_since_epoch: 1675758962852
    last_update_time_since_epoch: 1675758962852
    ]

Similarly, the same execution can be queried with conditions:

.. code-block:: python

    executions_with_conditions = store.get_executions(
        list_options = mlmd.ListOptions(
            filter_query='type = "Trainer" AND properties.state.string_value IS NOT NULL'))

.. _Step 5:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 5: Read Dataset and record input event
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now, it's time to define the input ``Event`` and read the data.

.. code-block:: python

    input_event = metadata_store_pb2.Event()
    input_event.type = metadata_store_pb2.Event.DECLARED_INPUT


Read the input data by specifying the ``input_event.artifact_id`` as the id of the dataset artifact we created :ref:`Step 3`.

.. code-block:: python

    input_event.artifact_id = data_artifact_id

Specify that this dataset artifact would be used as the input of the trainer execution using the id of the ``Execution`` we 
created for trainer run in :ref:`Step 4`.

.. code-block:: python
    
    input_event.execution_id = run_id

Finally, record this input ``Event`` in the metadata store.

.. code-block:: python

    store.put_events([input_event])

.. _Step 6:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 6: Train model and create saved model artifact
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Next, we declare the output ``Artifact``, which is the saved model, using the ``SavedModel`` artifact type we created in 
:ref:`Step 1`.

.. code-block:: python

    model_artifact = metadata_store_pb2.Artifact()
    model_artifact.uri = 'path/to/model/file'
    model_artifact.properties["version"].int_value = 1
    model_artifact.properties["name"].string_value = 'MNIST-v1'
    model_artifact.type_id = model_type_id
    [model_artifact_id] = store.put_artifacts([model_artifact])

.. _Step 7:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 7: Write saved model and record output event
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now, it's time to declare the output ``Event``.

.. code-block:: python

    output_event = metadata_store_pb2.Event()
    output_event.type = metadata_store_pb2.Event.DECLARED_OUTPUT

The output in this workflow would be the saved model. So we use the id of saved model ``Artifact`` we created in :ref:`Step 6` 
to specify.

.. code-block:: python

    output_event.artifact_id = model_artifact_id

Similar to the input event, we need to specify that this saved model artifact would be the output of the trainer execution.

.. code-block:: python

    output_event.execution_id = run_id

Finally, submit this output ``Event`` to the Metadata Store.

.. code-block:: python

    store.put_events([output_event])

.. _Step 8:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 8: Mark execution completed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We can now mark the execution as completed.

.. code-block:: python

    trainer_run.id = run_id
    trainer_run.properties["state"].string_value = "COMPLETED"
    store.put_executions([trainer_run])

.. _Step 9:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Step 9: Annote the experiment with a context
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Finally, we group artifacts and executions under a ``Context`` using ``Attribution`` and ``Association``.

Register a ``ContextType`` first.

.. important::
    The ``name`` of the ``ContextType`` must be unique.

.. code-block:: python

    experiment_type = metadata_store_pb2.ContextType()
    experiment_type.name = "Experiment"
    experiment_type.properties["note"] = metadata_store_pb2.STRING
    experiment_type_id = store.put_context_type(experiment_type)

Create a ``Context`` using the ``ContextType`` we just registered.

.. code-block:: python

    my_experiment = metadata_store_pb2.Context()
    my_experiment.type_id = experiment_type_id
    my_experiment.name = "exp1"
    my_experiment.properties["note"].string_value = "My first experiment."
    [experiment_id] = store.put_contexts([my_experiment])

Group the model ``Artifact`` to this experiment ``Context`` using ``Attribution``.

.. code-block:: python

    attribution = metadata_store_pb2.Attribution()
    attribution.artifact_id = model_artifact_id
    attribution.context_id = experiment_id

Group the trainer ``Execution`` to this experiment ``Context`` using ``Association``.

.. code-block:: python

    association = metadata_store_pb2.Association()
    association.execution_id = run_id
    association.context_id = experiment_id

Submit this ``Context`` to Metadata Store.

.. code-block:: python

    store.put_attributions_and_associations([attribution], [association])

You may confirm your previous works by querying the artifacts and executions that are linked to the experiment context.

.. code-block:: python

    experiment_artifacts = store.get_artifacts_by_context(experiment_id)
    experiment_executions = store.get_executions_by_context(experiment_id)

The ``experiment_artifacts`` should be like below:

.. code-block:: text

    [id: 2
    type_id: 11
    uri: "path/to/model/file"
    properties {
    key: "name"
    value {
        string_value: "MNIST-v1"
    }
    }
    properties {
    key: "version"
    value {
        int_value: 1
    }
    }
    create_time_since_epoch: 1675759027103
    last_update_time_since_epoch: 1675759027103
    ]

And the ``experiment_executions`` should be like below:

.. code-block:: text

    [id: 1
    type_id: 12
    properties {
    key: "state"
    value {
        string_value: "COMPLETED"
    }
    }
    create_time_since_epoch: 1675758962852
    last_update_time_since_epoch: 1675759074438
    ]

You can also use neighborhood queries to fetch these artifacts and executions with conditions:

.. code-block:: python

    experiment_artifacts_with_conditions = store.get_artifacts(
        list_options = mlmd.ListOptions(
            filter_query=('contexts_a.type = "Experiment" AND contexts_a.name = "exp1"')))
    experiment_executions_with_conditions = store.get_executions(
        list_options = mlmd.ListOptions(
            filter_query=('contexts_a.id = {}'.format(experiment_id))))

.. seealso::

    `Github repo for google/ml-metadata <https://github.com/google/ml-metadata>`__
    `MLMD API documentation <https://www.tensorflow.org/tfx/ml_metadata/api_docs/python/mlmd>`__











