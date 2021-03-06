.. _amazon_s3:

================
Amazon S3
================

This document covers the Amazon Web Services S3 (Simple Storage Service). In
case you want to use another provider's S3 storage services, please refer to
:ref:`other_s3_compatible_storages`.

Partitioning/Layering
^^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

  mail_location = obox:%8{md5;format=hex:user}/%u:INDEX=~/:CONTROL=~/

We'll use the first 8 characters of the hex representation of the MD5 hash of
the username at the beginning of each object path. This is S3's ``dispersion
prefix`` which identifies which internal shard the data is stored in.

In AWS, by default, the sharding prefix is ignored for a bucket and it can be
enabled per request to AWS support.

.. Note:: The AWS sharding prefix is limited to hex characters \[0-9a-f] only.

When a S3 bucket is created, AWS creates a single shared partition for the
bucket with a default limit of 3,500 TPS for ``PUTs/DELETEs/POSTs`` § and 5500
GET requests per second (source).

This 3,500 TPS limit is generally too small and quickly surpassed by Dovecot
which results in a spike of ``503: Slow Down`` log events. It is strongly
recommended to contact AWS to request they manually set up 1 layer of hex
partitioning (``0-9a-f``),  to create16 dedicated partitions for your bucket.

``1 hex`` layer of partitioning thus means a theoretical capacity of 56,000
PUT/POST/DELETE and 88,000 GETs per second.

Per ``AWS``, you can go pretty deep in the number of layers, but most customers
do not need more than 2 layers of partitioning, (2 layers = 16x16 = 256
partitions = this would theoretically provide you up to: ``~896,000
PUT/POST/DELETE TPS and 1,408,000`` GET TPS if requests are distributed evenly
across the partitions).

Configuration
^^^^^^^^^^^^^

.. code-block:: none

  plugin {
    obox_fs = fscache:1G:/var/cache/mails:compress:gz:6:aws-s3:https://ACCESSKEY:SECRET@BUCKETNAME.s3.amazonaws.com/
    obox_index_fs = compress:gz:6:aws-s3:https://ACCESSKEY:SECRET@BUCKETNAME.s3.amazonaws.com/
  }

.. Note::
        If the version you are using does not support the ``aws-s3`` scheme,
        try using the more generic ``s3`` scheme.

        .. versionadded:: 2.3.10

Get ACCESSKEY and SECRET from `www.aws.amazon.com <https://aws.amazon.com/>`_
``-> My account -> Security credentials -> Access`` credentials. Create the
``BUCKETNAME`` from ``AWS Management Console -> S3 -> Create Bucket``.

If the ``ACCESSKEY`` or ``SECRET`` contains any special characters, they can be
%hex-encoded.

If the ``aws-s3`` scheme is used Dovecot defaults to prepend the following URL
parameters, refer to :ref:`http_based_object_storages` for details. The same
setup can be achieved by using the ``s3`` scheme and adding the parameters
manually.

.. code-block:: none

  addhdrvar=x-amz-security-token:%{auth:token}&loghdr=x-amz-request-id&loghdr=x-amz-id-2

.. Note::

  dovecot.conf handles %variable expansion internally as well, so % needs to be
  escaped as %% and ':' needs to be escaped as %%3A.


AWS Signing v2 vs v4
""""""""""""""""""""

S3 driver uses the ``AWS2`` signing method by default, but ``AWS4`` can be used
by adding the bucket region parameter to the S3 URL:

.. code-block:: none

  plugin {
    obox_index_fs = compress:gz:6:aws-s3:https://ACCESSKEY:SECRET@host/?bucket=BUCKETNAME&region=eu-central-1
  }

Configuration using AWS IAM
"""""""""""""""""""""""""""

Dovecot supports AWS Identity and Access Management (IAM) for authenticating
requests to AWS S3 using the AWS EC2 Instance Metadata Service (IMDS), solely
version 2 of IMDS (IMDSv2) is supported.

Using IAM allows running Dovecot with S3 Storage while not keeping the
credentials in the configuration.

A requirement for using IMDSv2 is that Dovecot is running on an AWS EC2
instance, otherwise the IMDS will not be reachable. Additionally an IAM role
must be configured which allows trusted entities, EC2 in this case, to
assume that role. The role (for example ``s3access``) that will be assumed must
have the ``AmazonS3FullAccess`` policy attached.

The ``auth_role`` can be configured as a URL parameter which specifies the IAM
role to be assumed. If no ``auth_role`` is configured, no IAM lookup will be
done. The IAM requests are targeted by default to the IMDSv2 service in EC2
(``169.254.169.25:80``). It is possible to override this default by appending
``auth_host=NEWHOST:NEWPORT`` to the URL, this is mainly used for testing
purposes.

.. code-block:: none

  plugin {
    obox_fs = aws-s3:https://BUCKETNAME.s3.amazonaws.com/?auth_role=s3access&region=eu-central-1
  }

.. versionadded:: 2.3.10

For more information about IAM roles for EC2 please refer to:
`IAM roles for Amazon EC2 <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html>`_

For general information about IAM:
`IAM UserGuide <https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html>`_

Deleting multiple objects per Request
"""""""""""""""""""""""""""""""""""""

The ``aws-s3`` and ``s3`` drivers support bulk-deletion. The ``bulk-delete``
option is enabled by default to delete up to 1000 keys with one request.
To change this behaviour refer to ``bulk_delete_limit`` at
:ref:`http_based_object_storages`. Bulk delete can only efficiently run on
multiple objects if configured to do so, via setting
``obox_max_parallel_deletes`` greater one (refer to :ref:`obox_settings`).

  .. versionadded:: 2.3.10

.. Warning:: AWS instances are known to react badly when high packets per second network traffic is generated by e.g. DNS lookups. Please see :ref:`os_configuration_dns_lookups`.
