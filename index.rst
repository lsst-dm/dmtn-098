.. default-domain:: py

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. _overview:

Overview
========

The :mod:`lsst.verify` package provides a framework for characterizing the LSST Science Pipelines through specific metrics, which are configured by the ``verify_metrics`` package.
However, :mod:`lsst.verify` does not yet specify how to organize the code that measures and stores the metrics.
This document proposes an extension of :mod:`lsst.verify` that interacts with the Task framework to make it easy to write new metrics and apply them to the Science Pipelines.

The proposed design, :ref:`shown below <fig-classes>`, is similar to the `make measurements from output datasets <https://dmtn-057.lsst.io/#option-make-measurements-from-output-datasets>`_ option proposed in DMTN-057.
Each metric will have an associated :class:`lsst.pipe.base.Task` class that is responsible for measuring it based on data previously written to a Butler repository.
These tasks will be grouped together for execution, first as plugins to a central metrics-managing task, and later as components of a :class:`lsst.pipe.base.Pipeline`.
The central task or pipeline will handle the details of directing the potentially large number of input datasets to the measurement tasks that analyze them.

This design proposal strives to be consistent with the recommendations on metrics generation and usage provided by `DMTN-085`_, without assuming them more than necessary.
However, the design does require that ``QAWG-REC-34`` ("Metric values should have Butler dataIds") be adopted; otherwise, the output dataset type of the proposed measurement task would be ill-defined.

.. figure:: /_static/class_diagram.svg
   :target: _images/class_diagram.svg
   :name: fig-classes

   Classes introduced in this design (blue), including a compatibility module for adapting the framework to a non-``PipelineTask`` system. For simplicity, ``Task`` config classes are treated as part of the corresponding ``Task``.


.. _design-goals:

Design Goals
============

The goals of this design are based on those `presented in DMTN-057 <https://dmtn-057.lsst.io/#design-goals>`_.
In particular, the system must be easy to extend, must support a variety of metrics, and must be agnostic to how data processing is divided among task instances. It must not add extra configuration requirements to task users who are not interested in metrics, and it must be possible to disable metrics selectively during commissioning and operations.

`DMTN-085`_ makes several recommendations that, if adopted, would impose additional requirements on a measurement creation framework. Specifically:

* ``QAWG-REC-31`` recommends that the computation and aggregation of measurements be separated.
* ``QAWG-REC-32`` recommends that measurements be stored at the finest granularity at which they can be reasonably defined.
* ``QAWG-REC-34`` and ``QAWG-REC-35`` recommend that measurements be Butler datasets with data IDs.
* ``QAWG-REC-41`` recommends that metrics can be submitted to SQuaSH (and, presumably, measured first) from arbitrary execution environments

DMTN-085 Section 4.11.1 also informally proposes that the granularity of a metric be part of its definition, and notes that some metrics may need to be measured during data processing pipeline execution, rather than as a separate step.
I note in the appropriate sections how these capabilities can be accommodated.

The capabilities and requirements of ``PipelineTask`` are much clearer now than they were when `DMTN-057`_ was written (note: it refers to ``PipelineTask`` by its previous name, ``SuperTask``).
Since reusability outside the Tasks framework is not a concern, the design proposed here can be tightly coupled to the existing :class:`lsst.pipe.base.PipelineTask` API.

.. _components-primary:

Primary Components
==================

The framework creates :class:`lsst.pipe.base.PipelineTask` subclasses responsible for measuring metrics and constructing :class:`lsst.verify.Measurement` objects.
Metrics-measuring tasks (hereafter ``MetricTasks``) will be added to data processing pipelines, and the ``PipelineTask`` framework will be responsible for scheduling metrics computation and collecting the results.
It is expected that ``PipelineTask`` will provide some mechanism for grouping tasks together (e.g., sub-pipelines), which will make it easier to enable and configure large groups of metrics.
``PipelineTask`` is not available for general use at the time of writing, so initial implementations may need to avoid referring to its API directly (see the :ref:`Butler Gen 2 MetricTask <components-compatibility-metrictask>`).

Because ``MetricTasks`` are handled separately from data processing tasks, the latter can be run without needing to know about or configure metrics.
Metrics that *must* be calculated while the pipeline is running may be integrated into pipeline tasks as subtasks, with the measurement(s) being added to the list of pipeline task outputs, but doing so greatly reduces the flexibility of the framework and is not recommended.

While this proposal places ``MetricTask`` and its supporting classes in :mod:`lsst.verify` (see :ref:`Figure 1 <fig-classes>`), its subclasses can go in any package that can depend on both :mod:`lsst.verify` and :mod:`lsst.pipe.base`.
For example, subclasses of ``MetricTask`` may be defined in the packages of the task they instrument, in plugin packages similar to ``meas_extensions_*``, or in a dedicated pipeline verification package.
The framework is therefore compatible with any future policy decisions concerning metric implementations.

.. _components-primary-metrictask:

MetricTask
----------

The code to compute any metric shall be a subclass of ``MetricTask``, a :class:`~lsst.pipe.base.PipelineTask` specialized for metrics.
Each ``MetricTask`` shall read the necessary data from a repository, and produce a :class:`lsst.verify.Measurement` of the corresponding metric.
Measurements may be associated with particular quanta or data IDs, or they may be repository-wide.

Because metric measurers may read a variety of datasets, ``PipelineTask``'s ability to automatically manage dataset types is essential to keeping the framework easy to extend.

.. _components-primary-metrictask-abstract:

Abstract Members
^^^^^^^^^^^^^^^^

``run(undefined) : lsst.pipe.base.Struct``
    Subclasses may provide a ``run`` method, which should take multiple datasets of a given type.
    Its return value must contain a field, ``measurement``, mapping to the resulting :class:`lsst.verify.Measurement`.

    ``MetricTask`` shall do nothing (returning :py:const:`None` in place of a :class:`~lsst.verify.Measurement`) if the data it needs are not available.
    Behavior when the data are available for some quanta but not others is TBD.

    Supporting processing of multiple datasets together lets metrics be defined with a different granularity from the Science Pipelines processing, and allows for the aggregation (or lack thereof) of the metric to be controlled by the task configuration with no code changes.
    Note that if ``QAWG-REC-32`` is implemented, then the input data will typically be a list of one item.

``getInputDatasetTypes(config: cls.ConfigClass) : dict from str to DatasetTypeDescriptor [initially str to str]``
    While required by the ``PipelineTask`` API, this method will also be used by pre-``PipelineTask`` code to identify the (Butler Gen 2) inputs to the ``MetricTask``.

``getOutputMetric(config: cls.ConfigClass) : lsst.verify.Name``
    A class method returning the metric calculated by this object.
    May be configurable to allow one implementation class to calculate families of related metrics.

.. _components-primary-metrictask-concrete:

Concrete Members
^^^^^^^^^^^^^^^^

``getOutputDatasetTypes(config: cls.ConfigClass) : dict from str to DatasetTypeDescriptor``
    This method may need to be overridden to reflect Butler persistence of :class:`lsst.verify.Measurement` objects, if individual objects are not supported as a persistable dataset.

``saveStruct(lsst.pipe.base.Struct, outputDataRefs: dict, butler: lsst.daf.butler.Butler)``
    This method may need to be overridden to support Butler persistence of :class:`lsst.verify.Measurement` objects, if individual objects are not supported as a persistable dataset


.. _components-primary-metadatametrictask:

SingleMetadataMetricTask
------------------------

This class shall simplify implementations of metrics that are calculated from a single key in the pipeline's output metadata.
The class shall provide the code needed to map a metadata key (possibly across multiple quanta) to a single metric.

Based on the examples implemented in :mod:`lsst.ap.verify.measurements`, the process of calculating a metric from *multiple* metadata keys is considerably more complex.
It is better that such metrics inherit from ``MetricTask`` directly than to try to provide generic support through a single class.

.. _components-primary-metadatametrictask-abstract:

Abstract Members
^^^^^^^^^^^^^^^^

``getInputMetadataKey(config: cls.ConfigClass) : str``
    Shall name the key containing the metric information, with optional task prefixes following the conventions of :meth:`lsst.pipe.base.Task.getFullMetadata`.
    The name may be an incomplete key in order to match an arbitrary top-level task or an unnecessarily detailed key name.
    May be configurable to allow one implementation class to calculate families of related metrics.

``makeMeasurement(values: iterable of any) : lsst.verify.Measurement``
    A workhorse method that accepts the metadata values extracted from the metadata passed to ``run``.

.. _components-primary-metadatametrictask-concrete:

Concrete Members
^^^^^^^^^^^^^^^^

``run(metadata: iterable of lsst.daf.base.PropertySet) : lsst.pipe.base.Struct``
    This method shall take multiple metadata objects (possibly all of them, depending on the granularity of the metric).
    It shall look up keys partially matching ``getInputMetadataKey`` and make a single call to ``makeMeasurement`` with the values of the keys.
    Behavior when keys are present in some metadata objects but not others is TBD.

``getInputDatasetTypes(config: cls.ConfigClass) : dict from str to DatasetTypeDescriptor``
    This method shall return a single mapping from ``"metadata"`` to the dataset type of the top-level data processing task's metadata.
    The identity of the top-level task shall be extracted from the ``MetricTask``'s config.


.. _components-primary-ppdbmetrictask:

PpdbMetricTask
--------------

This class shall simplify implementations of metrics that are calculated from a prompt products database.

``PpdbMetricTask`` has a potential forward-compatibility problem: at present, the most expedient way to get a :class:`~lsst.dax.ppdb.Ppdb` that points to the correct database is by loading it from the data processing pipeline's config. However, the Butler is later expected to support database access directly, and we should adopt the new system when it is ready.

The problem can be solved by making use of the ``PipelineTask`` framework's existing support for configurable input dataset types, and by delegating the process of constructing a :class:`~lsst.dax.ppdb.Ppdb` object to a replaceable subtask.
The cost of this solution is an extra configuration line for every instance of ``PpdbMetricTask`` included in a metrics calculation, at least until we can adopt the new system as a default.

.. _components-primary-ppdbmetrictask-abstract:

Abstract Members
^^^^^^^^^^^^^^^^

``makeMeasurement(handle: lsst.dax.ppdb.Ppdb, outputDataId: DataId) : lsst.verify.Measurement``
    A workhorse method that takes a database handle and computes a metric using the :class:`~lsst.dax.ppdb.Ppdb` API.
    ``outputDataId`` is used to identify a specific metric for subclasses that support fine-grained metrics (see discussion of ``adaptArgsAndRun``, below).

``dbLoader : lsst.pipe.base.Task``
    A subtask responsible for creating a :class:`~lsst.dax.ppdb.Ppdb` object from the dataset type.
    Its ``run`` method must accept a dataset of the same type as indicated by ``PpdbMetricTask.getInputDatasetTypes``.

    Until plans for Butler database support are finalized, config writers should explicitly retarget this task instead of assuming a default.
    It may be possible to enforce this practice by not providing a default implementation and clearly documenting the supported option(s).

.. _components-primary-ppdbmetrictask-concrete:

Concrete Members
^^^^^^^^^^^^^^^^

``adaptArgsAndRun(dbInfo: dict from str to any, inputDataIds: unused, outputDataId: dict from str to DataId) : lsst.pipe.base.Struct``
    This method shall load the database using ``dbLoader`` before calling ``makeMeasurement``.
    ``PpdbMetricTask`` overrides ``adaptArgsAndRun`` in order to support fine-grained metrics: while a repository should have only one prompt products database, metrics may wish to examine subsets grouped by visit, CCD, etc., and if so these details must be passed to ``makeMeasurement``.

    This method is not necessary in the initial implementation, which will not support fine-grained metrics.

``run(dbInfo: any) : lsst.pipe.base.Struct``
    This method shall be a simplified version of ``adaptArgsAndRun`` for use before ``PipelineTask`` is ready.
    Its behavior shall be equivalent to ``adaptArgsAndRun`` called with empty data IDs.

``getInputDatasetTypes(config: cls.ConfigClass) : dict from str to DatasetTypeDescriptor``
    This method shall return a single mapping from ``"dbInfo"`` to a suitable dataset type: either the type of the top-level data processing task's config, or some future type specifically designed for database support.

.. _components-primary-metriccomputationerror:

MetricComputationError
----------------------

This subclass of :py:class:`RuntimeError` may be raised by ``MetricTask`` to indicate that a metric could not be computed due to algorithmic or system issues.
It is provided to let higher-level code distinguish failures in the metrics framework from failures in the pipeline code.

Note that being unable to compute a metric due to *insufficient* input data is not considered a failure, and in such a case ``MetricTask`` should return :py:const:`None` instead of raising an exception.

.. _components-compatibility:

Compatibility Components
========================

We expect to deploy new metrics before ``PipelineTask`` is ready for general use.
Therefore, the initial framework will include extra classes that allow ``MetricTask`` to function without ``PipelineTask`` features.

By far the best way to simultaneously deal with the incompatible Butler 2 and Butler 3 APIs would be an adapter class that allows ``MetricTask`` classes initially written without ``PipelineTask`` support to serve as :class:`lsst.pipe.base.PipelineTask`.
Unfortunately, the design of such an adapter is complicated by the strict requirements on :class:`~lsst.pipe.base.PipelineTask` constructor signatures and the use of configs as a :class:`~lsst.pipe.base.Task`'s primary API.

I suspect that both problems may be solved by applying a decorator to the appropriate :class:`type` objects rather than using a conventional class or object adapter\ :cite:`book:patterns` for :class:`~lsst.pipe.base.Task` or :class:`~lsst.pex.config.Config` objects, but the design of such an decorator is best addressed separately.

.. _components-compatibility-metrictask:

MetricTask
----------

This ``MetricTask`` shall be a subclass of :class:`~lsst.pipe.base.Task` that has a :class:`~lsst.pipe.base.PipelineTask`-like interface but does not depend on any Butler Gen 3 components. Concrete ``MetricTasks`` will implement this interface before ``PipelineTask`` is available, and can be migrated individually afterward (possibly through a formal deprecation procedure, if ``MetricTask`` is used widely enough to make it necessary).

.. _components-compatibility-metrictask-abstract:

Abstract Members
^^^^^^^^^^^^^^^^

``run(undefined) : lsst.pipe.base.Struct``
    Subclasses may provide a ``run`` method, which should take multiple datasets of a given type.
    Its return value must contain a field, ``measurement``, mapping to the resulting :class:`lsst.verify.Measurement`.

    ``MetricTask`` shall do nothing (returning :py:const:`None` in place of a :class:`~lsst.verify.Measurement`) if the data it needs are not available.
    Behavior when the data are available for some quanta but not others is TBD.

    Supporting processing of multiple datasets together lets metrics be defined with a different granularity from the Science Pipelines processing, and allows for the aggregation (or lack thereof) of the metric to be controlled by the task configuration with no code changes.
    Note that if ``QAWG-REC-32`` is implemented, then the input data will typically be a list of one item.

``adaptArgsAndRun(inputData: dict, inputDataIds: dict, outputDataId: dict) : lsst.pipe.base.Struct``
    The default implementation of this method shall be equivalent to calling ``PipelineTask.adaptArgsAndRun``, followed by calling ``addStandardMetadata`` on the result.
    Subclasses may override ``adaptArgsAndRun``, but are then responsible for calling ``addStandardMetadata`` themselves.

    ``outputDataId`` shall contain a single mapping from ``"measurement"`` to exactly one data ID.
    The method's return value must contain a field, ``measurement``, mapping to the resulting :class:`lsst.verify.Measurement`.

    Behavior requirements as for ``run``.

``getInputDatasetTypes(config: cls.ConfigClass) : dict from str to str``
    This method shall identify the Butler Gen 2 inputs to the ``MetricTask``.

``getOutputMetric(config: cls.ConfigClass) : lsst.verify.Name``
    A class method returning the metric calculated by this object.
    May be configurable to allow one implementation class to calculate families of related metrics.

.. _components-compatibility-metrictask-concrete:

Concrete Members
^^^^^^^^^^^^^^^^

``addStandardMetadata(measurement: lsst.verify.Measurement, outputDataId: dict)``
    This method may add measurement-specific metadata agreed to be of universal use (both across metrics and across clients, including but not limited to SQuaSH), breaking the method API if necessary.
    This method shall not add common information such as the execution environment (which is the responsibility of the ``MetricTask``'s caller) or information specific to a particular metric (which is the responsibility of the corresponding class).

    This is an unfortunately inflexible solution to the problem of adding client-mandated metadata keys.
    However, it is not clear whether any such keys will still be needed after the transition to Butler Gen 3 (see `SQR-019`_ and `DMTN-085`_), and any solution that controls the metadata using the task configuration would require independently configuring every single ``MetricTask``.

.. _components-compatibility-metricscontrollertask:

MetricsControllerTask
---------------------

This class shall execute a configurable set of metrics, handling Butler I/O and :class:`~lsst.verify.Measurement` output internally in a manner similar to :class:`~lsst.jointcal.JointcalTask`.
The ``MetricTask`` instances to be executed shall *not* be treated as subtasks, instead being managed using a multi-valued :class:`lsst.pex.config.RegistryField` much like ``meas_base`` plugins.

``MetricsControllerTask`` shall ignore any configuration in a ``MetricTask`` giving its metric a specific level of granularity; the granularity shall instead be inferred from ``MetricsControllerTask`` inputs.
In addition, ``MetricsControllerTask`` will not support metrics that depend on other metrics.

Some existing frameworks (i.e., :py:mod:`lsst.ap.verify` and :py:mod:`lsst.jointcal`) store metrics computed by a task as part of one or more :py:class:`lsst.verify.Job` objects.
``MetricsControllerTask`` will not be able to work with such jobs, but will not preempt them, either -- they can continue to record metrics that are not managed by ``MetricsControllerTask``.

.. _components-compatibility-metricscontrollertask-concrete:

Concrete Members
^^^^^^^^^^^^^^^^

``runDataRefs(datarefs: list of lsst.daf.persistence.ButlerDataRef) : lsst.pipe.base.Struct``
    This method shall, for each configured ``MetricTask`` and each ``dataref``, load the metric's input dataset(s) and pass them to the task (via ``adaptArgsAndRun``), collecting the resulting ``Measurement`` objects and persisting them to configuration-specified files.
    The return value shall contain a field, ``jobs``, mapping to a list of :class:`lsst.verify.Job`, one for each dataref, containing the measurements.

    The granularity of each ``dataref`` shall define the granularity of the corresponding measurement, and must be the same as or coarser than the granularity of each ``MetricTask's`` input data.
    The safest way to support metrics of different granularities is to handle each granularity with an independently configured ``MetricsControllerTask`` object.

    It is assumed that, since ``MetricsControllerTask`` is a placeholder, the implementation of ``runDataRefs`` will be something simple like a loop. However, it may use internal dataset caching or parallelism to speed things up if it proves necessary.

``measurers : iterable of MetricTask``
    This attribute contains all the metric measuring objects to be called by ``runDataRefs``.
    It is initialized from a :class:`~lsst.pex.config.RegistryField` in ``MetricsControllerConfig``.

``metadataAdder: lsst.pipe.base.Task``
    A subtask responsible for adding Job-level metadata required by a particular client (e.g., SQuaSH).
    Its ``run`` method must accept a :class:`lsst.verify.Job` object and return a :class:`lsst.pipe.base.Struct` whose ``job`` field maps to a modified :class:`~lsst.verify.Job`.

.. _components-compatibility-makemeasurerregistry:

MetricRegistry
--------------

This class shall expose a single instance of :class:`lsst.pex.config.Registry`.
``MetricsControllerConfig`` will depend on this class to create a valid :class:`~lsst.pex.config.RegistryField`.
It can be easily removed once ``MetricsControllerTask`` is retired.

Concrete Members
^^^^^^^^^^^^^^^^

``registry : lsst.pex.config.Registry``
    This registry will allow ``MetricsControllerConfig`` to handle all ``MetricTask`` classes decorated by ``register``.
    It should not require a custom subclass of :class:`lsst.pex.config.Registry`, but if the need arose, ``MetricRegistry`` could be easily turned into a singleton class.


.. _components-compatibility-register:

register
--------

``register(name: str) : callable(MetricTask-type)``
    This class decorator shall register the class with ``MetricRegistry.registry``.
    If ``MetricRegistry`` does not exist, it shall have no effect.

    This decorator can be phased out once ``MetricsControllerTask`` is retired.

.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa

.. _SQR-019: https://sqr-019.lsst.io/

.. _DMTN-057: https://dmtn-057.lsst.io/

.. _DMTN-085: https://dmtn-085.lsst.io/
