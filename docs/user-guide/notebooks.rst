==================
Kubeflow Notebooks
==================

Kubeflow Notebooks provides a way to run web-based development environments inside your Kubernetes cluster by running them inside Pods.

Create a Kubeflow Notebook
--------------------------

The dashboard will give you an overview of the Notebook Servers currently available on your Kubeflow installation. In a freshly installed Kubeflow there will be no Notebook Server.
You will create a new Notebook Server by clicking on ``Notebooks`` in the left-side navigation and then clicking on the ``New notebook`` button.

In the ``New Notebook`` section you will be able to specify several options for the notebook you are creating. In the image section choose an image of ``jupyter-tensorflow-full``, it is required for our example notebook. Please leave the CPU and memory requirements to the default ones.

Once the Notebook Server is created you connect to it, by clicking on the ``Connect`` button from the left side to it and access your Jupyter Notebook environment which will be opened in a new tab.

.. image:: ../_static/notebook-1.png

For testing the server we will upload the `Tensorflow 2 quickstart for experts example <https://www.tensorflow.org/tutorials/quickstart/advanced>`_.

Click on the link above and click on the ``Download Notebook`` button just below the heading. This will download the file ``advanced.ipynb`` into your usual Download location. This file will be used to create the example notebook.

On the ``Notebook Server`` page, click on the ``Upload`` button, which is located in the side-bar, at the top, and select the ``advanced.ipnyb`` file.

.. image:: ../_static/notebook-2.png

Once uploaded, click on the notebook name to open a new tab with the notebook content.

.. image:: ../_static/notebook-3.png

You can read through the content for a better understanding of what this notebook does. Click on the Run button ``▶︎``  to execute each stage of the document, or click on the double-chevron ``>>`` to execute the entire document.

Kubeflow Notebook Volume
------------------------

In order to see the volume of the notebook that you just created in the previous step, please click on Volumes on the left side-bar. You will see a volume that has the same name as the notebook with ”-volume” at the end.

.. image:: ../_static/notebook-4.png

Delete a Kubeflow Notebook
--------------------------

In order to delete a new Notebook, you will click on Notebooks in the left-side navigation. Go to the notebook you want to delete, an click on the small trash bin icon situated alongside the Notebook.

.. image:: ../_static/notebook-5.png

A new window will appear on your screen. Click ``Delete``.

.. image:: ../_static/notebook-6.png

Container Images
----------------

// TODO: update with contents from https://www.kubeflow.org/docs/components/notebooks/container-images/

.. seealso::
   - `Arrikto Kubeflow Notebooks <https://docs.arrikto.com/features/notebook-images.html#>`_
   - `Get started with Charmed Kubeflow <https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow#heading--kubeflow-notebooks>`_
   - `Kubeflow Notebooks <https://www.kubeflow.org/docs/components/notebooks/>`_
   - `Example Notebook Servers <https://github.com/kubeflow/kubeflow/tree/master/components/example-notebook-servers>`_
