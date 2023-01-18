==========================
Install Kubeflow on Nimbus
==========================

This document will introduce you to all you need to know to get started with Charmed Kubeflow on Nimbus.

**Prerequisites**:

To keep things simple, we are going to make the following assumptions:

* You run Ubuntu 20.04 (focal) or later.
* You have at least 32GB free memory and 50GB of disk space.
* You have access to the internet for downloading the required snaps and charms.

Create a Nimbus Ubuntu 20.04 VM
===============================

.. code-block:: shell

    # login DBC account to deloy ubuntu 20.04 vm on nimbus
    # username/password: vmware/B1gd3m0z
    nimbus deploy ovf atlas-ubuntu-vm-3 http://sc-prd-rdops-templates.eng.vmware.com/nimbus-templates/atlas-ubuntu-20-4/atlas-ubuntu-20-04/atlas-ubuntu-20-04.ovf --cpus=16

    # login this vm, and edit /etc/environment and remove proxy related env. vars, then reboot
 

Access Charmed Kubeflow
=======================

Please note that this tutorial is dedicated to users who aim to install Charmed Kubeflow on their own machine or on a virtual machine with direct access to the browser. In case you run it on a public cloud, the deployment process is the same, but please follow these `instructions <https://charmed-kubeflow.io/docs/dashboard>`_ for accessing the dashboard.


Login Nimbus VM
---------------

The journey of access Charmed Kubeflow will start from login Nimbus VM.

.. code-block:: shell

    ssh -D localhost:1080 vmware@<vm_ip>


Install and prepare MicroK8s
----------------------------

The first step on our journey is to install `MicroK8s <https://microk8s.io/>`_. MicroK8s is installed from a snap package. The published snap maintains different channels for different releases of Kubernetes.

.. code-block:: shell 

    sudo snap install microk8s --classic --channel=1.24/stable

For MicroK8s to work without having to use sudo for every command, it creates a group called microk8s. To make it more convenient to run commands, you will add the current user to this group:

.. code-block:: shell

    sudo usermod -a -G microk8s $USER
    newgrp microk8s

It is also useful to make sure the user has the proper access and ownership of any kubectl configuration files:

.. code-block:: shell

    sudo chown -f -R $USER ~/.kube

MicroK8s will start up as soon as it is installed. It is a completely functional Kubernetes, running with the least amount of overhead possible. However, for our purposes we will need a Kubernetes with a few more features. A lot of extra services are available as MicroK8s “add-ons” - code which is shipped with the snap and can be turned on and off when it is needed. We can now enable some of these features to get a Kubernetes where we can usefully install Kubeflow. We will add a DNS service, so the applications can find each other, storage, an ingress controller so we can access Kubeflow components and the MetalLB load balancer application. These will be enabled simply at the same time:

.. code-block:: shell

    microk8s enable dns storage ingress metallb:10.64.140.43-10.64.140.49

You can see that we added some detail when enabling MetalLB, in this case the address pool to use. Many of the add-ons have extra configuration options, which can be found in the `MicroK8s documentation <https://microk8s.io/docs/addon-metallb>`_.

It will take minimum 5 minutes for MicroK8s to install and set up these additional features. Before we do anything else, we should check that the add-ons have been enabled successfully and that MicroK8s is ready for action. We can do this by requesting the status, and supplying the --wait-ready option, which tells microk8s to finish whatever processes it is working on before returning:

.. code-block:: shell

    microk8s status --wait-ready

Now we have a working Kubernetes ready, the next step is to install Juju.


Install Juju
------------

`Juju <https://juju.is/>`_ is an operation Lifecycle manager (OLM) for clouds, bare metal or Kubernetes. We will be using it to deploy and manage the components which make up Kubeflow.
As with MicroK8s, Juju is installed from a snap package:

.. code-block:: shell

    sudo snap install juju --classic

As Juju already has a built-in knowledge of MicroK8s and how it works, there is no additional setup or configuration needed. All we need to do is run the command to deploy a Juju controller to the Kubernetes we set up with MicroK8s:

.. code-block:: shell

    juju bootstrap microk8s

The controller is Juju’s agent, running on Kubernetes, which can be used to deploy and control the components of Kubeflow.

The controller can work with different ``models``, which map to namespaces in Kubernetes. You set up a specific model for Kubeflow:

.. code-block:: shell

    juju add-model kubeflow

Model name must be Kubeflow: Due to an assumption made in the upstream Kubeflow Dashboard code, Kubeflow must be deployed in the Kubernetes namespace ``kubeflow`` and so we have to use the model name ``kubeflow`` here.

That’s it for installing Juju!


Deploying Charmed Kubeflow
--------------------------

Charmed Kubeflow is really a collection of charms. Each of these charms deploy and control one application which goes to make up Kubeflow. You can actually just install the components you want, by individually deploying the charms and relating them to each other to build up Kubeflow. The bundles are really a recipe for a particular deployment of Kubeflow, configuring and relating the applications so you end up with a working deployment with the minimum of effort.

.. code-block:: shell

    juju deploy kubeflow --trust

Juju will now fetch the applications and begin deploying them to the MicroK8s Kubernetes. This process can take several minutes. You can track the progress by running:

.. code-block:: shell

    watch -c juju status --color

This will show a list of the applications and their current status. Don’t be surprised if a few show up error messages to begin with - a lot of the components rely on the operation of others, so it can take up to 20 minutes before everything is ready and talking to one another.

While that is going in, there are two pieces of post-install configuration which can usefully be done at this point.


Configure the components
------------------------

For authentication and allowing access to the dashboard service, some components will need to be configured with the URL to be allowed. This depends on the underlying network provider, but for the known case of running on a local MicroK8s, we also know what the URL will be. It is configured with Juju using the following commands:

.. code-block:: shell

    juju config dex-auth public-url=http://10.64.140.43.nip.io
    juju config oidc-gatekeeper public-url=http://10.64.140.43.nip.io

Finding the URL: If you have a different setup for MicroK8s, or you are adapting this tutorial for a different Kubernetes, you can find the URL required by examining the IP address of the ``istio-ingressgateway`` service. For example, you can determinine this information using kubectl: ``microk8s kubectl -n kubeflow get svc istio-ingressgateway-workload -o jsonpath='{.status.loadBalancer.ingress[0].ip}'``

To enable simple authentication, and set a username and password for your Kubeflow deployment, run the following commands:

.. code-block:: shell

    juju config dex-auth static-username=admin
    juju config dex-auth static-password=admin

Feel free to use a different (more secure!) password if you wish.

Login to Charmed Kubeflow
-------------------------

Please note that if you are in a public cloud, follow `this guide <https://charmed-kubeflow.io/docs/dashboard>`_.
The URL for the Kubeflow dashboard is the same as the one determined earlier for the configuration steps - in the case of a default MicroK8s install, it’s: http://10.64.140.43.nip.io

From a browser on your local machine, this can be reached just by copying and pasting the URL. You should then see the dex login screen, where you should enter the username( it does say email address, but whatever string you entered to configure it will work fine) and your password from the configuration step.

However, for remote deployments, or running on a virtual machine, creating a SOCKS proxy is required to access the dashboard. This can be done as follows:

1. Connection to the machine using ssh with SOCKS proxy enabled through the -D 1080 parameter. As in the example below:

.. code-block:: shell

    ssh -D localhost:1080 vmware@<vm_machine_public_ip>

2. Go to the browser on your computer, go to Settings > Network > Network Proxy, and enable SOCKS proxy pointing to: 127.0.0.1:1080. If it's firfox browser, the setting is as below:

.. image:: ../_static/install-firfox-socket-setting.png

If it's chrome browser, the setting is as below:

.. image:: ../_static/install-chrome-socket-setting.png

You should now see the Kubeflow “Welcome” page:

.. image:: ../_static/install-welcome.png

Click on the “Start Setup” button. On the next screen you will be asked to create a namespace. This is just a way of keeping all the files and settings from one project in a single, easy-to-access place. You can choose any name you like…

.. image:: ../_static/install-namespace.png

Once you click on the “Finish” button, the Dashboard will be displayed!

.. image:: ../_static/install-dashboard.png


Reference
=========

.. seealso::

    `Get started with Charmed Kubeflow <https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow>`_
