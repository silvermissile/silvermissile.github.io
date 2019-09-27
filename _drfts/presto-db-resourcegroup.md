============================
Resource Group Configuration
============================

.. note::
    Resource groups are currently experimental and must be enabled with the
    ``experimental.resource-groups-enabled=true`` config flag.

Resource groups place limits on resource usage, and can enforce queueing policies on
queries that run within them or divide their resources among sub groups. A query
belongs to a single resource group, and consumes resources from that group (and its ancestors).
Except for the limit on queued queries, when a resource group runs out of a resource
it does not cause running queries to fail; instead new queries become queued.
A resource group may have sub groups or may accept queries, but may not do both.

The resource groups and associated selection rules are configured by a manager which is pluggable.
An implementation that uses a static file can be installed via the ``presto-resource-group-managers``
plugin and enabled by adding ``resource-groups.configuration-manager=file`` to
``etc/resource-groups.properties`` and setting ``resource-groups.config-file`` to the
location of a JSON config file with the properties described below.

To use an implementation that is backed by a database ``resource-groups.configuration-manager=db``
can be added to ``etc/resource-groups.properties`` and set the ``resource-groups.config-db-url`` to
the jdbc url of the config database.

For example:
``resource-groups.config-db-url=jdbc:mysql://myhost.mydomain.com:3306/mydatabase?user=admin_user&password=admin_password``.

The tables ``resource_groups``, ``selectors`` and ``resource_groups_global_properties`` will be created if they do not exist.
Once created a resource group and selector must be added in order to connect:

    ``insert into resource_groups (resource_group_id, name, soft_memory_limit, max_queued, max_running)``
    ``values (1, 'global', '95%', 100, 100);``
    ``insert into selectors (resource_group_id, user_regex, source_regex)``
    ``values (1, 'user_name1|user_name2', 'presto-cli|verifier.*');``

To use the ``cpuQuotaPeriod`` (see below) the following sql can be run to set it to 1 hour:

    ``insert into resource_groups_global_properties (name, value) values ('cpu_quota_period', '1h');``

Note: This requires the ``soft_cpu_limit`` and ``hard_cpu_limit`` to be set for all resource groups (details below).
For example:

    ``insert into resource_groups (resource_group_id, name, soft_memory_limit, max_queued, max_running,``
    ``soft_cpu_limit, hard_cpu_limit) values (1, 'global', '95%', 100, 100, '5m', '10m');``

Resource Group Properties
-------------------------

* ``name`` (required): name of the group. May be a template (see below).

* ``maxQueued`` (required): maximum number of queued queries. Once this limit is reached
  new queries will be rejected.

* ``maxRunning`` (required): maximum number of running queries.

* ``softMemoryLimit`` (required): maximum amount of distributed memory this
  group may use before new queries become queued. May be specified as
  an absolute value (i.e. ``1GB``) or as a percentage (i.e. ``10%``) of the cluster's memory.

* ``softCpuLimit`` (optional): maximum amount of CPU time this
  group may use in a period (see ``cpuQuotaPeriod``) before a penalty will be applied to
  the maximum number of running queries. ``hardCpuLimit`` must also be specified.

* ``hardCpuLimit`` (optional): maximum amount of CPU time this
  group may use in a period.

* ``schedulingPolicy`` (optional): specifies how queued queries are selected to run,
  and how sub groups become eligible to start their queries. May be one of three values:

    * ``fair`` (default): queued queries are processed first-in-first-out, and sub groups
      must take turns starting new queries (if they have any queued).

    * ``weighted``: queued queries are selected stochastically in proportion to their priority
      (specified via the ``query_priority`` :doc:`session property </sql/set-session>`). Sub groups are selected
      to start new queries in proportion to their ``schedulingWeight``.

    * ``query_priority``: all sub groups must also be configured with ``query_priority``.
      Queued queries will be selected strictly according to their priority.

* ``schedulingWeight`` (optional): weight of this sub group. See above.
  Defaults to ``1``.

* ``jmxExport`` (optional): If true, group statistics are exported to JMX for monitoring.
  Defaults to ``false``.

* ``subGroups`` (optional): list of sub groups.

* ``runningTimeLimit`` (optional): Time limit for queries to run within a resource group

* ``queuedTimeLimit`` (optional): Time limit for a query to be queued within a resource group

Selector Properties
-------------------

* ``user`` (optional): regex to match against user name. Defaults to ``.*``

* ``source`` (optional): regex to match against source string. Defaults to ``.*``

* ``group`` (required): the group these queries will run in.

Global Properties
-----------------

* ``cpuQuotaPeriod`` (optional): the period in which cpu quotas are enforced.

Selectors are processed sequentially and the first one that matches will be used.
In the example configuration below, there are five resource group templates.
In the ``adhoc_${USER}`` group, ``${USER}`` will be expanded to the name of the
user that submitted the query. ``${SOURCE}`` is also supported, which expands
to the source submitting the query. The source name can be set as follows:

  * CLI: use the ``--source`` option.

  * JDBC: set the ``ApplicationName`` client info property on the ``Connection`` instance.

There are three selectors that define which queries run in which resource group:

  * The first selector places queries from ``bob`` into the admin group.

  * The second selector states that all queries that come from a source that includes "pipeline"
    should run in the user's personal pipeline group, which belongs to the ``global.pipeline``
    parent group.

  * The last selector is a catch all, which puts all queries into the user's adhoc group.

All together these selectors implement the policy that ``bob`` is an admin and
all other users are subject to the follow limits:

  * Users are allowed to have up to 2 adhoc queries running. Additionally, they may run one pipeline.

  * No more than 5 "pipeline" queries may run at once.

  * No more than 100 total queries may run at once, unless they're from the admin.

``` json

    {
      "rootGroups": [
        {
          "name": "global",
          "softMemoryLimit": "80%",
          "maxRunning": 100,
          "maxQueued": 1000,
          "schedulingPolicy": "weighted",
          "jmxExport": true,
          "subGroups": [
            {
              "name": "adhoc_${USER}",
              "softMemoryLimit": "10%",
              "maxRunning": 2,
              "maxQueued": 1,
              "schedulingWeight": 9,
              "schedulingPolicy": "query_priority"
            },
            {
              "name": "pipeline",
              "softMemoryLimit": "20%",
              "maxRunning": 5,
              "maxQueued": 100,
              "schedulingWeight": 1,
              "jmxExport": true,
              "subGroups": [
                {
                  "name": "pipeline_${USER}",
                  "softMemoryLimit": "10%",
                  "maxRunning": 1,
                  "maxQueued": 100,
                  "schedulingPolicy": "query_priority"
                }
              ]
            }
          ]
        },
        {
          "name": "admin",
          "softMemoryLimit": "100%",
          "maxRunning": 100,
          "maxQueued": 100,
          "schedulingPolicy": "query_priority",
          "jmxExport": true
        }
      ],
      "selectors": [
        {
          "user": "bob",
          "group": "admin"
        },
        {
          "source": ".*pipeline.*",
          "group": "global.pipeline.pipeline_${USER}"
        },
        {
          "group": "global.adhoc_${USER}"
        }
      ],
      "cpuQuotaPeriod": "1h"
    }
```
