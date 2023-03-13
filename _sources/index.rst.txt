.. Kubeflow documentation master file, created by
   sphinx-quickstart on Fri Jan 13 06:56:36 2023.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

.. image:: /_static/kubeflow-logo.png
   :align: center
   :scale: 20%

Welcome
=======

Open Source (OSS) Kubeflow enables you to operationalize much of an ML workflow on top of Kubernetes. It comprises a number of ML components and services; SDKs and APIs; integrated development environments (IDEs); and libraries for data science and machine learning.

The Kubeflow Enterprise (KFE) distribution introduces important additional features to address gaps in OSS Kubeflow and commonly expressed needs of MLOps engineers and data scientists.

- **Automation:** With KFE you can orchestrate an end-to-end ML workflow from your IDE. Start by tagging cells in Jupyter Notebooks to define pipeline steps, hyperparameter tuning, GPU usage, and metrics tracking. Click a button to define the necessary Kubernetes services and run a scalable ML pipeline and serve the best model. Or use the KFE Kale SDK to do all the above within your preferred IDE.

- **Portability:** KFE enables you to deploy and upgrade a Kubeflow environment using GitOps processes across all major public clouds and on-prem infrastructure. Move ML workflows seamlessly across clouds.

- **Security:** KFE security features enable you to manage teams and user access via GitLab or any ID provider via Istio/OIDC. Isolate user ML data access within their own namespace while enabling notebook and pipeline collaboration in shared namespaces. Manage secrets and credentials securely, and efficiently.

Docs
----

.. toctree::
    :hidden:

    introduction/index
    introduction/oss
    introduction/ekf
    introduction/mlops

.. toctree::
    :caption: Install
    :hidden:

    install/index
    install/ubuntu

.. toctree::
    :caption: Integration
    :hidden:

    integration/index
    integration/gitlab
    integration/identity
    integration/auth

.. toctree::
    :caption: User Guide
    :hidden:

    user-guide/index
    user-guide/notebooks
    user-guide/feast
    user-guide/spark
    user-guide/mlmd
    user-guide/mlflow
    user-guide/training
    user-guide/tensorboard
    user-guide/katib
    user-guide/kserve
    user-guide/kfp

.. toctree::
    :caption: Use Cases
    :hidden:

    use-cases/helmet

.. toctree::
    :caption: Operation Guide
    :hidden:

    operation-guide/index

.. toctree::
    :caption: Internals
    :hidden:

    internals/auth

License
-------

Open Source (OSS) Kubeflow is released under the Apache License 2.0.

Kubeflow Enterprise and its documents are offered as free, open-source software. You don’t need a support agreement or license to deploy them.

.. Indices and tables
   ==================

   * :ref:`genindex`
   * :ref:`modindex`
   * :ref:`search`
