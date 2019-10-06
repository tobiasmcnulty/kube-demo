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

    # bakerydemo.yaml
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
            image: wagtail/bakerydemo:gcloud
            imagePullPolicy: Always
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

We'll come back to the Postgres error in a bit. e can even start a shell inside the running
container and poke around::

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

Lab 3: Configuration
--------------------

Let's give our Pod access to the managed Postgres instance we have set up in Google Cloud.

Open your ``bakerydemo.yaml`` file and prepend (or append, it doesn't matter) a new
YAML document for the Secret configuration, below.

**Important:**

* Additional YAML documents are separated by three dashes (``---``) on their own line in the
  file, so be sure to include those.
* Substitute the provided ``PASSWORD`` and ``DATABASE_NAME`` in your ``DATABASE_URL``.
* Change ``YOUR_USER_NAME`` in the ``GS_BUCKET_NAME`` variable to your username (or anything
  else to uniquely identify your Google Cloud Storage bucket, which will be created for you).

.. code:: yaml

    # bakerydemo.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: bakerydemo-secrets
      labels:
        app: bakerydemo
    type: Opaque
    stringData:
      DATABASE_URL: "postgres://demo:PASSWORD@10.63.96.3/DATABASE_NAME"
      DJANGO_SECRET_KEY: "a-long-and-random-string"
      # Bucket name must contain only lowercase letters, numbers, dashes (-), underscores (_),
      # and dots (.). See: https://cloud.google.com/storage/docs/naming
      GS_BUCKET_NAME: "YOUR_USER_NAME-doaf9j0uzq"  # must be globally unique, so add a few random characters
      GS_PROJECT_ID: "kubernetes-lighting-talk"  # [sic]
      # When using Jinja2 with Ansible (or another deployment tool), you could pull in
      # vault-encrypted variables, like so:
      # DJANGO_SECRET_KEY: "{{ DJANGO_SECRET_KEY }}"
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    # ...

You'll also need to add the following to the bottom of your ``Deployment``, with the same
indentation as ``env`` (this tells Kubernetes to load all the keys in our secret into
the enironment for the process):

.. code:: yaml

    # bakerydemo.yaml
            envFrom:
            - secretRef:
                name: bakerydemo-secrets

Apply these changes to the cluster::

    $ kubectl apply -f bakerydemo.yaml

Give it a few minutes to restart the pod, then get your new pod name and inspect the logs::

    $ kubectl get pods
    $ kubectl logs <YOUR_POD_NAME> --tail=10
    your mercy for graceful operations on workers is 60 seconds
    mapped 312672 bytes (305 KB) for 8 cores
    *** Operational MODE: preforking+threaded ***
    *** uWSGI is running in multiple interpreter mode ***
    spawned uWSGI master process (pid: 1)
    spawned uWSGI worker 1 (pid: 16, cores: 4)
    spawned uWSGI worker 2 (pid: 17, cores: 4)
    spawned uWSGI http 1 (pid: 18)
    WSGI app 0 (mountpoint='') ready in 2 seconds on interpreter 0x5612617dbc70 pid: 16 (default app)
    WSGI app 0 (mountpoint='') ready in 2 seconds on interpreter 0x5612617dbc70 pid: 17 (default app)

Hopefully you'll see that uwsgi has started. If not, try re-running the ``logs`` command a few times
and look for errors.

Let's load some initial data into the database with a Django management command::

    $ kubectl get pod
    $ kubectl exec -it <YOUR_POD_NAME> -- /venv/bin/python manage.py load_initial_data
    /venv/lib/python3.7/site-packages/dotenv.py:56: UserWarning: Not reading .env - it doesn't exist.
      warnings.warn("Not reading {0} - it doesn't exist.".format(dotenv))
    Awesome. Your data is loaded! The bakery's doors are almost ready to open...

**You may receive an error the first time this runs,** attempting to apply an ACL to the Google
Cloud Storage bucket. It's harmless; just run the same command again until you see the success
message above.

Lab 4: Accessing our app from the outside world
-----------------------------------------------

To access our app from the outside world, at minimum we need a ``Service`` object.
We're also going to create an ``Ingress`` object here, to help map a domain name
to our app and automatically generate a Let's Encrypt certificate for us.

Add the following to the end of ``bakerydemo.yaml`` (again, being careful to keep
a ``---`` between each YAML document):

.. code:: yaml

    ---
    # This Service makes our Pod(s) accessible with a static, private IP from WITHIN the cluster
    apiVersion: v1
    kind: Service
    metadata:
      name: bakerydemo
      labels:
        app: bakerydemo
    spec:
      # All pods with the 'app: bakerydemo' label are included in this Service!
      selector:
        app: bakerydemo
      ports:
      # Map port 80 to port 8000 on the Pod
      - protocol: TCP
        port: 80
        targetPort: 8000
    ---
    # This Ingress exposes our service to the outside world with a domain. Note,
    # this assumes the cluster as the Nginx Ingress Controller and a cert-manager
    # ClusterIssuer called "letsencrypt-production" already configured (at Caktus,
    # Tech Support will pre-configure the cluster like this for you).
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: bakerydemo
      annotations:
        kubernetes.io/ingress.class: nginx
        # If using kubesail.com, comment out this line:
        certmanager.k8s.io/cluster-issuer: "letsencrypt-production"
    spec:
      tls:
      - hosts:
        - YOUR_USER_NAME.kubedemo.caktus-built.com
        secretName: bakerydemo-tls
      rules:
      - host: YOUR_USER_NAME.kubedemo.caktus-built.com
        http:
          paths:
          - path: /
            backend:
              serviceName: bakerydemo
              servicePort: 80

I have wildcard DNS set up for this subdomain, so you can really pick anything that
matches ``*.kubedemo.caktus-built.com`` (and that doesn't conflict with someone else).

Re-apply our configuration and wait for the certificate to be generated::

    $ kubectl apply -f bakerydemo.yaml
    $ kubectl get pod
    NAME                          READY   STATUS    RESTARTS   AGE
    bakerydemo-76d45bdb7f-4mjbt   1/1     Running   0          41m
    cm-acme-http-solver-twnxt     1/1     Running   0          6s

If you're quick enough, you might notice the ``cm-acme-http-solver`` that was
created automatically by ``cert-manager`` to solve the Let's Encrypt challenge.
The pod will disappear once the certificate is issued (or if the pod sticks
around, that might indicate a problem).

Finally, navigate to https://YOUR_USER_NAME.kubedemo.caktus-built.com in your browser.

If you'd like a superuser account for yourself to login to the admin (at ``/admin/``),
you can create that the usual way as well::

    $ kubectl get pod
    $ kubectl exec -it <YOUR_POD_NAME> -- /venv/bin/python manage.py createsuperuser
    /venv/lib/python3.7/site-packages/dotenv.py:56: UserWarning: Not reading .env - it doesn't exist.
      warnings.warn("Not reading {0} - it doesn't exist.".format(dotenv))
    Username (leave blank to use 'root'): tobias
    Email address: tobias@...
    Password:
    Password (again):
    Superuser created successfully.

Good luck and have fun!
