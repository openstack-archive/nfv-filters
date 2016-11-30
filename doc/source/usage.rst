=====
Usage
=====
Follow the `Installation <installation.html>`_ document first. This document
describes how to install this project.

To use nfv-filters in a project::

    import nfv_filters


local.conf settings
-------------------
See `Configuring filters <http://docs.openstack.org/developer/nova/filter_scheduler.html#configuring-filters>`_
to learn how to use configure Nova Scheduler filters.

To make available a new filter in Nova, not located in Nova Scheduler default
filters, you should add the file containing the filter class to the variable
`scheduler_available_filters` in the default section::

    [[post-config|$NOVA_CONF]]
    [DEFAULT]
    filter_scheduler.available_filters=nova.scheduler.filters.all_filters,nfv_filters.nova.scheduler.filters.aggregate_instance_type_filter

To enable this filter in Nova Scheduler, add your filter to the variable
`scheduler_default_filters`::

    [[post-config|$NOVA_CONF]]
    [DEFAULT]
    scheduler_default_filters=RamFilter,ComputeFilter,AggregateInstanceTypeFilter

