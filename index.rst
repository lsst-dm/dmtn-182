:tocdepth: 1

.. sectnum::

Abstract
========

Given the authentication implementation described in DMTN-169_, Butler then has to decide whether to permit an operation on a collection following requirement DMS-REQ-0340 in LSE-61_.
This document proposes having the Butler service maintain a list of groups and do a group membership check of the requesting user.
Other options and their trade-offs are also discussed.

.. _DMTN-169: https://dmtn-169.lsst.io/
.. _LSE-61: https://docushare.lsst.org/docushare/dsweb/Get/LSE-61

Requirements
============

DMS-REQ-0340 states:

    All Level 3 data products shall be configured to have the ability to have access restricted to the owner, a list of people, a named group, or be completely public.

Per LPM-231_, "Level 3" here refers to User Generated data products.
The Butler service will also serve Data Release data products.
Those will be public and read-only.

.. _LPM-231: https://docushare.lsst.org/docushare/dsweb/Get/LPM-231

A local Butler instance does not impose any access control and instead relies on the underlying file system permissions.
However, the Butler service as described in DMTN-169_ will receive authenticated requests from users and must decide whether to allow an operation while meeting the DMS-REQ-0340 requirement.

The Butler organizes data into collections, which support read and write operations.
All Butler operations are done on collections.
Controlling access to collections therefore is sufficient to control access to data served by the Butler service.

Collections are organized hierarchically as paths.
The current plan is to have the Butler service support two user-writable path structures in addition to common data products: one with a top-level directory per user, and one for collaborations where the top-level directory corresponds to a group.
However, within those paths, the owners of that portion of the Butler namespace may choose to grant access to some collections to other users or groups.

The identity management system for the Rubin Science Platform is responsible for authenticating users and maintaining their group memberships (see SQR-039_ and SQR-044_).
A user of the Butler service will obtain a token from the identity management system and present it as part of an API call.
The Butler service can, in turn, use that token to obtain the identity of the user and their group membership using the token API (see SQR-049_).
With that information, it must somehow decide whether an operation is permitted.

.. _SQR-039: https://sqr-039.lsst.io/
.. _SQR-044: https://sqr-044.lsst.io/
.. _SQR-049: https://sqr-049.lsst.io/

Proposed solution
=================

Assume that Butler collection paths are of the form:

- ``/u/{username}/...``
- ``/g/{group}/...``
- ``/other/...``

with arbitrary structure below those top-level paths.
The exact naming may be different.
The important semantics are that there are two top-level path patterns, one for user-owned hierarchies and one for group-owned hierarchies, and then a third type of path that doesn't match either of those patterns.

This proposal assumes that paths that do not begin with ``/u`` or ``/g`` will be written outside of the Butler service (via direct access to the Butler registry, for example) and will otherwise be public and read-only.

Each collection may optionally be associated in the Butler registry with an :abbr:`ACL (Access Control List)`.
An ACL contains a list of groups that have access to that collection.

When the Butler receives a user request, it first checks whether the request is to one of the public collections.

1. If the path does not begin with ``/u`` or ``/g``, read access is allowed and write access is denied.

If no access control decision is yet made, the Butler service then uses the associated token to obtain metadata about the user by making a call to the ``/auth/api/v1/user-info`` endpoint (see SQR-049_).
This will return information such as:

.. code-block:: json

   {
     "username": "alice",
     "name": "Alice Example",
     "uid": 124187,
     "groups": [
       {
         "id": 124187,
         "name": "alice",
       },
       {
         "id": 204173,
         "name": "example-group"
       },
       {
         "id": 205671,
         "name": "other-group"
       }
     ]
   }

The Butler service may (and generally should) cache this information for a given token.
The cache should have a maximum lifetime of 30 minutes per LSE-279_, which says that it must be possible to revoke access to a user within 30 minutes.
Consider using a shorter cache timeout if the result of the authorization check is to deny access, since the most likely failure is that a person was just added to a group.

.. _LSE-279: https://docushare.lsst.org/docushare/dsweb/Get/LSE-279

It then makes two more checks:

2. If the path starts with ``/u/{username}``, access will be granted and the collection created if the ``username`` attribute of the supplied token matches ``{username}``.
3. If the path starts with ``/g/{group}``, access will be granted if ``{group}`` appears as the ``name`` field of a group in the ``groups`` attribute of the supplied token.

If neither of these rules grants access, the Butler will then check the ACL:

4. Take the set intersection of the list of group names from the ``groups`` attribute of the supplied token and the groups in the ACL.
   If the intersection is non-empty, access is allowed; otherwise, access is denied.
   (GIDs should be used instead of group names, but the necessary API is not yet available in the identity management system.
   See ref:`future-work`.)

If there is no ACL, access is denied.

The Butler will also support an additional API call to set or modify the ACL for a collection.
This action is authorized using only authorization rules 2 and 3 (the rules based on the path structure).
In other words, only the user or group who owns the collection, because the collection is in their data area, can change the ACL.
Members of the ACL cannot change the ACL.
(Administrators of the Butler can of course bypass this and make ACL changes directly if necessary.)

The identity management system will guarantee that every user is also the sole member of a group whose name matches the username.
(This is desirable anyway for POSIX file system semantics for the Notebook Aspect of the Rubin Science Platform.)
Therefore, to grant a specific user access to a collection, the username of the user can be added to the ACL alongside any other group.
In addition, every user is a member of a single group representing all authorized users.
A collection can then be made public by adding this group to the ACL.
This satisfies the DMS-REQ-0340 requirement.

.. _future-work:

Future work
-----------

The system described above isn't robust against changes to group names.
Each ACL referring to the old group name would need to be updated with the new group name.
This could be avoided by storing GIDs instead of group names in the ACL, since GIDs are guaranteed to not change when the group is renamed.
However, the identity management system does not yet support retrieving the GID of a group given its name.
This is blocked by switching to a new group system.

Once this functionality is available, the Butler should use GIDs instead of group names.
Alternately, implementation of this proposal could be delayed until the new group management system is ready.

Variations
----------

This approach assumes that all collections outside of the ``/u`` and ``/g`` paths are public.
If there is a later need to restrict access to collections outside of the user or group namespaces based on group membership or other user metadata, that can be handled with expanding ACL checks to other paths.
In this case, caching of group data will become more important to prevent hammering the identity management system with API calls for heavily-used collections.

Due to the expected use of nested groups of collections, the Butler may want to allow an ACL to be associated with a path prefix or wildcard and not only a single collection.
This changes the logic for finding an ACL that applies to a given collection, but the rest of the authorization logic is unchanged.

If there is a need to separate read access from write access in ACLs, each collection can be associated with two ACLs, one controlling read and one controlling write.
The implicit ownership checks (the first two authorization rules) grant both read and write access.
The third authorization rule is applied to either the read ACL or the write ACL depending on the operation.
The API to set or clear the ACL would take an additional parameter specifying whether to act on the read ACL or the write ACL.

If there are high-volume public collections inside the user or group namespaces, the ACL could be expanded to support a public flag which, if set, grants read access to all authenticated users.
This would allow short-cutting authorization decisions for those collections before having to make a call to the identity management system.
However, this adds additional complexity, including two ways to specify the same thing (setting the public flag and adding the all-users group to a read ACL), so is probably best avoided unless the volume warrants it.
The most important collections to exclude from identity management calls are the large, high-volume public collections like data releases, and those are not expected to be in the user or group namespaces.

Alternatives
============

The following alternative implementations were discussed and rejected.

Centralized authorization
-------------------------

The identity management system would provide an authorization API for the use of the Butler.
Each time the Butler receives a request, it would present that request to the authorization API and ask if that request should be allowed.

Advantages:

- Minimizes the work in the Butler
- Centralizes security decisions in the identity management system

Drawbacks:

- Centralized authorization systems are disfavored in security design because authorization logic is deeply tied to the data and operations model of a service.
  This is known to lead to maintainability issues.
  Either the authorization system and all the services it protects have to constantly change in lockstep, requiring coordinating changes and deployments to the authorization service when adding new features to any other service, or the authorization service has to use a complex abstract grammar of subjects, verbs, and objects in which to express generic authorization rules.
  The latter adds complexity and confusion without saving effort; defining the verbs and objects tends to be more tedious than directly implementing the authorization logic.

Pure group semantics
--------------------

Do not expose a group whose name matches the name of the user.
Instead, only support ad hoc groups.
If a user wishes to grant access to a collection to a set of people who are not already represented by a group, have them create a group and populate it with the users to whom they want to grant access.

Advantages:

- Simplifies group semenatics.
  All groups are managed groups, and there are no synthesized, artificial groups whose membership cannot be changed.
- Avoids user frustration when they add multiple people to multiple collections for a given project and then later discover there is no easy way to add or remove a given person to all of the relevant collections.

Disadvantages:

- Does not satisfy DMS-REQ-0340
- Forces users to do more work up-front rather than merely giving them the option
