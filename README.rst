Kubernetes Demo
===============

Prerequisites
-------------

* Download and install the Google Cloud SDK: https://cloud.google.com/sdk/install
* Download and install ``kubectl``: https://kubernetes.io/docs/tasks/tools/install-kubectl/
* Login to Google Cloud and install credentials for connecting to the cluster::

    $ gcloud auth login
    $ gcloud beta container clusters get-credentials kubedemo-cluster --region us-east1 --project kubernetes-lighting-talk

* Verify you can connect to the cluster::

    $ kubectl cluster-info
    Kubernetes master is running at https://w.x.y.z
    ...

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


Lab 1: Creating a Namespace
---------------------------

Namespaces facilitate logical separation of resources in a shared cluster. For example,
you might have a **projectname-staging** namespace and a **projectname-production**
namespace. For now, just create one with your username::

    $ kubectl create namespace tobias

Now, let's update our kubectl configuration to use this namespace by default::

    $ kubectl config set-context --current --namespace=tobias

.. tip::
    There is much more we're not covering about managing configurations for different clusters.
    See the relevant section of the `Kubernetes Cheat Sheet
    <https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-context-and-configuration>`_,
    or poke around in ``~/.kube/config`` yourself to understand how the file is constructed.

Lab 2: Creating a Deployment and Connecting to a Pod
----------------------------------------------------

Let's get some code running in our new namespace. Start a file named ``bakerydemo.yaml``
in your current directory and add the following to it:

.. code:: yaml

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: bakerydemo
    spec:
      # Attempt to keep a single instance of this Pod running at all times
      replicas: 1
      template:
        # This is the Pod definition!
        metadata:
          labels:
            app: bakerydemo
        spec:
          containers:
          - name: bakerydemo
            # The user, repository, and tag of the image we're deploying
            image: wagtail/bakerydemo:latest
            # On which port(s) does this container listen?
            ports:
            - containerPort: 8000
            # Kubernetes boilerplate (how to lookup Pod hostnames):
            env:
            - name: GET_HOSTS_FROM
              value: dns

Now, save the file and run::

    $ kubectl apply -f bakerydemo.yaml

This sends our "object model" for a single ``Deployment`` to the cluster configured in
``~/.kube/config``. Let's see if it worked::

    $ kubectl get pods
    NAME                          READY   STATUS    RESTARTS   AGE
    YOUR_POD_NAME                 1/1     Running   0          1m

(Where you see ``<YOUR_POD_NAME>`` below, fill it in with the actual name you see when
running ``kubectl get pods``.)

Assuming that worked, let's look a little closer at the Pod::

    $ kubectl describe pod <YOUR_POD_NAME>
    <snip>
    Events:
      Type    Reason     Age   From                                                 Message
      ----    ------     ----  ----                                                 -------
      Normal  Scheduled  17s   default-scheduler                                    Successfully assigned tobias/bakerydemo-6fbb6fc759-7bpxt to gke-kubedemo-cluster-default-c152b5f2-7d05
      Normal  Pulling    16s   kubelet, gke-kubedemo-cluster-default-c152b5f2-7d05  Pulling image "wagtail/bakerydemo:latest"
      Normal  Pulled     15s   kubelet, gke-kubedemo-cluster-default-c152b5f2-7d05  Successfully pulled image "wagtail/bakerydemo:latest"
      Normal  Created    15s   kubelet, gke-kubedemo-cluster-default-c152b5f2-7d05  Created container bakerydemo
      Normal  Started    15s   kubelet, gke-kubedemo-cluster-default-c152b5f2-7d05  Started container bakerydemo

We can also look at the logs for the Pod::

    $ kubectl logs <YOUR_POD_NAME>
    psql: could not connect to server: No such file or directory
      Is the server running locally and accepting
      connections on Unix domain socket "/var/run/postgresql/.s.PGSQL.5432"?
    Postgres is unavailable - sleeping

We can even start a shell inside the running container and poke around::

    $ kubectl exec -it <YOUR_POD_NAME> -- /bin/bash
    # ps aux
    USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    root           1  0.0  0.0   2388  1560 ?        Ss   18:29   0:00 /bin/sh /code/docker-entrypoint.sh /venv/bin/uwsgi --show-config
    root        3585  0.0  0.0   5752  3636 pts/0    Ss   18:35   0:00 /bin/bash
    root        3598  0.0  0.0   4048   752 ?        S    18:35   0:00 sleep 1
    root        3599  0.0  0.0   9392  3104 pts/0    R+   18:35   0:00 ps aux

.. tip::
    There are many more useful commands to learn for interacting with Pods, too. Check out the relevant
    section of the `Kubernetes Cheat Sheet
    <https://kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-running-pods>`_.
