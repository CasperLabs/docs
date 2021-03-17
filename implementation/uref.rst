.. _uref-head:

Unforgable Reference (URef)
===========================

This key type is used for storing any type of value except ``Account``.
Additionally, ``URef``\ s used in contracts carry permission information with them
to prevent unauthorized usage of the value stored under the key. This permission
information is tracked by the runtime, meaning that if a malicious contract
attempts to produce a ``URef`` with permissions that contract does not actually
have, we say the contract has attempted to “forge” the unforgable reference, and
the runtime will raise a forged ``URef`` error. Permissions for a ``URef`` can be
given across contract calls, allowing data stored under a ``URef`` to be shared in
a controlled way. The 32-byte identifier representing the key is generated
randomly by the runtime (see :ref:`Execution Semantics <execution-semantics-urefs>` for
for more information). The serialization for ``Access Rights`` that define the permissions for ``URefs`` is detailed in the :ref:`CLValues<serialization-standard-values>` section.


.. _uref-permissions:

Permissions for ``URef``\ s
~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the runtime, a ``URef`` carries its own permissions called ``AccessRights``.
Additionally, the runtime tracks what ``AccessRights`` would be valid for each
``URef`` to have in each context. As mentioned above, if a malicious contract
attempts to use a ``URef`` with ``AccessRights`` that are not valid in its
context, then the runtime will raise an error; this is what enforces the
security properties of all keys. By default, in all contexts, all ``URef``\ s
are invalid (both with any ``AccessRights``, or no ``AccessRights``); however, a
``URef`` can be added to a context in the following ways:

-  it can exist in a set of “known” ``URef``\ s
-  it can be freshly created by the runtime via the ``new_uref`` function
-  for called contracts, it can be passed in by the caller via the arguments to
   ``call_contract``
-  it can be returned back to the caller from ``call_contract`` via the ``ret``
   function

Note: that only valid ``URef``\ s may be added to the known ``URef``\ s or cross call
boundaries; this means the system cannot be tricked into accepted a forged
``URef`` by getting it through a contract or stashing it in the known ``URef``\ s.

The ability to pass ``URef``\ s between contexts via ``call_contract`` / ``ret``, allows
them to be used to share state among a fixed number of parties, while keeping it
private from all others.