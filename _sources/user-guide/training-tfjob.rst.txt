=======================================
Tensorflow Distributed Training (TFJob)
=======================================

--------
Overview
--------

This section will guide you through creating and managing a ``TFJob`` CR on Kubeflow to run Tensorflow training jobs on Kubernetes. The Kubeflow implementation of ``TFJob`` is in `training-operator <https://github.com/kubeflow/training-operator>`_.

-------------
Prerequisites
-------------

* Deployed Kubeflow instance, and access to Kubeflow dashboard. For sample Kubeflow deployment on Nimbus (only for VMware internal users) refer to :ref:`install-ubuntu`. More details on Kubeflow can be found here `Documentation | Kubeflow <https://www.kubeflow.org/docs/>`__.
* Deployed training-operator. 

------------
Instructions
------------

You’ll need a working Kubeflow deployment with Tensorflow Operator up and running. During the journey through the ``TFJob``, each step will show us something new. Let’s go!

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Verify that TFJob support is included in your Kubeflow deployment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Check that the PyTorch custom resource is installed:

.. code-block:: shell

    $ microk8s kubectl get crd
    NAME                                             CREATED AT
    ...
    tfjobs.kubeflow.org                         2023-01-31T06:02:59Z
    ...

Check that the Training operator is running via:

.. code-block:: shell

    $ microk8s kubectl get pods -n kubeflow
    NAME                                READY   STATUS    RESTARTS   AGE
    ...
    training-operator-0                 2/2     Running   4 (6d1h ago)    6d2h
    ...


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Creating a Tensorflow training job: Mnist example
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Deploy the TFJob resource with **CPU** to start training:

.. code-block:: shell

  USER_NAMESPACE=admin
  kubectl config set-context --current --namespace=${USER_NAMESPACE}

  cat <<EOF | microk8s kubectl create -n $USER_NAMESPACE -f -
  apiVersion: "kubeflow.org/v1"
  kind: TFJob
  metadata:
    name: tfjob-simple
  spec:
    tfReplicaSpecs:
      Worker:
        replicas: 2
        restartPolicy: OnFailure
        template:
          metadata:
            annotations:
              sidecar.istio.io/inject: "false"
        template:
          spec:
            containers:
              - name: tensorflow
                image: gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0
                command:
                  - "python"
                  - "/var/tf_mnist/mnist_with_summaries.py"
  EOF

You should now be able to see the created pods matching the specified number of replicas.

.. code-block:: shell

    $ microk8s kubectl get pods -l job-name=tfjob-simple -n admin


^^^^^^^^^^^^^^^^^^
Monitoring a TFJob
^^^^^^^^^^^^^^^^^^

Check the events for your job to see if the pods were created.

.. code-block:: shell

    $ microk8s kubectl describe tfjobs tfjob-simple -n admin
    ...
    Events:
    Type    Reason                   Age                From              Message
    ----    ------                   ----               ----              -------
    Normal  SuccessfulCreatePod      78s                tfjob-controller  Created pod: tfjob-simple-worker-0
    Normal  SuccessfulCreatePod      77s                tfjob-controller  Created pod: tfjob-simple-worker-1
    Normal  SuccessfulCreateService  77s                tfjob-controller  Created service: tfjob-simple-worker-0
    Normal  SuccessfulCreateService  77s                tfjob-controller  Created service: tfjob-simple-worker-1

Check the logs to see the training result when the training process completed.

.. code-block:: shell

    $ microk8s kubectl logs -f tfjob-simple-worker-0 -n admin
    $ microk8s kubectl logs -f tfjob-simple-worker-1 -n admin

.. seealso::

   `Using TFJob to train a model with TensorFlow <https://www.kubeflow.org/docs/components/training/tftraining/#running-the-mnist-example>`_
