.. _kubeflow_pipelines:

==================
Kubeflow Pipelines
==================

.. toctree::
   :hidden:

   kfp-mlflow-seldon
   kfp-mlflow-katib

The official Kubeflow Documentation explains the recommended workflow for creating a pipeline. This documentation is well worth reading thoroughly to understand how pipelines are constructed. For this example run-through though, we can take a shortcut and use one of the `Kubeflow testing pipelines <https://github.com/canonical/bundle-kubeflow/blob/master/tests/pipelines/mnist.py>`_.

Install pipeline compiler
-------------------------

To install the pipeline compiler tools, you will need to first have Python3 available, and whichever ``pip`` install tool is relevant for your OS. On Ubuntu 20.04 and similar systems:

.. code-block:: shell

   sudo apt update
   sudo apt install python3-pip

Next, use ``pip`` to install the Kubeflow Pipeline package:

.. code-block:: shell

   pip3 install kfp

(Depending on your operating system, you may need to use ``pip`` instead of ``pip3`` here, but make sure the package is installed for Python3.)

Get a pipeline example
----------------------

Next fetch the Kubeflow repository:

.. code-block:: shell

   git clone https://github.com/canonical/bundle-kubeflow.git

The example pipelines are Python files, but to be used through the dashboard, they need to be compiled into a YAML file. The ``dsl-compile`` command can be used for this usually, but for code which is part of a larger package, this is not always straightforward. A reliable way to compile such files is to execute them as a python module in interactive mode, and then use the ``kfp`` tools within Python to compile the file.

Compile pipeline
----------------

First, change to the right directory:

.. code-block:: shell

   cd bundle-kubeflow/tests

Then execute the ``pipelines/mnist.py`` file as a module:

.. code-block:: shell

   python3 -i -m pipelines.mnist

With the terminal now in interactive mode, we can import the kfp module:

.. code-block:: python

   import kfp

... and execute the function to compile the YAML file:

.. code-block:: python

   kfp.compiler.Compiler().compile(mnist_pipeline, 'mnist.yaml')

In this case, ``mnist_pipeline`` is the name of the main pipeline function in the code, and ``mnist.yaml`` is the file we want to generate.

Add the compiled pipeline
-------------------------

Once you have the compiled YAML file (or downloaded it from the link above) go to the "Kubeflow Pipelines" Dashboard and click on the "Upload Pipeline" button.

In the upload section choose the "Upload a file" section and choose the ``mnist.yaml`` file. Then click "Create" to create the pipeline.

.. image:: ../_static/kfp-1.png

Once the pipeline is created we will be redirected to its Dashboard. Create an experiment first:

.. image:: ../_static/kfp-2.png

Execute the pipeline
--------------------

Once the experiment is added, you will be redirected to "Start a Run". For this test select "One-off" run and leave all the default parameters and options. Then click "Start" to create your first Pipeline run!

.. image:: ../_static/kfp-3.png

Look at results
---------------

Once the run is started, the browser will redirect to "Runs", detailing all the stages of the pipeline run. After a few minutes there should be a checkpoint showing that it has been executed successfully.

.. image:: ../_static/kfp-4.png

In order to see it, you click on it and a new window will open that will show all the steps of the pipeline that have been executed. Clicking on each step of the pipeline, a new window will open on the right side, showing you the detailed information of the corresponding pipeline task, such as its pod, logs, YAML, etc.

.. image:: ../_static/kfp-5.png

Delete pipeline
---------------

In order to delete the pipeline, you need to select it, using the thick box placed on the left side of the name. Then, go to the top right corner, and click "Delete".

A new window will appear and ask you to confirm the pipeline deletion. Click again on "Delete".

.. image:: ../_static/kfp-6.png

That’s it, your pipeline is now deleted!

.. seealso::
   - `Get started with Charmed Kubeflow <https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow#heading--kubeflow-pipeline>`_
