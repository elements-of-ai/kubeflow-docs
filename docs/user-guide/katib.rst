=====================
Hyperparameter Tuning
=====================

We would use **Katib** for hyperparameter tuning. Katib is a Kubernetes-native project for automated machine learning (AutoML). 
It supports hyperparameter tuning, early stopping and neural architecture search (NAS). It can tune hyperparameters of applications 
written in any language of the users' choice and natively supports many ML frameworks, such as TensorFlow, MXNet, PyTorch, XGBoost, 
and others. In this guide, we would use Katib to automate the tuning of machine learning hyperparameters which control the AI 
learning way and rate. Katib would also be used to offer neural architecture search features in order to help you find your model's 
optimal architecture. If you are unfamiliar with hyperparameters tuning or Katib, please refer to 
`Kubeflow documentation <https://www.kubeflow.org/docs/components/katib/overview/>`_ for more information.

We would use Katib to get the most effective configuration for the current task by running multiple experiments, each representing
a single tuning operation. An experiment consists of an objective specifying what is to be optimized, a search space representing
the constraints used for the optimization, and an algorithm that is used to find the optimal values.

In this guide, we would show two ways to run Katib experiments.

Run Katib experiments from CLI
--------------------------------------

Katib experiments can be ran through Kubeflow UI.

We first download an example to create a Katib experiment.

.. code-block:: shell

    curl https://raw.githubusercontent.com/kubeflow/katib/master/examples/v1beta1/hp-tuning/grid.yaml > grid-example.yaml

You should now be able to see the file ``grid-example.yaml``. Note that if you are using a different namespace than ``kubeflow``, 
make sure to change that in ``grid-example.yaml``.

As istio sidecar is incompatible with Katib experiments, we disable it using ``yq``. For ``yq`` installation, please refer to 
`this github page <https://github.com/mikefarah/yq/#install>`_ .

.. code-block:: shell

    yq -i '.spec.trialTemplate.trialSpec.spec.template.metadata.annotations."sidecar.istio.io/inject" = "false"' grid-example.yaml

And we are now ready to apply the YAML file to start our experiments.

.. code-block:: shell

    kubectl apply -f grid-example.yaml

Katib experiments would then start to run. We can inspect experiment progress using ``kubectl`` by running the following command:

.. code-block:: shell

    kubectl -n kubeflow get experiment grid-example -o yaml

Run Katib experiments from Kubeflow UI
--------------------------------------

We can also use the Kubeflow UI to run Katib experiments. 

Again, we first save following contents as ``grid-example.yaml`` which would be used to generate Katib experiment.

.. code-block:: yaml

    ---
    apiVersion: kubeflow.org/v1beta1
    kind: Experiment
    metadata:
      namespace: <YOUR_NAME_SPACE>
      name: grid
    spec:
      objective:
        type: maximize
        goal: 0.99
        objectiveMetricName: Validation-accuracy
        additionalMetricNames:
          - Train-accuracy
      algorithm:
        algorithmName: grid
      parallelTrialCount: 3
      maxTrialCount: 12
      maxFailedTrialCount: 3
      parameters:
        - name: lr
          parameterType: double
          feasibleSpace:
            min: "0.001"
            max: "0.01"
            step: "0.001"
        - name: num-layers
          parameterType: int
          feasibleSpace:
            min: "2"
            max: "5"
        - name: optimizer
          parameterType: categorical
          feasibleSpace:
            list:
              - sgd
              - adam
              - ftrl
      trialTemplate:
        primaryContainerName: training-container
        trialParameters:
          - name: learningRate
            description: Learning rate for the training model
            reference: lr
          - name: numberLayers
            description: Number of training model layers
            reference: num-layers
          - name: optimizer
            description: Training model optimizer (sdg, adam or ftrl)
            reference: optimizer
        trialSpec:
          apiVersion: batch/v1
          kind: Job
          spec:
            template:
              metadata:
                annotations:
                  sidecar.istio.io/inject: "false"
              spec:
                containers:
                  - name: training-container
                    image: docker.io/kubeflowkatib/mxnet-mnist:latest
                    command:
                      - "python3"
                      - "/opt/mxnet-mnist/mnist.py"
                      - "--batch-size=64"
                      - "--lr=${trialParameters.learningRate}"
                      - "--num-layers=${trialParameters.numberLayers}"
                      - "--optimizer=${trialParameters.optimizer}"
                restartPolicy: Never

Open the ``grid-example.yaml`` file and edit it, changing ``.metadata.namespace`` to your own. Note that here we also disable istio 
sidecar using ``sidecar.istio.io/inject: "false"``, under ``.spec.trialTemplate.trialSpec.spec.template.metadata.annotations``.

On Kubeflow in your browswer, go to Experiments (AutoML) from left panel, and select "New Experiment".

    .. image:: ../_static/user-guide-katib-experiment.png

Click the link labelled "Edit and submit YAML", and paste the contents of the ``grid-example.yaml`` file we just saved and edited. 
Afterwards click "Create".

    .. image:: ../_static/user-guide-katib-createExperiment.png

Once the experiment has been created and submitted, in the Katib dashboard under "Experiment (AutoML)", you should see the experiment
start to run. To inspect the experiment for more details, click on the experiment.

    .. image:: ../_static/user-guide-katib-dashboard.png

In the Experiment Details view, you can see how your experiment is progressing, such as information of each running trial, the 
experiment's YAML file, and a plot recording parameters and metrics related to your experiment.

    .. image:: ../_static/user-guide-katib-dashboard2.png

    .. image:: ../_static/user-guide-katib-details.png

The experiment would keep running until the objective you set in the YAML file gets realized, or timed out. In this example, when
the experiment finishes, you should be able to see the recommended hyperparameters information, as well as the results of all trial
and the optimal metrics.

    .. image:: ../_static/user-guide-katib-dashboardSuccess.png

    .. image:: ../_static/user-guide-katib-result.png

.. note:: 
    the experiment may take some time to finish, maybe from 30 minutes to about 2 hours.

.. seealso::

    `Get started with Charmed Kubeflow <https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow>`_


