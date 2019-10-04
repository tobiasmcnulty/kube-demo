Kubernetes Demo
===============

Prerequisites
-------------

* Download and install the Google Cloud SDK: https://cloud.google.com/sdk/install
* Download and install ``kubectl``: https://kubernetes.io/docs/tasks/tools/install-kubectl/
* Login to Google Cloud and install credentials for connecting to the cluster::

    gcloud auth login
    gcloud beta container clusters get-credentials kubedemo-cluster --region us-east1 --project kubernetes-lighting-talk

Alternate (non-Google) instructions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you don't have a Google account, you can try using the free service at kubesail.com. But its
memory restrictions might stop your containers from running unexpectedly.

* Sign up for a free account (GitHub login recommended) at https://kubesail.com
* Go to the Account page and follow the instructions to create your ``~/.kube/config`` file
* Ensure you can run and see something like the following::

    $ kubectl get resourcequota -o yaml
    apiVersion: v1
    items:
    - apiVersion: v1
    kind: ResourceQuota
    metadata:
        creationTimestamp: "2019-09-10T23:55:59Z"
        name: tobiasmcnulty-quota
        namespace: tobiasmcnulty
        resourceVersion: "44468880"
    <snip>
