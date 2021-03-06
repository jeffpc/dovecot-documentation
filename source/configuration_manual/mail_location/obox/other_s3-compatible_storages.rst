.. _other_s3_compatible_storages:

==================================
Other S3-compatible Storages
==================================

This document covers the S3 compatible storage services. In case you want to
use AWS S3 please refer to :ref:`amazon_s3`.

The configuration for ``s3`` is likely to be similar to Amazon S3 ``aws-s3``,
but generally the bucket name must be specified explicitly via the bucket
parameter:

.. code-block:: none

  plugin {
    obox_fs = fscache:1G:/var/cache/mails:compress:gz:6:s3:https://ACCESSKEY:SECRET@host/?bucket=BUCKETNAME
    obox_index_fs = compress:gz:6:s3:https://ACCESSKEY:SECRET@host/?bucket=BUCKETNAME
  }

It's important that the S3 storage has an efficient way to list objects with a
given prefix. Many S3 storages either don't implement the listing at all, or
it's only used for disaster recovery type of purposes to list all objects. If
this is the case, you can still use the storage together with Cassandra.

Google Cloud Storage
^^^^^^^^^^^^^^^^^^^^^

GCS is similar to AWS in that a "dispersion prefix" is required to properly
shard among the Google Cloud storage nodes. Google has provided verification
that 6 characters of dispersion prefix is "enough distribution to ensure access
to pretty massive resources on our side without gymnastics on our end."

The example ``mail_location`` setting used for Amazon S3 should be used for
GCS (see :ref:`amazon_s3`).
