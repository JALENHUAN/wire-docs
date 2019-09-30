.. _helm-prod:

**************************************************
Installing wire-server (production) components
**************************************************

WARNING
===========

TODO: THIS PAGE IS WORK-IN-PROGRESS

Introduction
=============

This contains instructions towards a more production-ready setup.
Depending on your use-case and requirements, you may only need to
configure a subset of the following sections.

It is *strongly recommended* to have followed the demo installation :ref:`helm` before continuing with this page.

Part 1 deals with installing components *outside* kubernetes, such as databases, and makes use of ansible.

Part 2 is similar to the demo installation, and uses helm to install so-called "charts" on kubernetes.

Part 3 details other possible configuration options.

Prerequisites
===============

Part 1 - VM installation
=========================

TODO coming soon

Part 2 - helm chart installation
===================================

TODO coming soon





Part 3 - Additional requirements recommended for a production setup
====================================================================

-  more server resources to ensure
   `high-availability <#persistence-and-high-availability>`__
-  an email/SMTP server to send out registration emails
-  depending on your required functionality, you may or may not need an
   `AWS account <https://aws.amazon.com/>`__. See details about
   limitations without an AWS account in the following sections.
-  one or more people able to maintain the installation
-  official support by Wire (`contact us <https://wire.com/pricing/>`__)


SMTP server
-----------

**Assumptions**: none

**Provides**:

-  full control over email sending

**You need**:

-  SMTP credentials (to allow for email sending; prerequisite for
   registering users and running the smoketest)

**How to configure**:

-  *if using a gmail account, ensure to enable* `'less secure
   apps' <https://support.google.com/accounts/answer/6010255?hl=en>`__
-  Add user, SMTP server, connection type to ``values/wire-server``'s
   values file under ``brig.config.smtp``
-  Add password in ``secrets/wire-server``'s secrets file under
   ``brig.secrets.smtpPassword``

Load balancer on bare metal servers
-----------------------------------

**Assumptions**:

-  You installed kubernetes on bare metal servers or virtual machines
   that can bind to a public IP address.
-  **If you are using AWS or another cloud provider, see**\ `Creating a
   cloudprovider-based load
   balancer <#load-balancer-on-cloud-provider>`__\ **instead**

**Provides**:

-  Allows using a provided Load balancer for incoming traffic
-  SSL termination is done on the ingress controller
-  You can access your wire-server backend with given DNS names, over
   SSL and from anywhere in the internet

**You need**:

-  A kubernetes node with a *public* IP address (or internal, if you do
   not plan to expose the Wire backend over the Internet but we will
   assume you are using a public IP address)
-  DNS records for the different exposed addresses (the ingress depends
   on the usage of virtual hosts), namely:

   -  ``nginz-https.<domain>``
   -  ``nginz-ssl.<domain>``
   -  ``assets.<domain>``
   -  ``webapp.<domain>``
   -  ``account.<domain>``
   -  ``teams.<domain>``

-  A wildcard certificate for the different hosts (``*.<domain>``) - we
   assume you want to do SSL termination on the ingress controller

**Caveats**:

-  Note that there can be only a *single* load balancer, otherwise your
   cluster might become
   `unstable <https://metallb.universe.tf/installation/>`__

**How to configure**:

::

   cp values/metallb/demo-values.example.yaml values/metallb/demo-values.yaml
   cp values/nginx-lb-ingress/demo-values.example.yaml values/nginx-lb-ingress/demo-values.yaml
   cp values/nginx-lb-ingress/demo-secrets.example.yaml values/nginx-lb-ingress/demo-secrets.yaml

-  Adapt ``values/metallb/demo-values.yaml`` to provide a list of public
   IP address CIDRs that your kubernetes nodes can bind to.
-  Adapt ``values/nginx-lb-ingress/demo-values.yaml`` with correct URLs
-  Put your TLS cert and key into
   ``values/nginx-lb-ingress/demo-secrets.yaml``.

Install ``metallb`` (for more information see the
`docs <https://metallb.universe.tf>`__):

.. code:: sh

   helm upgrade --install --namespace metallb-system metallb wire/metallb \
       -f values/metallb/demo-values.yaml \
       --wait --timeout 1800

Install ``nginx-lb-ingress``:

::

   helm upgrade --install --namespace demo demo-nginx-lb-ingress wire/nginx-lb-ingress \
       -f values/nginx-lb-ingress/demo-values.yaml \
       -f values/nginx-lb-ingress/demo-secrets.yaml \
       --wait

Now, create DNS records for the URLs configured above.

Load Balancer on cloud-provider
-------------------------------

AWS
~~~

`Upload the required
certificates <https://aws.amazon.com/premiumsupport/knowledge-center/import-ssl-certificate-to-iam/>`__.
Create and configure ``values/aws-ingress/demo-values.yaml`` from the
examples.

::

   helm upgrade --install --namespace demo demo-aws-ingress wire/aws-ingress \
       -f values/aws-ingress/demo-values.yaml \
       --wait

To give your load balancers public DNS names, create and edit
``values/external-dns/demo-values.yaml``, then run
`external-dns <https://github.com/helm/charts/tree/master/stable/external-dns>`__:

::

   helm repo update
   helm upgrade --install --namespace demo demo-external-dns stable/external-dns \
       --version 1.7.3 \
       -f values/external-dns/demo-values.yaml \
       --wait

Things to note about external-dns:

-  There can only be a single external-dns chart installed (one per
   kubernetes cluster, not one per namespace). So if you already have
   one running for another namespace you probably don't need to do
   anything.
-  You have to add the appropriate IAM permissions to your cluster (see
   the
   `README <https://github.com/helm/charts/tree/master/stable/external-dns>`__).
-  Alternatively, use the AWS route53 console.

Other cloud providers
~~~~~~~~~~~~~~~~~~~~~

This information is not yet available. If you'd like to contribute by
adding this information for your cloud provider, feel free to read the
`contributing guidelines <../CONTRIBUTING.md>`__ and open a PR.

Real AWS services
-----------------

**Assumptions**:

-  You installed kubernetes and wire-server on AWS

**Provides**:

-  Better availability guarantees and possibly better functionality of
   AWS services such as SQS and dynamoDB.
-  You can use ELBs in front of nginz for higher availability.
-  instead of using a smtp server and connect with SMTP, you may use
   SES. See configuration of brig and the ``useSES`` toggle.

**You need**:

-  An AWS account

**How to configure**:

-  Instead of using fake-aws charts, you need to set up the respective
   services in your account, create queues, tables etc. Have a look at
   the fake-aws-\* charts; you'll need to replicate a similar setup.

   -  Once real AWS resources are created, adapt the configuration in
      the values and secrets files for wire-server to use real endpoints
      and real AWS keys. Look for comments including
      ``if using real AWS``.

-  Creating AWS resources in a way that is easy to create and delete
   could be done using either `terraform <https://www.terraform.io/>`__
   or `pulumi <https://pulumi.io/>`__. If you'd like to contribute by
   creating such automation, feel free to read the `contributing
   guidelines <../CONTRIBUTING.md>`__ and open a PR.

Persistence and high-availability
---------------------------------

Currently, due to the way kubernetes and cassandra
`interact <https://github.com/kubernetes/kubernetes/issues/28969>`__,
cassandra cannot reliably be installed on kubernetes. Some people have
tried, e.g. `this
project <https://github.com/instaclustr/cassandra-operator>`__ though at
the time of writing (Nov 2018), this does not yet work as advertised. We
recommend therefore to install cassandra, (possibly also elasticsearch
and redis) separately, i.e. outside of kubernetes (using 3 nodes each).

For further higher-availability:

-  scale your kubernetes cluster to have separate etcd and master nodes
   (3 nodes each)
-  use 3 instead of 1 replica of each wire-server chart

Security
--------

For a production deployment, you should, as a minimum:

-  Ensure traffic between kubernetes nodes, etcd and databases are
   confined to a private network
-  Ensure kubernetes API is unreachable from the public internet (e.g.
   put behind VPN/bastion host or restrict IP range) to prevent
   `kubernetes
   vulnerabilities <https://www.cvedetails.com/vulnerability-list/vendor_id-15867/product_id-34016/Kubernetes-Kubernetes.html>`__
   from affecting you
-  Ensure your operating systems get security updates automatically
-  Restrict ssh access / harden sshd configuration
-  Ensure no other pods with public access than the main ingress are
   deployed on your cluster, since, in the current setup, pods have
   access to etcd values (and thus any secrets stored there, including
   secrets from other pods)
-  Ensure developers encrypt any secrets.yaml files

Additionally, you may wish to build, sign, and host your own docker
images to have increased confidence in those images. We haved "signed
container images" on our roadmap.

Sign up with a phone number (Sending SMS)
-----------------------------------------

**Provides**:

-  Registering accounts with a phone number

**You need**:

-  a `Nexmo <https://www.nexmo.com/>`__ account
-  a `Twilio <https://www.twilio.com/>`__ account

**How to configure**:

See the ``brig`` chart for configuration.

.. _3rd-party-proxying:

3rd-party proxying
------------------

You need Giphy/Google/Spotify/Soundcloud API keys (if you want to
support previews by proxying these services)

See the ``proxy`` chart for configuration.

TURN servers (Audio/Video calls)
--------------------------------

Not yet supported.

Metrics/logging
---------------

* :ref:`monitoring`
* :ref:`logging`

--------------------


Status
------

Code in this repository should be considered **beta**. We do not (yet)
run our production infrastructure on kubernetes.

Supported features:

-  wire-server (API)

   -  [x] user accounts, authentication, conversations
   -  [x] assets handling (images, files, ...)
   -  [x] (disabled by default) 3rd party proxying
   -  [x] notifications over websocket
   -  [ ] notifications over
      `FCM <https://firebase.google.com/docs/cloud-messaging/>`__/`APNS <https://developer.apple.com/notifications/>`__
      push notifications
   -  [x] audio/video calling servers using :ref:`understand-restund`)

-  wire-webapp

   -  [x] fully functioning web client (like ``https://app.wire.com``)

-  wire-account-pages

   -  [x] user account management (a few pages relating to e.g. password reset)

-  wire-team-settings

   -  [x] team management (including invitations, requires access to a
      private repository)

Prerequisites
-------------

As a minimum for a demo installation, you need:

-  a **Kubernetes cluster** with enough resources. There are `many
   different
   options <https://kubernetes.io/docs/setup/pick-right-solution/>`__. A
   tiny subset of those solutions we tried include:

   -  if using AWS, you may want to look at:

      -  `EKS <https://aws.amazon.com/eks/>`__ (if you're okay having
         all your data in one of the EKS-supported US regions)
      -  `kops <https://github.com/kubernetes/kops>`__

   -  if using regular physical or virtual servers:

      -  `kubespray <https://github.com/kubernetes-incubator/kubespray>`__

-  a **Domain Name** under your control and the ability to set DNS
   entries
-  the ability to generate **SSL certificates** for that domain name

   -  you could use e.g. `Let's Encrypt <https://letsencrypt.org/>`__

Required server resources
~~~~~~~~~~~~~~~~~~~~~~~~~

-  For an ephemeral in-memory demo-setup

   -  a single server with 8 CPU cores, 32GB of memory, and 20GB of disk
      space is sufficient.

-  For a production setup, you need at least 3 servers. For an optimal
   setup, more servers are required, it depends on your environment.

Contents of this repository
---------------------------

-  ``bin/`` - some helper bash scripts
-  ``charts/`` - so-called "`helm <https://www.helm.sh/>`__ charts" -
   templated kubernetes configuration in YAML
-  ``docs/`` - further documentation
-  ``values/`` - example override values to helm charts

Development setup
-----------------

You need to install

-  `helm <https://docs.helm.sh/using_helm/#installing-helm>`__ (v2.11.x
   is known to work)
-  `kubectl <https://kubernetes.io/docs/tasks/tools/install-kubectl/>`__
   (v1.12.x is known to work)

and you need to configure access to a kubernetes cluster (minimum v1.9+,
1.12+ recommended).

For any of the listed ``helm install`` or ``helm upgrade`` commands in
the documentation, you must first enable the wire charts helm repo (a
mirror of this github repository hosted publicly on AWS's S3)

.. code:: shell

   helm repo add wire https://s3-eu-west-1.amazonaws.com/public.wire.com/charts

(You can see available charts by running ``helm search wire/``. To see
new versions as time passes, you may need to run ``helm repo update``)

Optionally, if working in a team and you'd like to share
``secrets.yaml`` files between developers using a private git repository
and encrypted files, you may wish to install

-  `sops <https://github.com/mozilla/sops>`__
-  `helm-secrets
   plugin <https://github.com/futuresimple/helm-secrets>`__

If you're a maintainer of wire-server-deploy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

see `maintainers.md <docs/maintainers.md>`__

Installing wire-server
----------------------

Demo installation
~~~~~~~~~~~~~~~~~

-  AWS account not required
-  Requires only a kubernetes cluster

The demo setup is the easiest way to install a functional wire-server
with limitations (such as no persistent storage, no high-availability,
missing features). For the purposes of this demo, we assume you **do not
have an AWS account**. Try this demo first before trying to configure a
more complicated setup involving persistence and higher availability.

(Alternatively, you can run replace ``wire/<chart>`` with
``charts/<chart>`` in all subsequent commands. This will read charts
from your local file system. Make sure your working directory is the
root of this repo, and that after changing any of the chart files, you
run ``./bin/update.sh`` on them.)

*For all the following ``helm upgrade`` commands, it can be useful to
run a second terminal with ``kubectl --namespace demo get pods -w`` to
see what's happening.*

Install non-persistent, non-highly-available databases
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Please note that this setup is for demonstration purposes; no data is
ever written to disk, so a restart will wipe data. Even without restarts
expect it to be unstable: you may experience total service
unavailability and/or*\ **total data loss after a few hours/days**\ *due
to the way kubernetes and
cassandra*\ `interact <https://github.com/kubernetes/kubernetes/issues/28969>`__\ *.
For more information on this see the production installation section.*

The following will install (or upgrade) 3 single-pod databases and 3
ClusterIP services to reach them:

-  **databases-ephemeral**

   -  cassandra-ephemeral
   -  elasticsearch-ephemeral
   -  redis-ephemeral

.. code:: shell

   helm upgrade --install --namespace demo demo-databases-ephemeral wire/databases-ephemeral --wait

To delete: ``helm delete --purge demo-databases-ephemeral``

Install AWS service mocks
^^^^^^^^^^^^^^^^^^^^^^^^^

The code in wire-server still depends on some AWS services for some of
its functionality. To ensure wire-server services can correctly start
up, install the following "fake" (limited-functionality, non-HA) aws
services:

-  **fake-aws**

   -  fake-aws-sqs
   -  fake-aws-sns
   -  fake-aws-s3
   -  fake-aws-dynamodb

.. code:: shell

   helm upgrade --install --namespace demo demo-fake-aws wire/fake-aws --wait

To delete: ``helm delete --purge demo-fake-aws``

Install a demo SMTP server
^^^^^^^^^^^^^^^^^^^^^^^^^^

You can either install this very basic SMTP server, or configure your
own (see SMTP options in `this
section <docs/configuration.md#smtp-server>`__)

.. code:: shell

   helm upgrade --install --namespace demo demo-smtp wire/demo-smtp --wait

Install wire-server
^^^^^^^^^^^^^^^^^^^

-  **wire-server**

   -  cassandra-migrations
   -  elasticsearch-index
   -  galley
   -  gundeck
   -  brig
   -  cannon
   -  nginz
   -  proxy (optional, disabled by default)
   -  spar (optional, disabled by default)
   -  webapp (optional, enabled by default)
   -  team-settings (optional, disabled by default - requires access to
      a private repository)
   -  account-pages (optional, disabled by default - requires access to
      a private repository)

Start by copying the necessary ``values`` and ``secrets`` configuration
files:

::

   cp values/wire-server/demo-values.example.yaml values/wire-server/demo-values.yaml
   cp values/wire-server/demo-secrets.example.yaml values/wire-server/demo-secrets.yaml

In ``values/wire-server/demo-values.yaml`` (referred to as
``values-file`` below) and ``values/wire-server/demo-secrets.yaml``
(referred to as ``secrets-file``), the following has to be adapted:

-  turn server shared key (needed for audio/video calling)

   -  Generate with e.g.
      ``openssl rand -base64 64 | env LC_CTYPE=C tr -dc a-zA-Z0-9 | head -c 42``
      or similar
   -  Add key to secrets-file under ``brig.secrets.turn.secret``
   -  (this will eventually need to be shared with a turn server, not
      part of this demo yet)

-  zauth private/public keys (For authentication; ``access tokens`` and
   ``user tokens`` (cookies) are signed and validated with these)

   -  Generate from within
      `wire-server <https://github.com/wireapp/wire-server>`__ with
      ``./dist/zauth -m gen-keypair -i 1`` if you have everything
      compiled; or alternatively with docker using
      ``docker run --rm quay.io/wire/alpine-intermediate /dist/zauth -m gen-keypair -i 1``
   -  add both to secrets-file under ``brig.zauth`` and the public one
      to secrets-file under ``nginz.secrets.zAuth.publicKeys``

-  domain names and urls

   -  in your values-file, replace ``example.com`` and other domains and
      subdomains with domains of your choosing. Look for the
      ``# change this`` comments. You can try using
      ``sed -i 's/example.com/<your-domain>/g' <values-file>``.

Try linting your chart, are any configuration values missing?

.. code:: sh

   helm lint -f values/wire-server/demo-values.yaml -f values/wire-server/demo-secrets.yaml wire/wire-server

If you're confident in your configuration, try installing it:

.. code:: sh

   helm upgrade --install --namespace demo demo-wire-server wire/wire-server \
       -f values/wire-server/demo-values.yaml \
       -f values/wire-server/demo-secrets.yaml \
       --wait

If pods fail to come up the ``helm upgrade`` may fail or hang; you may
wish to run ``kubectl get pods -n demo -w`` to see which pods are
failing to initialize. Describing the pods may provide information as to
why they're failing to initialize.

If installation fails you may need to delete the release
``helm delete --purge demo-wire-server`` and try again.

Adding a load balancer, DNS, and SSL termination
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

-  If you're on bare metal or on a cloud provider without external load
   balancer support, see `configuring a load balancer on bare metal
   servers <docs/configuration.md#load-balancer-on-bare-metal-servers>`__
-  If you're on AWS or another cloud provider, see `configuring a load
   balancer on cloud
   provider <docs/configuration.md#load-balancer-on-cloud-provider>`__

Beyond the demo
^^^^^^^^^^^^^^^

For further configuration options (some have specific requirements about
your environment), see
`docs/configuration.md <docs/configuration.md>`__.

Support with a production on-premise (self-hosted) installation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`Get in touch <https://wire.com/pricing/>`__.

Monitoring
----------

See the `monitoring guide <./docs/monitoring.md>`__

Troubleshooting
---------------

There are multiple artifacts which combine to form a running wire-server
deployment; these include:

-  docker images for each service
-  kubernetes configs for each deployment (from helm charts)
-  configuration maps for each deployment (from helm charts)

If you wish to get some information regarding the code currently running
on your cluster you can run the following:

::

   ./bin/deployment-info.sh <namespace> <deployment-name (e.g. brig)>

Example run:

::

   ./deployment-info.sh demo brig
   docker_image:               quay.io/wire/brig:2.50.319
   chart_version:              wire-server-0.24.9
   wire_server_commit:         8ec8b7ce2e5a184233aa9361efa86351c109c134
   wire_server_link:           https://github.com/wireapp/wire-server/releases/tag/image/2.50.319
   wire_server_deploy_commit:  01e0f261ca8163e63860f8b2af6d4ae329a32c14
   wire_server_deploy_link:    https://github.com/wireapp/wire-server-deploy/releases/tag/chart/wire-server-0.24.9

Note you'll need ``kubectl``, ``git`` and ``helm`` installed

It will output the running docker image; the corresponding wire-server
commit hash (and link) and the wire-server helm chart version which is
running.


