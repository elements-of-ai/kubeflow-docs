=========================================
PyTorch Distributed Training (PyTorchJob)
=========================================

--------
Overview
--------

This section will guide you through creating and managing a ``PyTorchJob`` CR on Kubeflow to run PyTorch training jobs on Kubernetes. The Kubeflow implementation of ``PyTorchJob`` is in `training-operator <https://github.com/kubeflow/training-operator>`_.

-------------
Prerequisites
-------------

* Deployed Kubeflow instance, and access to Kubeflow dashboard. For sample Kubeflow deployment on Nimbus (only for VMware internal users) refer to :ref:`install-ubuntu`. More details on Kubeflow can be found here `Documentation | Kubeflow <https://www.kubeflow.org/docs/>`__.
* Deployed training-operator. 

------------
Instructions
------------

You’ll need a working Kubeflow deployment with PyTorch Operator up and running. During the journey through the ``PyTorchJob``, each step will show us something new. Let’s go!

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Verify that PyTorchJob support is included in your Kubeflow deployment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Check that the PyTorch custom resource is installed:

.. code-block:: shell

    $ microk8s kubectl get crd
    NAME                                             CREATED AT
    ...
    pytorchjobs.kubeflow.org                         2023-01-31T06:02:59Z
    ...

Check that the Training operator is running via:

.. code-block:: shell

    $ microk8s kubectl get pods -n kubeflow
    NAME                                READY   STATUS    RESTARTS   AGE
    ...
    training-operator-0                 2/2     Running   4 (6d1h ago)    6d2h
    ...


^^^^^^^^^^^^^^^^^^^
Create a new volume
^^^^^^^^^^^^^^^^^^^

In user's namespace, create a new volume named ``data``, with ``ReadWriteOnce`` access mode. The training job will save logs and model file in the volume.

.. code-block:: shell

    USER_NAMESPACE=admin
    kubectl config set-context --current --namespace=${USER_NAMESPACE}

    cat <<EOF | microk8s kubectl create -n $USER_NAMESPACE -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: data
    labels:
        app: data
    spec:
    storageClassName: microk8s-hostpath
    accessModes:
    - ReadWriteOnce
    resources:
        requests:
        storage: 10Gi
    EOF


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Creating a PyTorch training job
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can create a training job by defining a PyTorchJob config file. See the manifests for the distributed NLP example. You may change the config file based on your requirements.

Deploy the PyTorchJob resource with **CPU** to start training:

.. code-block:: shell

  cat <<EOF | microk8s kubectl create -n $USER_NAMESPACE -f -
  apiVersion: "kubeflow.org/v1"
  kind: "PyTorchJob"
  metadata:
    name: "pytorchjob-distributed-training"
  spec:
    pytorchReplicaSpecs:
      Master:
        replicas: 1
        restartPolicy: OnFailure
        template:
          metadata:
            annotations:
              sidecar.istio.io/inject: "false"
          spec:
            volumes:
              - name: data
                persistentVolumeClaim:
                  claimName: data
            containers:
              - volumeMounts:
                  - name: data
                    mountPath: "/opt/pytorch/output"
                name: pytorch
                image: harbor-repo.vmware.com/kubeflow_learning/lab3-pytorch-training:dlc-0.0.1
                imagePullPolicy: IfNotPresent
      Worker:
        replicas: 2
        restartPolicy: OnFailure
        template:
          metadata:
            annotations:
              sidecar.istio.io/inject: "false"
          spec:
            containers: 
              - name: pytorch
                image: harbor-repo.vmware.com/kubeflow_learning/lab3-pytorch-training:dlc-0.0.1
                imagePullPolicy: IfNotPresent
    EOF

Deploy the PyTorchJob resource with **GPU** to start training:


.. code-block:: shell

  cat <<EOF | microk8s kubectl create -n $USER_NAMESPACE -f -
  apiVersion: "kubeflow.org/v1"
  kind: "PyTorchJob"
  metadata:
    name: "pytorchjob-distributed-training-gpu"
  spec:
    pytorchReplicaSpecs:
      Master:
        replicas: 1
        restartPolicy: OnFailure
        template:
          metadata:
            annotations:
              sidecar.istio.io/inject: "false"
          spec:
            volumes:
              - name: data02
                persistentVolumeClaim:
                  claimName: data02
            containers:
              - volumeMounts:
                - name: data02
                  mountPath: "/opt/pytorch/output"
                name: pytorch
                image: harbor-repo.vmware.com/juanl/kubeflow-pytorch-training-cuda:v1
                args: ["--backend", "nccl"]
                imagePullPolicy: IfNotPresent
                resources: 
                  limits:
                    nvidia.com/gpu: 1
      Worker:
        replicas: 1
        restartPolicy: OnFailure
        template:
          metadata:
            annotations:
              sidecar.istio.io/inject: "false"
          spec:
            containers: 
              - name: pytorch
                image: harbor-repo.vmware.com/juanl/kubeflow-pytorch-training-cuda:v1
                args: ["--backend", "nccl"]
                imagePullPolicy: IfNotPresent
                resources: 
                  limits:
                    nvidia.com/gpu: 1
  EOF

You should now be able to see the created pods matching the specified number of replicas.

.. code-block:: shell

    microk8s kubectl get pods -l job-name=pytorchjob-distributed-training -n admin


^^^^^^^^^^^^^^^^^^^^^^^
Monitoring a PyTorchJob
^^^^^^^^^^^^^^^^^^^^^^^

Training takes 5-10 minutes to complet the training process to monitor the job status to become success. Logs can be inspected to see its training progress. 

.. code-block:: shell

    microk8s kubectl get pods -n admin | grep pytorchjob-distributed-training
    microk8s kubectl logs -f pytorchjob-distributed-training-master-0 -n admin
    microk8s kubectl logs -f pytorchjob-distributed-training-worker-0 -n admin
    microk8s kubectl logs -f pytorchjob-distributed-training-worker-1 -n admin

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Test whether PyTorchJob is distributed training
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Firstly, deploy the single pod to start training:

.. code-block:: shell

  cat <<EOF | microk8s kubectl create -n $USER_NAMESPACE -f -
  apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      sidecar.istio.io/inject: "false"
    name: "pytorch-training-single-pod"
  spec:
    volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data
    containers:
      - name: pytorch
        image: harbor-repo.vmware.com/kubeflow_learning/lab3-pytorch-training:dlc-0.0.1
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - name: data
            mountPath: "/opt/pytorch/output"
  EOF

Waiting 10-15 minutes to complet the training process to check logs.

Secondly, compare the training logs between the single pod and the PytorchJob.

.. image:: ../_static/user-guide-training-pytorchjob-result.png

From the picture, the model was trained 48 times for epoch 6 in the single-pod. And after using Pytorch operator, the model individually was trained 16 times in the master and 2 workers, although the loss value after each training is different, the accuracy obtained is the same after the master communicates with the 2 workers.

.. seealso::

   `Use PytorchJob to train a model for predict Spam email <https://vmware.github.io/ml-ops-platform-for-vsphere/docs/kubeflow-tutorial/lab3/>`_
