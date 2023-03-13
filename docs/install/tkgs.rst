======================================
Install Kubeflow on vSphere with Tanzu
======================================

This document will guide you to install Kubeflow on vSphere with Tanzu.

Prerequisites
=============

Adhere to the following requirements before deploying Kubeflow package on TKGS clusters.

For the deployment on vSphere with Tanzu (TKGS), Kubeflow will be installed on a Tanzu Kubernetes Cluster (TKC). So before the deployment of Kubeflow, you should get vSphere and TKC ready.

- For a greenfield deployment (no vSphere with Tanzu deployed on servers yet), you will need to deploy vSphere with Tanzu first. Please refer to VMware official document `vSphere with Tanzu Configuration and Management <https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-152BE7D2-E227-4DAA-B527-557B564D9718.html>`__.

- With the proper vSphere with Tanzu installation and configuration, provison TKGS clusters. See `Workflow for Provisioning Tanzu Kubernetes Clusters Using the TKGS v1alpha2 API <https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-3040E41B-8A54-4D23-8796-A123E7CAE3BA.html>`__.

- If you want to use GPU resources on vSphere platform, setup vGPU TKG with the following document `Deploy AI/ML Workloads on Tanzu Kubernetes Clusters <https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-2B4CAE86-BAF4-4411-ABB1-D5F2E9EF0A3D.html>`__.

- Connect to the cluster from your client host. See `Connect to a Tanzu Kubernetes Cluster as a vCenter Single Sign-On User <https://docs.vmware.com/en/VMware-vSphere/7.0/vmware-vsphere-with-tanzu/GUID-AA3CA6DC-D4EE-47C3-94D9-53D680E43B60.html>`__

- Install kctrl, a kapp-controller's native CLI on your client host. It will be used to install Kubeflow Carvel Package. See `Installing kapp-controller CLI: kctrl <https://carvel.dev/kapp-controller/docs/v0.40.0/install/#installing-kapp-controller-cli-kctrl>`__

Deploy Kubeflow Package on TKGS Clusters
========================================

1. Add PackageRepository (use kubectl instead?)

   .. code-block:: shell

      kubectl create ns carvel-kubeflow-namespace
      kubectl config set-context --current --namespace=carvel-kubeflow-namespace

      kctrl package repository add \
	  --repository kubeflow-carvel-repo \
	  --url projects.registry.vmware.com/kubeflow/kubeflow-carvel-repo:0.11

2. Show the values schema of Kubeflow package (use the config table below?)

   .. code-block:: shell

      $ kctrl package available get -p kubeflow.community.tanzu.vmware.com/1.6.0 --values-schema

      Target cluster 'https://10.117.233.12:6443' (nodes: tkgs-ubucluster-carvel-testing-control-plane-87649, 2+)

      Values schema for 'kubeflow.community.tanzu.vmware.com/1.6.0'

      Key                                  Default                                                                 Type     Description  
      CD_REGISTRATION_FLOW                 true                                                                    boolean  Turn on Registration Flow, so that Kubeflow Central Dashboard will prompt new users to create a namespace (profile)  
      Dex.config                           |-                                                                      string   Configuration file of Dex  
					     issuer: http://dex.auth.svc.cluster.local:5556/dex                               
					     storage:                                                                         
					       type: kubernetes                                                               
					       config:                                                                        
						 inCluster: true                                                              
					     web:                                                                             
					       http: 0.0.0.0:5556                                                             
					     logger:                                                                          
					       level: "debug"                                                                 
					       format: text                                                                   
					     oauth2:                                                                          
					       skipApprovalScreen: true                                                       
					     enablePasswordDB: true                                                           
					     staticPasswords:                                                                 
					     - email: user@example.com                                                        
					       hash: $2y$12$4K/VkmDd1q1Orb3xAt82zu8gk7Ad6ReFR4LCP9UeYE90NLiN9Df72             
					       # https://github.com/dexidp/dex/pull/1601/commits                              
					       # FIXME: Use hashFromEnv instead                                               
					       username: user                                                                 
					       userID: "15841185641784"                                                       
					     staticClients:                                                                   
					     # https://github.com/dexidp/dex/pull/1664                                        
					     - idEnv: OIDC_CLIENT_ID                                                          
					       redirectURIs: ["/login/oidc"]                                                  
					       name: 'Dex Login Application'                                                  
					       secretEnv: OIDC_CLIENT_SECRET                                                  
      Dex.use_external                     false                                                                   boolean  If set to True, the embedded Dex will not be created, and you will need to configure OIDC_Authservice with external IdP manually  
      IP_address                           ""                                                                      string   EXTERNAL_IP address of istio-ingressgateway, valid only if service_type is LoadBalancer  
      OIDC_Authservice.OIDC_AUTH_URL       /dex/auth                                                               string   AuthService will initiate an Authorization Code OIDC flow by hitting this URL. Normally discovered automatically through the OIDC Provider's well-known endpoint  
      OIDC_Authservice.OIDC_CLIENT_ID      kubeflow-oidc-authservice                                               string   AuthService will use this Client ID when it needs to contact your OIDC provider and initiate an OIDC flow  
      OIDC_Authservice.OIDC_CLIENT_SECRET  pUBnBOY80SnXgjibTYM9ZWNzY2xreNGQok                                      string   AuthService will use this Client Secret to authenticate itself against your OIDC provider in combination with CLIENT_ID when attempting to access your OIDC Provider's protected endpoints  
      OIDC_Authservice.OIDC_PROVIDER       http://dex.auth.svc.cluster.local:5556/dex                              string   URL to your OIDC provider. AuthService expects to find information about your OIDC provider at OIDC_PROVIDER/.well-known/openid-configuration, and will use this information to contact your OIDC provider and initiate an OIDC flow later on  
      OIDC_Authservice.OIDC_SCOPES         profile email groups                                                    string   Comma-separated list of scopes to request access to. The openid scope is always added.  
      OIDC_Authservice.REDIRECT_URL        /login/oidc                                                             string   AuthService will pass this URL to the OIDC provider when initiating an OIDC flow, so the OIDC provider knows where it needs to send the OIDC authorization code to. It defaults to AUTHSERVICE_URL_PREFIX/oidc/callback. This assumes that you have configured your API Gateway to pass all requests under a hostname to Authservice for authentication  
      OIDC_Authservice.SKIP_AUTH_URI       /dex                                                                    string   Comma-separated list of URL path-prefixes for which to bypass authentication. For example, if SKIP_AUTH_URL contains /my_app/ then requests to <url>/my_app/* are allowed without checking any credentials. Contains nothing by default  
      OIDC_Authservice.USERID_CLAIM        email                                                                   string   Claim whose value will be used as the userid (default email)  
      OIDC_Authservice.USERID_HEADER       kubeflow-userid                                                         string   Name of the header containing the user-id that will be added to the upstream request  
      OIDC_Authservice.USERID_PREFIX       ""                                                                      string   Prefix to add to the userid, which will be the value of the USERID_HEADER  
      imageswap_labels                     true                                                                    boolean  Add labels k8s.twr.io/imageswap: enabled to Kubeflow namespaces, which enable imageswap webhook to swap images.  
      service_type                         LoadBalancer                                                            string   Service type of istio-ingressgateway. Available options: "LoadBalancer" or "NodePort"  

      Succeeded

3. Create a values.yaml files according to values schema

   .. code-block:: shell

      cat > values.yaml << 'EOF'

      imageswap_labels: True

      service_type: "LoadBalancer"

      IP_address: ""

      CD_REGISTRATION_FLOW: True

      OIDC_Authservice:
	OIDC_AUTH_URL: /dex/auth
	OIDC_PROVIDER: http://dex.auth.svc.cluster.local:5556/dex
	OIDC_SCOPES: "profile email groups"
	REDIRECT_URL: /login/oidc
	SKIP_AUTH_URI: "/dex"
	USERID_CLAIM: email
	USERID_HEADER: kubeflow-userid
	USERID_PREFIX: ""
	OIDC_CLIENT_ID: kubeflow-oidc-authservice
	OIDC_CLIENT_SECRET: pUBnBOY80SnXgjibTYM9ZWNzY2xreNGQok

      Dex:
	use_external: False
	config: |
	  issuer: http://dex.auth.svc.cluster.local:5556/dex
	  storage:
	    type: kubernetes
	    config:
	      inCluster: true
	  web:
	    http: 0.0.0.0:5556
	  logger:
	    level: "debug"
	    format: text
	  oauth2:
	    skipApprovalScreen: true
	  enablePasswordDB: true
	  staticPasswords:
	  - email: user@example.com
	    hash: $2y$12$4K/VkmDd1q1Orb3xAt82zu8gk7Ad6ReFR4LCP9UeYE90NLiN9Df72
	    # https://github.com/dexidp/dex/pull/1601/commits
	    # FIXME: Use hashFromEnv instead
	    username: user
	    userID: "15841185641784"
	  staticClients:
	  # https://github.com/dexidp/dex/pull/1664
	  - idEnv: OIDC_CLIENT_ID
	    redirectURIs: ["/login/oidc"]
	    name: 'Dex Login Application'
	    secretEnv: OIDC_CLIENT_SECRET

      EOF

4. Install Kubeflow package

   .. code-block:: shell

      kctrl package install \
	  --wait-check-interval 5s \
	  --wait-timeout 30m0s \
	  --package-install kubeflow \
	  --package kubeflow.community.tanzu.vmware.com \
	  --version 1.6.0 \
	  --values-file values.yaml

Trouble shooting
================

Delete the Kubeflow Package
^^^^^^^^^^^^^^^^^^^^^^^^^^^

- Uninstall installed package.

   .. code-block:: shell

      kctrl package installed delete --package-install kubeflow

- When deleting the Kubeflow package, Some resources got stuck at deleting status.

   .. code-block:: shell

      # take namespace knative-serving as an example
      kubectl patch ns knative-serving -p '{"spec":{"finalizers":null}}'
      kubectl delete ns knative-serving --grace-period=0 --force

Reconciliating Issue
^^^^^^^^^^^^^^^^^^^^

Kapp-controller keeps reconciliating Kubeflow, which prevent you from editing a Kubeflow resource. In this case, you may want to pause or trigger of Kubeflow.


1. Pausing reconciliation for a package install

   .. code-block:: shell

      kctrl package installed pause --package-install kubeflow

2. Trigger reconciliation of package install

   .. code-block:: shell

      kctrl package installed kick --package-install kubeflow --wait --wait-check-interval 5s --wait-timeout 30m0s

Debug
^^^^^

- Check PackageInstall Status

   .. code-block:: shell

      kubectl get PackageInstall kubeflow -o yaml

- Print status of app created by package installation

   .. code-block:: shell

     kctrl package installed status --package-install kubeflow

- Update to an updated values file

   .. code-block:: shell

      kctrl package installed update --package-install kubeflow --values-file values.yaml

Configure the Kubeflow Package
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

========= =========== ===== =======
Parameter Description Type  Default
========= =========== ===== =======
False     False       False xxx
True      False       False yyy
False     True        False zzz
True      True        True  zzz
========= =========== ===== =======

.. seealso::

    `Get started with Charmed Kubeflow <https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow>`_
