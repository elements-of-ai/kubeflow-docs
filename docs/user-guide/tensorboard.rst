===========
TensorBoard
===========

Whether you're an expert or a beginner, TensorFlow offers an end-to-end platform that makes it easy for you to build and deploy ML 
models. TensorBoard provides the visualization and tooling needed for machine learning experimentation, such as tracking and 
visualizing loss and accuracy metrics, visualizing the model graph (ops and layers), displaying images, text, and audio data, etc.
It provides you with easy model building, robust ML prediction and powerful experimentation for research. You can learn more about 
the project on the `upstream website <https://www.tensorflow.org/tensorboard>`_.

In this guide, we will introduce you to some basic and powerful functionalities of Tensorboard by walking you through an example.

------------------------
Download exmple notebook
------------------------

Feel free to reuse the Notebook Server created in your previous guides. Connect to it and upload a new notebook for Tensorboard example.
`Download the notebook here <https://www.tensorflow.org/tensorboard/get_started>`_.

-----------------
Configure and run
-----------------

You can go through the example notebook to have a better idea about what this example is working on. Note that the ``log_dir`` path 
is imnportant - this location will be needed for Tensorboad creation.

    .. image:: ../_static/user-guide-tensorboard-logDir.png

Run the notebook, and navigate to "Tensorboards" from left toolbar on Kubeflow.

.. note:: 
    You may need to run ``pip install tensorflow`` to install tensorflow package if you do not have one.

    .. image:: ../_static/user-guide-tensorboard-toolbar.png

Click on "New Tensorboard". Name it and select the PVC checkbox. Select your notebook’s workspace volume from the dropdown list and 
fill in the ``Mount Path`` field with the ``log_dir`` you have noted in the previous step. In our example it’s ``logs/fit``.

    .. image:: ../_static/user-guide-tensorboard-new.png

That’s it! Click on "Create" and your Tensorboard should be up and running within minutes.

    .. image:: ../_static/user-guide-tensorboard-ready.png

You can then connect to it and see various metrics and graphs.

    .. image:: ../_static/user-guide-tensorboard-graph.png

.. seealso::

    `Get started with Charmed Kubeflow <https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow>`_