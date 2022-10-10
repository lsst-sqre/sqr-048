:tocdepth: 1

.. sectnum::

Abstract
========

Default Kubernetes security settings for both clusters and pods are optimized for quick usability rather than security.
This document provides security hardening recommendations for GKE-hosted Kubernetes clusters and for Kubernetes resources managed by Rubin Observatory.
These recommendations are intended primarily for the Rubin Science Platform and for other SQuaRE-run services.

See SQR-037_ and SQR-041_ for more complete security risk assessments of the SQuaRE-run Kubernetes platforms.

.. _SQR-037: https://sqr-037.lsst.io/
.. _SQR-041: https://sqr-041.lsst.io/

Overview
========

This document is divided into two major sections: :ref:`Configuration for GKE clusters <gke>` and :ref:`recommendations for Kubernetes resources <resources>`.
The former provides recommendations specific to Google Kubernetes Engine (GKE) for the overall cluster configuration.
The latter is applicable to any Kubernetes hosting environment.

Some of the recommendations are not yet fleshed out and fully tested.
They will be updated as we gain more experience.

The overall goal of these recommendations is to gather together all security hardening that is unlikely to cause operational problems as a checklist of recommendations.
There is also some discussion of more intrusive hardening and where that hardening work may be appropriate.

.. _gke:

Google Kubernetes Engine
========================

Some of the hardening measures can only be set when creating a new cluster or node pool.
Setting all hardening options when creating the cluster is therefore easiest.

Baseline recommendations
------------------------

These recommendations have no prerequisites and can be put in place now.

#. Add the cluster to a release channel.
   This tells Google to automatically upgrade the control plane and node pools of the cluster as new versions of Kubernetes (including security patches) are released.
   Unlike some other release series, the ``stable`` release channel is the lagging release channel and is usually too old for our purposes.
   The ``regular`` release channel should be used for most clusters.
   Consider the ``rapid`` release channel for test clusters used to verify compatibility with new Kubernetes versions.
#. Enable `shielded GKE nodes <https://cloud.google.com/blog/products/identity-security/exploring-container-security-bringing-shielded-vms-to-gke-with-shielded-gke-nodes>`__ with secure boot.
   Shielded nodes add additional protections against an attacker modifying the boot process or kernel.
   Secure boot verifies that all kernels and kernel modules are cryptographically signed.
   Secure boot can only be enabled when creating a node pool and cannot be added later without recreating the pool.
#. Use the ``cos_containerd`` image for all nodes.
   This is a minimal root operating system dedicated to running containers.
   It minimizes the attack surface and available tools for an attacker attempting to break out of a container and compromise the host node.
   Clusters that run containers that need to load additional kernel modules may need a second node pool for those containers that uses a different image.
#. Enable network policy enforcement.
   Unless this is done, ``NetworkPolicy`` resources defined for pods (see :ref:`Kubernetes resources <resources>`) have no effect.

Advanced recommendations
------------------------

These recommendations require prerequisites or more complex configuration and will require more work to enable.

#. Create a `private cluster <https://cloud.google.com/kubernetes-engine/docs/concepts/private-cluster-concept>`__.
   This disables all Internet access to the cluster control plane and nodes.
   It is intended for use with Cloud VPN to connect to a corporate network, which we don't have and don't want to set up.
   However, it appears that we can create a VM in Google Compute Engine within a VPC that's peered with the cluster VPC and use that as a bastion host using SSH port forwarding.
   This significantly reduces the attack surface area of Kubernetes.
   Our nodes will generally require outbound access to the Internet, so this configuration will also require setting up `Cloud NAT <https://cloud.google.com/nat/docs/overview#NATwithGKE>`__.
#. Manage all cluster administrator accounts using Google Cloud Identity.
   This will require creating a Google Cloud Identity organization (or reusing the AURA G Suite and creating a sub-organization for the Rubin Science Platform).
   It will then provide a unified view of all Google users with cluster access and allow setting security policies such as mandatory two-factor authentication.
   It also allows access to Security Health Analytics in the console.
#. Manage cluster access with `Google Groups <https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control#google-groups-for-gke>`__.
   Setting up Google Cloud Identity is a prerequisite for doing this.
#. Create a custom service account with minimal permissions for the node pool rather than using the default Google Compute Engine service account.
   The GCE default service account has more permissions that is necessary for a GKE node.
   This must be done when creating the node pool.
   To allow human users to create new node pools and clusters, they need to be granted access to the service account, so this is best done in combination with enabling Google Groups for access control.
#. Restrict cluster discovery RBAC permissions.
   By default, Kubernetes allows access to cluster discovery APIs from any authenticated user, including users with no permissions on the project.
   Restrict the ``system:discovery`` and ``system:basic-user`` Kubernetes ``ClusterRoleBinding`` resources to ``system:serviceaccounts`` plus the user groups.
   This depends on setting up the cluster to use Google Groups authorization, and becomes unnecessary if the cluster is private.
#. Enable `Workload Identity <https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity>`__.
   This is an improved way to access other Google Cloud services from within Kubernetes that also resolves several security issues with Google Cloud metadata endpoints.
   Setup for this is relatively complicated.
   See `Google's instructions <https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#enable_on_new_cluster>`__ for more information.
#. Enable the ```PodSecurity`` admission controller <https://cloud.google.com/kubernetes-engine/docs/how-to/podsecurityadmission>`__ and configure Pod Security Standards policies for all services.
#. Deploy pods without strong performance requirements using the GKE Sandbox.
   This runs the pod with a special wrapper that intercepts all system calls and applies aggressive sandboxing, making a container escape much more difficult.
   The cost is a noticeable performance degredation for pods that make a lot of system calls.
   It's ideal for infrequently-used and less-trusted services.

Cluster creation command
------------------------

The following ``gcloud`` commands set up a new cluster following the basic recommendations.

.. code-block:: sh

   gcloud container clusters create --zone <zone>     \
       --release-channel=regular                      \
       --enable-shielded-nodes --shielded-secure-boot \
       --image-type=cos_containerd                    \
       --enable-network-policy

.. _resources:

Kubernetes resources
====================

The following recommendations for pod hardening assume that network policy enforcement is enabled on the cluster.
They are consistent with but do not assume use of workload identity or pod security policies.

Pod hardening
-------------

These hardening settings should be added to the ``Deployment``.
The context shown in the YAML excerpts is relative to the ``spec.template`` for the ``Deployment``.
Some settings should be done at the pod spec level and some should be at the container level.
The container settings must be repeated for each container in the pod, if there are several.

#. Disable mounting of the Kubernetes service token except in the rare cases where the service needs to make Kubernetes API calls.

   .. code-block:: yaml

      spec:
        automountServiceAccountToken: false

   If the service does need to make Kubernetes API calls, give it its own service account.
   Do not use the ``default`` service account for the namespace.
   Instead, create a new Kubernetes service account and project that service account into the pod, as `described in the Kubernetes documentation <https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/>`__.

#. Configure the application to run as a non-root user.
   Unfortunately, the UID and GID must be specified as numbers.
   The default UID and GID for a newly-created user in a Debian-based distribution is 1000, so using that number will match the recommended pattern of a ``Dockerfile`` that creates an app user and sets it as the user.

   .. code-block:: yaml

      spec:
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          runAsGroup: 1000

   For Docker images that manage persistent stores, such as databases, the convention appears to be to use 999 as the UID instead of 1000.
   Check the ``Dockerfile`` for the relevant service to be certain.
   For services with persistent stores, also set ``fsGroup`` to the same GID.
   This controls the group ownership of volumes mounted inside the pod.

#. Disable privilege escalation in containers.

   .. code-block:: yaml

      spec:
        containers:
          - name: <name>
            securityContext:
              allowPrivilegeEscalation: false

   Be aware that this will disable setuid binaries and binaries with capabilities.
   Most services will not need this, but there may be rare exceptions.

#. Drop all capabilities.
   Docker enables a `surprisingly large number of capabilities <https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities>`__ by default.
   These are not needed with a typical well-written Docker application and can safely be dropped, which makes privilege escalation much harder for an attacker.

   .. code-block:: yaml

      spec:
        containers:
          - name: <name>
            securityContext:
              capabilities:
                drop:
                  - all

   Be aware that this will drop ``CAP_NET_RAW``, which will mean ``ping`` will not work inside containers.
   If a service needs some specific capabilities, those can be added back using ``add``.

#. Mount the root file system read-only inside the pod.

   .. code-block:: yaml

      spec:
        containers:
          - name: <name>
            securityContext:
              readOnlyRootFilesystem: true

#. Create a ``NetworkPolicy``.
   Ingress rules are useful for nearly every service unless that service should be available to every pod running in the cluster (which is rare).
   Egress rules are normally not worth the trouble, but are useful for pods that should only accept connections from a single other pod (databases, Redis servers, etc.).
   In that case, you can disable egress for some additional security, although be aware that this will break DNS lookups and all outbound connections from that pod.
   Here is an example (taken from a Helm chart) for a Redis server limited to one specific application:

   .. code-block:: yaml

      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      metadata:
        name: {{ template "helpers.fullname" . }}-redis-networkpolicy
      spec:
        podSelector:
          matchLabels:
            app: {{ template "helpers.fullname" . }}-redis
        policyTypes:
          - Ingress
          - Egress
        ingress:
          - from:
              - podSelector:
                  matchLabels:
                    name: {{ template "helpers.fullname" . }}
            ports:
              - protocol: TCP
                port: 6379

References
==========

- `Google cluster hardening recommendations <https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster>`__
- `Kubernetes security hardening <https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/>`__
- `CNCF Kubernetes security recommendations <https://www.cncf.io/blog/2019/01/14/9-kubernetes-security-best-practices-everyone-must-follow/>`__
- `CIS Benchmark for GKE <https://learn.cisecurity.org/benchmarks>`__
