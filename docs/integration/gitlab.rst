======
GitLab
======

--------
Overview
--------

From planning to production, `GitLab <https://about.gitlab.com/>`__ brings teams together to shorten cycle times, reduce costs, 
strengthen security, and increase developer productivity. It helps the engineering teams remove toolchain complexity and accelerate 
DevOps adoption. GitLab provides DevOps software and version control management that is based on Git. The platform provides tools 
for continuous integration, security, and continuous deployment. It offers features such as built-in continuous integration and 
deployment, project management, code review analytics, issue trackikng, etc, which can all useful for MLOps. More details can be 
found in `Gitlab official docs <https://docs.gitlab.com/ee/>`__.

In this guide, we will cover following contents:

* Deploy Gitlab on Kubernetes
* **TODO**

---------------------------
Deploy Gitlab on Kubernetes
---------------------------

The journey of integraing Gitlab into MLOps starting from deploying it on Kubernetes. In this guide, we will use `Helm <https://helm.sh/>`__.

.. _prerequisites:

^^^^^^^^^^^^^
Prerequisites
^^^^^^^^^^^^^

Before deployment, make sure following prerequisites are fulfilled:

* ``kubectl`` CLI installed. You may follow `the Kubernetes documentation <https://kubernetes.io/docs/tasks/tools/#kubectl>`__, or follow our doc :ref:`install-ubuntu`.
* Helm v3.3.1 or later installed. You may refer `the Helm documentation <https://helm.sh/docs/intro/install/>`__.
* (Optional) An external, production-ready PostgreSQL instance setup. This is optional, as the GitLab chart we would use in this guide already includes an in-cluster PostgreSQL deployment that is provided by `bitnami/PostgreSQL <https://artifacthub.io/packages/helm/bitnami/postgresql>`__ by default. This deployment, however, is for trial purposes only and not recommended for use in production.
* (Optional) An external, production-ready Redis instance setup. This is optional, as the GitLab chart we would use in this guide already includes an in-cluster Redis deployment that is provided by `bitnami/Redis <https://artifacthub.io/packages/helm/bitnami/redis>`__ by default. This deployment, however, is for trial purposes only and not recommended for use in production.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Use Helm chart for Gitlab deployment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

""""""""""""""""""""""""
Add Gitlab to helm repo
""""""""""""""""""""""""

After all :ref:`prerequisites` are fulfilled, we can start our Gitlab deployment on Kubernetes.

First, add ``gitlab`` to ``helm repo``.

.. code-block:: shell

    helm repo add gitlab https://charts.gitlab.io/

There may be some warnings, and this step can be skipped if you have already have ``gitlab`` in your ``helm repo``.

.. image:: ../_static/integration-gitlab-addRepo.png

Update ``helm repo`` with above change.

.. code-block:: shell

    helm repo update

And you should see successful message like below:

.. image:: ../_static/integration-gitlab-updateRepo.png

.. _deploy:

""""""""""""""""""""
Configure and deploy
""""""""""""""""""""

Before deployment, you should make some decisions about how you will run GitLab. Options can be specified using Helm’s 
``--set option.name=value`` command-line option. This guide will cover required values and common options. For a complete list of 
options, read `Installation command line options <https://docs.gitlab.com/charts/installation/command-line-options.html>`__.

In this guide, we deploy Gitlab using following command:

.. code-block:: shell

    helm upgrade --install gitlab gitlab/gitlab
      --timeout 600s
      --set global.hosts.externalIP=<your_ingress_externalIP>
      --set global.hosts.domain=<your_ingress_externalIP>.nip.io
      --set certmanager-issuer.email=admin@example.com
      --set global.time_zone=<timezone_that_is_consistent_with_your_machine>
      --set postgresql.image.tag=13.6.0

Note the following:

* All Helm commands are specified using Helm v3 syntax.
* Helm v3 requires that the release name be specified as a positional argument on the command line unless the ``--generate-name`` option is used.
* Helm v3 requires one to specify a duration with a unit appended to the value (e.g. ``120s`` = ``2m`` and ``210s`` = ``3m30s``). The ``--timeout`` option is handled as the number of seconds without the unit specification.
* You need to use a valid external IP (in a valid range) for field ``global.hosts.externalIP`` and ``global.hosts.domain``. These two fields are all required. (You may check ``svc`` and ``ingress`` using ``[microk8s] kubectl`` to get a valid range for the external IP. In my case, it is ``10.64.140.46``.)
* ``certmanager-issuer.email`` field is required and it is used for root user login. You may customize the value.
* ``global.time_zone`` is not required and it has a default value ``UTC``. It is mandatory that you make sure your deployed Gitlab time zone is consistent with the time zone of your machine. Otherwise, there may be a cookie issue which would cause ``422`` error code in later web UI accessing. (You may use ``date`` command to check your machine's time zone.)
* You can also use ``--version <installation version>`` option if you would like to install a specific version of GitLab.
* Above command enables you to deploy **enterprise** version. If you would like to deploy a **community** version, add ``--set global.edition=ce``.
* In this guide, all related ``pods``, ``svc``, ``deployment``, ``ingress`` would be in ``default`` namespace. You may customize it.

^^^^^^^^^^^^^^^^^^^^^^
Monitor the deployment
^^^^^^^^^^^^^^^^^^^^^^

Monitor the deployment process using following command:

.. code-block:: shell

    helm status gitlab

And you should see messages like below after running above ``helm upgrade --install`` command:

.. image:: ../_static/integration-gitlab-install.png

Wait for a few minutes untill all required ``pods``, ``svc``, ``deployment``, ``ingress`` are ready. 

Check all pods are ready:

.. image:: ../_static/integration-gitlab-pods.png

Check all services are there:

.. image:: ../_static/integration-gitlab-svc.png

Check all ingress are on:

.. image:: ../_static/integration-gitlab-ingress.png

Check all deployments are ready:

.. image:: ../_static/integration-gitlab-deploy.png

^^^^^^^^^^^^^^^^^^^^
Access Gitlab web UI
^^^^^^^^^^^^^^^^^^^^

If you did not manually set root initial password, you need to first get the password for initial login.  GitLab automatically 
created a random password for root user. This can be extracted by the following command:

.. code-block:: shell

    kubectl get secret <name_of_release>-gitlab-initial-root-password -n <namespace> -ojsonpath='{.data.password}' | base64 --decode ; echo

If you use above commands, the ``<name_of_release>`` would be ``gitlab``, and the ``<namespace>`` would be ``default``.

Copy the password.

Open you browswer, and go to the Gitlab web UI using the ``domain`` we set above ``https://gitlab.<domain>:<port_for_https>``, i.e. 
``https://gitlab.<your_ingress_externalIP>.nip.io:<port_for_https>``. (In my case, it is ``https://gitlab.10.64.140.46.nip.io:443/``.)

And you should see following login page:

.. image:: ../_static/integration-gitlab-login.png

Enter the email using ``certmanager-issuer.email`` we previously set in :ref:`deploy`. And enter the password using either you manually 
set one or the one we get from ``secret``.

Click "Sign in", and you should be located to home page:

.. image:: ../_static/integration-gitlab-home.png

---------------
Troubleshooting
---------------

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
422 error code on web UI after login
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After clicking "Sign in", instead of being guided to Gitlab home page, one sees ``422 The change you requested was rejected`` error. Below 
are some possible reasons:

* Time zone and clock of your deployed Gitlab is inconsistent with your machine (local or virtual machine, depending on which one you have used to deploy Gitlab). This would cause some cookie problems. Check your machine's time zone (using ``date`` command, for example), and use ``--set global.time_zone=<your_machine_timezone>`` in ``helm install`` step.
* Cookie issues. Clear your browser's cookies.
* External IP is not set properly. Run ``[microk8s] kubectl get svc -n default`` to make sure the ingress controller has a valid external IP allocated. And make sure the external IP is in a valid range.
* Double check if you are using ``https`` and corresponding port (``443`` in most cases).
* Change the domain. In some tutorials, you may see domain ``example.com``, ``xip.io``, etc. It may depend on your environment and network configurations. In my case, the working version is ``<externalIP>.nip.io``. And to access Gitlab on web UI, the one to be used would be ``https://gitlab.<externalIP>.nip.io:443``.

