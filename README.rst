Kubernetes Demo
===============

Prerequisites
-------------

* Download and install the Google Cloud SDK: https://cloud.google.com/sdk/install
* Download and install ``kubectl``: https://kubernetes.io/docs/tasks/tools/install-kubectl/ (please ensure you
  install v1.15 or later; v1.16 is current as of October, 2019)
* If you have a Google account, login to Google Cloud::

      $ gcloud auth login

* **If you don't have a Google account, temporary credentials will be provided during the talk.**
* After you've run ``gcloud auth login`` successfully, install credentials for connecting to the cluster::

    $ gcloud beta container clusters get-credentials kubedemo-cluster --region us-east1 --project kubernetes-lighting-talk

  Then, verify you can connect to the cluster::

    $ kubectl cluster-info
    Kubernetes master is running at https://w.x.y.z
    ...
