=======================================
Filter - Aggregate Instance Type Filter
=======================================

Problem description
===================

At present the filter scheduler allows operators to associate an instance
type with a host aggregate via the ``AggregateInstanceExtraSpecsFilter`` [1].
This filter enforces that an aggregate of hosts satisfies all conditions,
defined as extra specifications in the flavor; these conditions are set as
metadata in the aggregate class.

However, now the operator must include in the extra specs all the values to
be match in the aggregate metadata and can’t specify a unique value to
represent all data [use case 1]. Also, each variable included in the extra
specs must be present in the aggregate metadata, not giving the operator the
choice of making this variable not mandatory [use case 2] or force the
absence of this variable [use case 3].

Another limitation of the current implementation of
``AggregateInstanceExtraSpecsFilter`` filter is the logic imposed. The current
implemented logic implies an injective logic from flavor extra specifications
to aggregate metadata. That means, all elements present in the flavor must be
present in the aggregate to pass the filter. If the flavor has, for example,
three extra specs, the aggregate must have those three specs and the values
must satisfy the logic conditions present in the flavor extra specs. This new
filter has, as an aggregate option, a surjective logic. That means that,
instead of forcing the aggregate to satisfy the extra specs conditions
present in the flavor, now the flavor is enforced to satisfy the conditions
given by the aggregate defined as metadata. In this case, all metadata
elements must be present in flavor extra specs and must satisfy the
logic conditions present [use case 4].

To add also more flexibility to this new feature, the new sentinels defined in
this filter could be used in the aggregate metadata [use case 5].

Other limitation seen in the old filter [1] is the inability of using the
operators defined in AggregateInstanceExtraSpecsFilter [use case 6].

Finally, the latest limitation detected is how the namespaced variables are
filtered. Only those variables using a defined scope are actually used to
filter the host, nevertheless the rest of them are skipped and not being used
as part of the filter [use case 7].


Description
===========

A new host aggregates metadata entry flavor_extra_spec is added.

The sentinel value asterisk ``*`` may be used to specify that any value
is valid if the key is present. e.g. "key1" = "*"

The sentinel value tilde ``~`` may be used to specify that a key may
optionally be omitted. e.g. "key1" = "<or> 1 <or> ~"

The sentinel value exclamation mark ``!`` may be used to specify that the key
must not be present. e.g. "key1" = "!"

Tilde (optional value) sentinel and asterisk (present with any value) sentinel
are not mutually exclusive. Exclamation mark sentinel is exclusive with tilde
and asterisk. e.g. "key" = "<or> * <or> ~" is logically correct, meaning the
key may or may not exist with any value. In the case of having the key
"force_metadata_check" equal to "True", this is equivalent to not specifying
the value in the aggregate metadata. E.g:

flavor 1 extra specs: {"key": "~"}
flavor 2 extra specs: {"key": "<or> * <or> ~"}
flavor 3 extra specs: {"key": "*"}
flavor 4 extra specs: {}
aggregate 1 metadata: {"key": "1", "force_metadata_check": "False"}
aggregate 2 metadata: {}
aggregate 3 metadata: {"other key": "1", "force_metadata_check": "False"}

In this example:

* Flavor 1 can be booted in aggregate 2 and 3.

* Flavor 2 can be booted on all aggregates.

* Flavor 3 can be booted on aggregate 1 only.

* Flavor 4 can be booted on all aggregates.

The extra spec key ``force_metadata_check`` will be a reserved word. Its use
is explained in [use case 4], [use case 5] and [use case 6]. It's a boolean
value; if the key is present and the value is "True", the logic explained in
those use cases will apply. However, if the key is not present or the value
is "False", the filter won't apply this new logic.


Use Cases
=========

Use case 1
----------
An operator wants to filter hosts having a key in their aggregate
metadata, independently of its value, e.g.:

flavor extra specs: {"key": "*"}

All hosts inside a host aggregate containing this key, despite of the value,
will pass this check.

Use case 2
----------
An operator wants to filter hosts having a specific value, but if the
aggregate doesn’t have this key, the host should pass anyway, e.g..

flavor extra specs: {"key": "<or> 1 <or> ~"}
aggregate 1 metadata: {"key": "1"}
aggregate 2 metadata: {"key": "2"}
aggregate 3 metadata: {}

Hosts in aggregate 1 and 3 will pass this filter.

Use case 3
----------
In this case, the operator wants to stop any host inside an aggregate
containing a defined key, e.g.:

flavor extra specs: {"key": "!"}
aggregate 1 metadata: {"key": "1"}
aggregate 2 metadata: {}

Only hosts in aggregate 2 will pass.

Use case 4
----------
This use case could be used to force a flavor to contain a set of keys present
in the aggregate metadata. This constraint is added to the normal filter
process, which will try to match the keys present in the flavor with the keys
in the aggregate metadata. To activate in the filter this new verification
logic, a new metadata key is introduced: {"force_metadata_check": "True"}.
E.g.:

flavor 1 extra specs: {"key": "1"}
flavor 2 extra specs: {"key": "2"}
flavor 3 extra specs: {}
aggregate metadata: {"key": "1", "force_metadata_check": "True"}

In this example, hosts in this aggregate will pass only using the flavor 1.
Without the key ``force_metadata_check`` set to "True", the flavor 3 will
allow the use of hosts in the aggregate.

Use case 5
----------
If the key ``force_metadata_check``, explained in the last use case, is set,
the administrator will be able to use also the sentinels in the aggregate
metadata keys, e.g.:

flavor 1 extra specs: {"key": "1"}
flavor 2 extra specs: {"key": "2"}
flavor 3 extra specs: {}
aggregate metadata: {"key": "*", "force_metadata_check": "True"}

In this example, flavor 1 and 2 will pass hosts belonging to this aggregate.

Using this example, if the key ``force_metadata_check`` is removed (or set to
"False"), the unique accepted flavor will be 3. E.g.:

flavor 1 extra specs: {"key": "1"}
flavor 2 extra specs: {"key": "2"}
flavor 3 extra specs: {}
flavor 4 extra specs: {"key": "*", "key2": "2"}
aggregate metadata: {"key": "*"}

* Flavor 1 key value, "1", doesn't match the string "*".

* The same behaviour applies to flavor 2.

* Because flavor 3 doesn't have any requirement, it's accepted in this host
  aggregate; any flavor with extra specs will be accepted.

* Flavor 4 won't pass because "key2" doesn't exists in the aggregate metadata.

flavor 1 extra specs: {"key": "1"}
flavor 2 extra specs: {"key": "2"}
flavor 3 extra specs: {}
aggregate metadata: {"key": "!", "force_metadata_check": "True"}

In this third example, only flavor 3 will allow hosts in the aggregate.

This additional logic is backwards compatible with the existing one.

Use case 6
----------
Again, if the key ``force_metadata_check`` is set in the aggregate metadata,
the operator will be able to use the operator ``<or>`` to define multiple
values for a key. This change doesn’t break the logic of the old filter:
aggregate metadata checked inside the filter is a set of values combining the
data contained in the key of each aggregate metadata; this set now will
contain also the values inside the "or" junction. E.g.:

flavor 1 extra specs: {"key": "1"}
flavor 2 extra specs: {"key": "2"}
flavor 3 extra specs: {"key": "<or> 2 <or> 3"}
flavor 4 extra specs: {}
aggregate metadata: {"key": "<or> 1 <or> 2", "force_metadata_check": "True"}

In this example, only flavor 4 won’t pass the hosts inside the aggregate.

It should be noted that if the key ``force_metadata_check`` is not set, the
strings contained in the aggregate metadata keys will be checked literally.
Using the last example, if the key ``force_metadata_check`` is removed (or set
to "False"), the filter will use the aggregate metadata key value strings
without the new logic added by this filter, to maintain backwards
compatibility. E.g.:

flavor 1 extra specs: {"key": "1"}
flavor 2 extra specs: {"key": "2"}
flavor 3 extra specs: {"key": "<or> 2 <or> 3"}
flavor 4 extra specs: {}
flavor 5 extra specs: {"key": "<or> 1 <or> 2"}
aggregate metadata: {"key": "<or> 1 <or> 2"}

In this second example, no flavor will pass the filter. The 5th flavor have the
same string value in "key", but the current filter,
``AggregateInstanceExtraSpecsFilter``, compares each value in flavour's key,
"1" and "2", independently.

Use case 7
----------
The use of namespaced variables could be extended, allowing the operator to
filter hosts by these values. To maintain the backward compatibility, any
namespaced key without the escape scope used in the old filter,
``aggregate_instance_extra_specs``, will be considered optional: if the key is
not present in the aggregate metadata, the filter will skip this key; if the
key is also present in the aggregate metadata, the value will be checked as a
regular key. E.g.:

flavor extra specs: {"hw:cpu_policy": "shared"}
aggregate 1 metadata: {"hw:cpu_policy": "shared"}
aggregate 2 metadata: {"hw:cpu_policy": "dedicated"}
aggregate 3 metadata: {}

In this example, hosts in aggregates 1 and 3 will pass. But hosts in aggregate
2 won’t because namespace key is present in both extra specs and metadata and
the value is different. This new feature could collide with the old behaviour.

In the aggregate metadata, if the key ``force_metadata_check`` is set, all
keys, with or without namespace will be checked. This new feature check allows
the operator to define host aggregates with restrictions to spawn virtual
machines within their hosts. With this extension, the operator can easily,
only modifying the aggregate metadata (instead of the flavor definition),
define sets of hosts with specific properties and use restrictions.
