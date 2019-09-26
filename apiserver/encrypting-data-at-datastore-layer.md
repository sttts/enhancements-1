---
title: Encrypting Data at Datastore Layer
authors:
  - "@enj"
reviewers:
  - "@sttts"
  - "@deads2k"
  - "@p0lyn0mial"
approvers:
  - "@sttts"
  - "@derekwaynecarr"
  - "@smarterclayton"
creation-date: 2019-09-09
last-updated: 2019-09-20
status: implementable
see-also:
  - "/enhancements/etcd/storage-migration-for-etcd-encryption.md"
---

# Encrypting Data at Datastore Layer

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

Provide automatic and seamless support for encryption of data stored in etcd while maintaining the principles of OS4 (a self-managed platform that abstracts cluster configuration through intuitive Kubernetes style APIs).

## Motivation

Even in the presence of full disk encryption, users want the ability to encrypt data stored in etcd.  This provides an extra layer of protection against data leakage.  For example, it protects the loss of sensitive secret information if an etcd backup is exposed to the incorrect parties.

### Goals

1. User can enable encryption (globally, with limited config)
1. The keys used to encrypt are machine generated
1. The keys used to encrypt are periodically rotated automatically
1. User can check that encryption is active (coarse view)
1. User can recover a cluster from etcd backup as long they have the encryption keys
1. Allow encryption to be disabled once it has been enabled
1. Encryption at rest should not meaningfully degrade the performance of the cluster

### Non-Goals

1. Allow the user to force key rotation (see test plan section for workarounds for CI)
1. Support for the Kubernetes KMS envelope encryption
1. Support for hardware security modules
1. Allow the user to configure the keys that are used
1. The user has in-depth understanding of each phase of the encryption process

But, Non-Goals 1, 2 and 3 must be possible as future goals; the design should offer a way to add them at a later stage.

## Proposal

Add a new Encryption field to the `APIServer` config:

```diff
diff --git a/config/v1/types_apiserver.go b/config/v1/types_apiserver.go
index ea76aec0..355b25da 100644
--- a/config/v1/types_apiserver.go
+++ b/config/v1/types_apiserver.go
@@ -39,6 +39,9 @@ type APIServerSpec struct {
     // The values are regular expressions that correspond to the Golang regular expression language.
     // +optional
     AdditionalCORSAllowedOrigins []string `json:"additionalCORSAllowedOrigins,omitempty"`
+    // encryption allows the configuration of encryption of resources at the datastore layer.
+    // +optional
+    Encryption APIServerEncryption `json:"encryption"`
 }

 type APIServerServingCerts struct {
@@ -63,6 +66,38 @@ type APIServerNamedServingCert struct {
     ServingCertificate SecretNameReference `json:"servingCertificate"`
 }

+type APIServerEncryption struct {
+    // type defines what encryption type should be used to encrypt resources at the datastore layer.
+    // When this field is unset (i.e. when it is set to the empty string), identity is implied.
+    // The behavior of unset can and will change over time.  Even if encryption is enabled by default,
+    // the meaning of unset may change to a different encryption type based on changes in best practices.
+    //
+    // When encryption is enabled, all sensitive resources shipped with the platform are encrypted.
+    // This list of sensitive resources can and will change over time.  The current authoritative list is:
+    //
+    //   1. secrets
+    //   2. configmaps
+    //   3. routes.route.openshift.io
+    //   4. oauthaccesstokens.oauth.openshift.io
+    //   5. oauthauthorizetokens.oauth.openshift.io
+    //
+    // +unionDiscriminator
+    // +optional
+    Type EncryptionType `json:"type,omitempty"`
+}
+
+type EncryptionType string
+
+const (
+    // identity refers to a type where no encryption is performed at the datastore layer.
+    // Resources are written as-is without encryption.
+    EncryptionTypeIdentity EncryptionType = "identity"
+
+    // aescbc refers to a type where AES-CBC with PKCS#7 padding and a 32-byte key
+    // is used to perform encryption at the datastore layer.
+    EncryptionTypeAESCBC EncryptionType = "aescbc"
+)
+
 type APIServerStatus struct {
 }
```

The default value of empty string for the type will imply `identity` (meaning no encryption of the datastore layer).  This may change in a future release.

The kube-apiserver operator and openshift-apiserver operator will observe this config and begin encrypting sensitive resources when it is set.  These operators are responsible for determining what resources to encrypt.

### User Stories

#### Story 1

User enables encryption by setting `apiserver.spec.encryption.type` to `aescbc`.  After some time passes, user makes a backup of etcd.  The user confirms that the secret values are encrypted by checking to see if they have the `k8s:enc:aescbc:v1:` prefix.

### Implementation Details/Notes/Constraints

The encryption key rotation logic will be implemented using a distributed state machine with a series of controllers (more details below).

#### Fundamentals

- there are 
  - **non-encrypted** resources
  - **to-be-encrypted** resource: those which should be encrypted, but aren't yet
  - and **encrypted** resources: those which are already configured for encryption, but maybe not fully migrated.
- keys are actually pairs of `<encryption-function>` and `<encryption-key>`, where the encryption-function will be one of the supported encryption functions from upstream, e.g. `identity`, `aescbc`, and the encryption-key is a corresponding base64 key string.
- keys are eventually the same for all encrypted resources.
- keys are numbered with a strictly increasing integer.
- keys can be 
  - unused, 
  - read-key,
  - write-key, 
  - migrated-key
  for a given GR in an [`apiserver.config.k8s.io/v1.EncryptionConfig`](https://github.com/kubernetes/kubernetes/blob/49891cc270019245a3d4796e84b33bf36d0bae08/staging/src/k8s.io/apiserver/pkg/apis/config/v1/types.go#L24).

#### State

The distributed state machine is using the following data:

1. keys stored in `openshift-config-managed/<component>-encryption-<unsigned integer #>` secrets, and found via the label `encryption.operator.openshift.io/component` for the respective component. They are named as `key-<unsigned integer #>`.
2. the target encryption configuration stored in the `<operand-target-namespace>/encryption-config` secret as upstream [`apiserver.config.k8s.io/v1.EncryptionConfig`](https://github.com/kubernetes/kubernetes/blob/49891cc270019245a3d4796e84b33bf36d0bae08/staging/src/k8s.io/apiserver/pkg/apis/config/v1/types.go#L24) type.
3. `revision` label of running API server pods.
4. the observed encryption configuration stored in the `<operand-target-namespace>/encryption-config-<revision>` secret.
5. the encryption `APIServer` configuration defined above.

Pod revisions and key numbers are unrelated.

We say that a key `key-<n>` is

1. **created** - if its secret `openshift-config-managed/<component>-encryption-<n>` is created.
2. **configured read-key for resource GR** if it is defined as read-key for GR in the target encryption config secret. We call it **observed read-key for resource GR** if all API server instances are running with a corresponding config.
3. **configured write-key for resource GR** if it is defined as write-key for GR in the target encryption config secret. We call it **observed write-key for resource GR** if all API server instances are running with a corresponding config.
4. **migrated for resource GR** if the key secret `openshift-config-managed/<component>-encryption-<n>`'s annotation `encryption.operator.openshift.io/migrated-resources` lists the GR.
5. **deleted** - if its secret `openshift-config-managed/<component>-encryption-<n>` is deleted.

Each version of the operator has a fixed list of GRs that a supposed to be encrypted. All other GRs are **non-encrypted** for that operator. We say that such a resource of the former is **encrypted** if it is configured with at least a read-key in the target encryption config, and **to-be-encrypted** if it is not.

#### Invariants

The encryption configuration maintains the following invariants

- the read-keys for all **encrypted** GRs are the same.
- there is at most one non-identity write-key for all **encrypted** GRs (e.g. a new resource starts with `identity`, all others have a `aescbc` key already).

#### Transitions

The life-cycle of a key implemented by the controllers is: 1, 2, 3, 4, 5. 

A transition only takes place when all API servers have converged to the same revision, are running, and hence use the same encryption configuration.

- ->1: a new key secret is created if 
  - encryption is enabled via the API or
  - a new **to-be-encrypted** resource shows up or
  - the `EncryptionType` in the API does not match with the newest existing key or
  - based on time (once a **week** is the proposed rotation interval).
- 1->2: when a new **created** key is found.
- 2->3: when all API servers have **observed the read-key for resource GR**.
- 3->4: when all GR instances have been rewritten in etcd and the key is marked as **migrated** for that GR by the migration mechanism.
- 4->5: when another write-key has reached (4).

#### Controllers

The transitions are implemented by a series of controllers which each perform a single, _simple_ task.

##### Shared code: computing a new desired configuration

A new, desired encryption config is computed from the **created** keys and the old revision-config `<operand-target-namespace>/encryption-config-<revision>` by

- **gets** all API server pods by the matching `encryption.operator.openshift.io/component` label to
  - to check whether they converged 
  - and to know to which revision.
- **gets** the `<operand-target-namespace>/encryption-config-<revision>` secret.
- **gets** the keys in `openshift-config-managed` by matching `encryption.operator.openshift.io/component` label.

and then following:

- if to-be-encrypted GRs are missing in the old config, return desired config with (a) updated read-keys for all existing resources (b) added to-be-encrypted resourcdes with all the read-keys and `identity` for the write key.
- if configured read-keys do not match the **created** keys in the cluster, return desired config with updated read-keys.
- if a write-key does not match the latest **observed read-key**, return desired config with this latest **observed read-key** as write-key.
- if the write-key is marked as **migrated** for all **encrypted** resources, return desired config with all other read-keys removed.

##### encryptionKeyController

The `encryptionKeyController` implements the `->1` transition. It

- **watches** pods and secrets in `<operand-target-namespace>` (and is triggered by changes)
- **computes** a new, desired encryption config from `encryption-config-<revision>` and the existing keys in `openshift-config-managed`. 
- **derives** from the desired encryption config whether a new key is needed. It then creates it.

##### encryptionStateController

The `encryptionStateController` controller implements transitions `1->2`, `2-3`, and `4-5` (does it????). It

- **watches** pods and secrets in `<operand-target-namespace>` (and is triggered by changes)
- **computes** a new, desired encryption config from `encryption-config-<revision>` and the existing keys in `openshift-config-managed`. 
- **applies** the new, desired encryption config to `<operand-target-namespace>/encryption-config` if it differs.

##### encryptionMigrationController

The `encryptionMigrationController` controller implements transition `3-4`. It

- **watches** pods and secrets in `<operand-target-namespace>` (and is triggered by changes)
- **computes** a new, desired encryption config from `encryption-config-<revision>` and the existing keys in `openshift-config-managed`.
- **compares** desired with current target config and stops when they differ
- **checks** the write-key secret whether
  - `encryption.operator.openshift.io/migrated-timestamp` annotation is missing or
  - a write-key for a resource does not show up in the `encryption.operator.openshift.io/migrated-resources`
  And then **starts** a migration job (currently in-place synchronously, soon with the upstream migration tool)
- **updates** the `encryption.operator.openshift.io/migrated-timestamp` and  `encryption.operator.openshift.io/migrated-resources` annotations on the write-key secrets used for migration in the previous step.

##### encryptionPruneController

`encryptionPruneController` prevents an unbounded growth of old encryption keys.
For a given resource, if there are more than ten keys which have been migrated,
this controller will delete the oldest migrated keys until there are ten migrated
keys total.  These keys are safe to delete since no data in etcd is encrypted using
them.  Keeping a small number of old keys around is meant to help facilitate
decryption of old backups (and general precaution).

#### Resources

This list of resources is not configurable.  The following resources are encrypted:

kube-apiserver:

1. `secrets`
1. `configmaps`

openshift-apiserver:

1. `routes.route.openshift.io` (routes can contain embedded TLS private keys)
1. `oauthaccesstokens.oauth.openshift.io`
1. `oauthauthorizetokens.oauth.openshift.io`

### Risks and Mitigations

1. Deletion of an in-use encryption key will permanently break a cluster
    - Keys are stored in `openshift-config-managed` which is an immortal namespace
    - Keys require a two-phase delete via a finalizer as mentioned above
1. Encrypting a large number of resources may cause a high number of rollouts
    - Controller #1 above does not update the single complete secret during rollouts of the API server.  This causes it to collapse the addition of many new keys into a smaller set of changes.
1. We wish to use the upstream alpha migration controller to perform storage migration
    - It may not be ready for use in product environments
    - In case it proves to be unreliable, we have simple chunking based migration embedded in the operator via a controller

## Design Details

### Test Plan

1. Unit tests for all controllers
1. Unit tests for encryption config and state transition logic
1. No integration tests
1. New e2e suite in the operators to exercise rotation (will be a very slow suite with long timeout of three hours).  The operator's `unsupportedConfigOverrides` field will support a new `encryption` stanza that will allow forcing of rotation for testing purposes.  Setting this field will prevent upgrades (un-setting it after the fact will _not_ allow upgrades).
1. Encryption will be enabled by default in CI for `e2e-aws` (this will only exercise the off to on state)
1. QE will be asked to always have encryption enabled in all test clusters, especially long-lived clusters
1. The OpenShift Online environments will always run with encryption enabled

### Graduation Criteria

##### This feature will GA in 4.3 as long as:

1. Thorough end to end tests, especially around key rotation
1. Thorough unit tests around various invariants required to keep rotation working
1. Docs around verification of encryption and disaster recovery

### Upgrade / Downgrade Strategy

1. Downgrade
    - We may backport the encryption config observer to 4.2.  This would allow a downgrade to 4.2 if encryption had been enabled in 4.3.  The keys would be static since none of the other controllers would be present.
    - Downgrading from say, 4.4 to 4.3 may be an issue if new encrypted resources are added in 4.4.  Some of the group resource validation done by the controllers could be relaxed to accommodate this.
1. Upgrade
    - The addition of new encrypted resources is explicitly handled by the controllers (this logic is used on the first run).

### Version Skew Strategy

1. The controllers constantly handle revision skews between masters as part of their regular operation (and this has the same properties as an upgrade)
    - New keys go through a read and then write phase to prevent a new master from encrypting data in a way that an old master cannot decrypt
    - Storage migrations are not performed when there is a revision skew between masters

## Implementation History

1. [Original MVP](https://docs.google.com/document/d/16GGIgacLtmCJIgQrxjfItAt15EAuR6IxtO-vqIBoSwE)

## Drawbacks

1. Adds significant complexity to core operators
1. Complicates disaster recovery and backups
1. Possible performance impacts

## Alternatives

1. Skipping this enchantment and instead going straight to KMS - difficult to do because it is hard to support KMS universally whereas the file based approach will always work

## Infrastructure Needed

1. Some configuration or extra tooling may be required to enable encryption by default in `e2e-aws`
