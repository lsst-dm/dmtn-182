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

A local Butler instance does not impose any access control and instead relies on the underlying file system permissions.
However, the Butler service as described in DMTN-169_ will receive authenticated requests from users and must decide whether to allow an operation while meeting the DMS-REQ-0340 requirement.

The Butler organizes data into collections, which support read and write operations.
Collections are organized hierarchically has paths.
The current plan is to have the Butler service support two main path structures: one with a top-level directory per user, and one for collaborations where the top-level directory corresponds to a group.
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

with arbitrary structure below those top-level paths.
The exact naming may be different; the important semantics are that there are two top-level path patterns, one for user-owned hierarchies and one for group-owned hierarchies.

Each collection may optionally be associated with an :abbr:`ACL (Access Control List)`.
This is a simple list of groups that have access to that collection.

When the Butler receives a user request, it first uses the associated token to obtain metadata about the user by making a call to the ``/auth/api/v1/user-info`` endpoint (see SQR-049_).
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

For any Butler access request:

- If the path starts with ``/u/{username}``, access will be granted and the collection created if the ``username`` attribute of the supplied token matches ``{username}``.
- If the path starts with ``/g/{group}``, access will be granted if ``{group}`` appears as the ``name`` field of a group in the ``groups`` attribute of the supplied token.

If neither of these rules grants access, the Butler will then check whether this collection is associated with an ACL.
If so, it will perform a third authorization check:

- Take the set intersection of the list of group names from the ``groups`` attribute of the supplied token and the groups in the ACL.
  If the intersection is non-empty, access is allowed; otherwise, access is denied.

If there is no ACL, access is denied.

The Butler will also support an additional API call to set or clear the ACL for a collection.

The identity management system will guarantee that every user is also the sole member of a group whose name matches the username.
(This is desirable anyway for POSIX file system semantics for the Notebook Aspect of the Rubin Science Platform.)
Therefore, to grant a specific user access to a collection, the username of the user can be added to the ACL alongside any other group.
This satisfies the DMS-REQ-0340 requirement.

Variations
----------

Due to the expected use of nested groups of collections, the Butler may want to allow an ACL to be associated with a path prefix or wildcard and not only a single collection.
This changes the logic for finding an ACL that applies to a given collection, but the rest of the authorization logic is unchanged.

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
