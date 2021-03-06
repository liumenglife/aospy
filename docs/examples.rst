.. _examples:

########
Examples
########

.. note::

   The footnotes in this section provide scientific background to help
   you understand the motivation and physical meaning of these example
   calculations.  They can be skipped if you are familiar already or
   aren't interested in those details.

In this section, we use the `example data files
<https://github.com/spencerahill/aospy/tree/develop/aospy/test/data/netcdf>`_
included with aospy to demonstrate the standard aospy workflow of
executing and submitting multiple calculations at once.

These files contain timeseries of monthly averages of two variables
generated by an idealized [#idealized]_ aquaplanet [#aquaplanet]_
climate model:

1. Precipitation generated through gridbox-scale condensation
2. Precipitation generated through the model's convective
   parameterization [#var-defs]_

Using this data that was directly outputted by the model, let's
compute two other useful quantities: (1) the total precipitation rate,
and (2) the fraction of the total precipitation rate that comes from
the convective parameterization.  We'll compute the time-average over
the whole duration of the data, both at each gridpoint and aggregated
over specific regions.

Preliminaries
-------------

First we'll save the path to the example data in a local variable,
since we'll be using it in several places below.

.. ipython:: python

   import os  # Python built-in package for working with the operating system
   import aospy
   rootdir = os.path.join(aospy.__path__[0], 'test', 'data', 'netcdf')

Now we'll use the fantastic `xarray
<http://xarray.pydata.org/en/stable/>`_ package to inspect the data:

.. ipython:: python

   import xarray as xr
   xr.open_mfdataset(os.path.join(rootdir, '000[4-6]0101.precip_monthly.nc'),
                     decode_times=False)

We see that, in this particular model, the variable names for these
two forms of precipitation are "condensation_rain" and
"convection_rain", respectively.  The file also includes the
coordinate arrays ("lat", "time", etc.) that indicate where in space
and time the data refers to.

Now that we know where and what the data is, we'll proceed through the
workflow described in the :ref:`Using aospy <using-aospy>` section of
this documentation.

Describing your data
--------------------

Runs and DataLoaders
====================

First we create an :py:class:`aospy.Run` object that stores metadata
about this simulation.  This includes specifying where its files are
located via an :py:class:`aospy.data_loader.DataLoader` object.

DataLoaders specify where your data is located and organized.  Several
types of DataLoaders exist, each for a different directory and file
structure; see the :ref:`api-ref` for details.

For our simple case, where the data comprises a single file, the
simplest DataLoader, a ``DictDataLoader``, works well.  It maps your
data based on the time frequency of its output (e.g. 6 hourly, daily,
monthly, annual) to the corresponding netCDF files via a simple
dictionary:

.. ipython:: python

    from aospy.data_loader import DictDataLoader
    file_map = {'monthly': os.path.join(rootdir, '000[4-6]0101.precip_monthly.nc')}
    data_loader = DictDataLoader(file_map)

We then pass this to the :py:class:`aospy.Run` constructor, along with
a name for the run and an optional description.

.. ipython:: python

    from aospy import Run
    example_run = Run(
        name='example_run',
        description='Control simulation of the idealized moist model',
        data_loader=data_loader,
        default_start_date='0004',
        default_end_date='0006'
    )

.. note::

   Throughout aospy, date slice bounds can be specified with dates of any of
   the following types: 

   - ``str``, for partial-datetime string indexing
   - ``np.datetime64``
   - ``datetime.datetime``
   - ``cftime.datetime``

   If possible, aospy will automatically convert the latter three to the
   appropriate date type used for indexing the data read in; otherwise it will
   raise an error.  Therefore the arguments ``default_start_date`` and
   ``default_end_date`` in the ``Run`` constructor are calendar-agnostic (as
   are the ``date_ranges`` specified :ref:`when submitting
   calculations<Submitting calculations>`).
    
.. note::

   See the :ref:`API reference <api-ref>` for other optional arguments
   for this and the other core aospy objects used in this tutorial.

.. note::

   An important consideration can be the datatype used to store values in
   your datasets.  In particular, if the float32 datatype is used in
   storage, it can lead to undesired inaccuracies in the computation of
   reduction operations (like means) due to upstream issues (see
   `pydata/xarray#1346 <https://github.com/pydata/xarray/issues/1346>`_ for
   more information).  To address this it is recommended to always upcast
   float32 data to float64.  This behavior is turned on by default.  If you
   would like to disable this behavior you can set the ``upcast_float32``
   argument in your ``DataLoader`` constructors to ``False``.

Models
======

Next, we create the :py:class:`aospy.Model` object that describes the
model in which the simulation was executed.  One important attribute
is ``grid_file_paths``, which consists of a sequence (e.g. a tuple or
list) of paths to netCDF files from which physical attributes of that
model can be found that aren't already embedded in the output netCDF
files.

For example, often the land mask that defines which gridpoints are
ocean or land is outputted to a single, standalone netCDF file, rather
than being included in the other output files.  But we often need the
land mask, e.g. to define certain land-only or ocean-only
regions. [#land-mask]_ This and other grid-related properties shared
across all of a Model's simulations can be found in one or more of the
files in ``grid_file_paths``.

The other important attribute is ``runs``, which is a list of the
:py:class:`aospy.Run` objects that pertain to simulations performed in
this particular model.

.. ipython:: python

    from aospy import Model
    example_model = Model(
        name='example_model',
        grid_file_paths=(
            os.path.join(rootdir, '00040101.precip_monthly.nc'),
            os.path.join(rootdir, 'im.landmask.nc')
        ),
        runs=[example_run]  # only one Run in our case, but could be more
    )

Projects
========

Finally, we associate the ``Model`` object with an
:py:class:`aospy.Proj` object.  This is the level at which we specify
the directories to which aospy output gets written.

.. ipython:: python

    from aospy import Proj
    example_proj = Proj(
        'example_proj',
        direc_out='example-output',  # default, netCDF output (always on)
        tar_direc_out='example-tar-output', # output to .tar files (optional)
        models=[example_model]  # only one Model in our case, but could be more
    )

This extra :py:class:`aospy.Proj` level of organization may seem like
overkill for this simple example, but it really comes in handy once
you start using aospy for more than one project.

Defining physical quantities and regions
----------------------------------------

Having now fully specified the particular data of interest, we now
define the general physical quantities of interest and any geographic
regions over which to aggregate results.

Physical variables
==================

We'll first define :py:class:`aospy.Var` objects for the two variables
that we saw are directly available as model output:

.. ipython:: python

   from aospy import Var

   precip_largescale = Var(
       name='precip_largescale',  # name used by aospy
       alt_names=('condensation_rain',),  # its possible name(s) in your data
       def_time=True,  # whether or not it is defined in time
       description='Precipitation generated via grid-scale condensation',
   )
   precip_convective = Var(
       name='precip_convective',
       alt_names=('convection_rain', 'prec_conv'),
       def_time=True,
       description='Precipitation generated by convective parameterization',
   )

When it comes time to load data corresponding to either of these from
one or more particular netCDF files, aospy will search for variables
matching either ``name`` or any of the names in ``alt_names``,
stopping at the first successful one.  This makes the common problem
of model-specific variable names a breeze: if you end up with data
with a new name for your variable, just add it to ``alt_names``.

.. warning::

   This assumes that the name and all alternate names are unique to
   that variable, i.e. that in none of your data do those names
   actually signify something else.  If that was indeed the case,
   aospy can potentially grab the wrong data without issuing an error
   message or warning.

Next, we'll create functions that compute the total precipitation and
convective precipitation fraction and combine them with the above
:py:class:`aospy.Var` objects to define the new :py:class:`aospy.Var`
objects:

.. ipython:: python

    def total_precip(condensation_rain, convection_rain):
	"""Sum of large-scale and convective precipitation."""
        return condensation_rain + convection_rain

    def conv_precip_frac(precip_largescale, precip_convective):
	"""Fraction of total precip that is from convection parameterization."""
	total = total_precip(precip_largescale, precip_convective)
	return precip_convective / total.where(total)


    precip_total = Var(
	name='precip_total',
	def_time=True,
	func=total_precip,
	variables=(precip_largescale, precip_convective),
    )

    precip_conv_frac = Var(
       name='precip_conv_frac',
       def_time=True,
       func=conv_precip_frac,
       variables=(precip_largescale, precip_convective),
    )

Notice the ``func`` and ``variables`` attributes that weren't in the
prior ``Var`` constuctors.  These signify the function to use and the
physical quantities to pass to that function in order to compute the
quantity.

As of aospy version 0.3, ``Var`` objects are computed
recursively; this means that as long as things eventually lead back to
model-native quantities, you can express a computed variable (i.e. one
with ``func`` and ``variables`` attributes) in terms of other computed
variables.  For example we could equivalently express the
``precip_conv_frac`` more simply as the following:

.. ipython::

   precip_conv_frac = Var(
       name='precip_conv_frac',
       def_time=True,
       variables=(precip_convective, precip_total),
       func=lambda conv, total: conv / total,
   )

In this case, aospy will automatically know to load in
``precip_largescale`` and ``precip_convective`` in order to compute
``precip_total`` before passing it along to the function specified
in ``precip_conv_frac``.  Any depth of recursion is supported.
   
.. note::

   Although ``variables`` is passed a tuple of ``Var`` objects
   corresponding to the physical quantities passed to ``func``,
   ``func`` should be a function whose arguments are the
   :py:class:`xarray.DataArray` objects corresponding to those
   variables.  aospy uses the ``Var`` objects to load the DataArrays
   and then passes them to the function.

   This enables you to write simple, expressive functions comprising
   only the physical operations to perform (since the "data wrangling"
   part has been handled already).

.. warning::

   Order matters in the tuple of :py:class:`aospy.Var` objects passed
   to the ``variables`` attribute: it must match the order of the call
   signature of the function passed to ``func``.

   E.g. in ``precip_conv_frac`` above, if we had mistakenly done
   ``variables=(precip_convective, precip_largescale)``, the
   calculation would execute without error, but all of the results
   would be physically wrong.

Geographic regions
==================

Last, we define the geographic regions over which to perform
aggregations and add them to ``example_proj``.  We'll look at the
whole globe and at the Tropics:

.. ipython:: python

    from aospy import Region
    globe = Region(
        name='globe',
        description='Entire globe',
        west_bound=0,
        east_bound=360,
        south_bound=-90,
        north_bound=90,
        do_land_mask=False
    )

    tropics = Region(
	name='tropics',
	description='Global tropics, defined as 30S-30N',
        west_bound=0,
        east_bound=360,
        south_bound=-30,
        north_bound=30,
	do_land_mask=False
    )
    example_proj.regions = [globe, tropics]

We now have all of the needed metadata in place.  So let's start
crunching numbers!

Submitting calculations
-----------------------

Using :py:func:`aospy.submit_mult_calcs`
========================================

.. _Submitting calculations:

Having put in the legwork above of describing our data and the
physical quantities we wish to compute, we can submit our desired
calculations for execution using :py:func:`aospy.submit_mult_calcs`.
Its sole required argument is a dictionary specifying all of the
desired parameter combinations.

In the example below, we import and use the ``example_obj_lib`` module
that is included with aospy and whose objects are essentially
identical to the ones we've defined above.

.. ipython:: python

    from aospy.examples import example_obj_lib as lib

    calc_suite_specs = dict(
	library=lib,
	projects=[lib.example_proj],
	models=[lib.example_model],
	runs=[lib.example_run],
	variables=[lib.precip_largescale, lib.precip_convective,
                   lib.precip_total, lib.precip_conv_frac],
	regions='all',
	date_ranges='default',
	output_time_intervals=['ann'],
	output_time_regional_reductions=['av', 'reg.av'],
	output_vertical_reductions=[None],
	input_time_intervals=['monthly'],
	input_time_datatypes=['ts'],
	input_time_offsets=[None],
	input_vertical_datatypes=[False],
    )

See the :ref:`api-ref` on :py:func:`aospy.submit_mult_calcs` for more
on ``calc_suite_specs``, including accepted values for each key.

:py:func:`submit_mult_calcs` also accepts a second dictionary
specifying some options regarding how we want aospy to display,
execute, and save our calculations.  For the sake of this simple
demonstration, we'll suppress the prompt to confirm the calculations,
submit them in serial rather than parallel, and suppress writing
backup output to .tar files:

.. ipython:: python

    calc_exec_options = dict(prompt_verify=False, parallelize=False,
                             write_to_tar=False)

Now let's submit this for execution:

.. ipython:: python

    from aospy import submit_mult_calcs
    calcs = submit_mult_calcs(calc_suite_specs, calc_exec_options)

This permutes over all of the parameter settings in
``calc_suite_specs``, generating and executing the resulting
calculation.  In this case, it will compute all four variables and
perform annual averages, both for each gridpoint and regionally
averaged.

Although we do not show it here, this also prints logging information
to the terminal at various steps during each calculation, including
the filepaths to the netCDF files written to disk of the results.

.. warning::

   For date ranges specified using tuples of datetime-like objects,
   ``aospy`` will check to make sure that datapoints exist for the full extent
   of the time ranges specified.  For date ranges specified as tuples of
   strings, however, this check is currently not implemented.  This is mostly
   harmless (i.e. it will not change the results of calculations); however, it
   can result in files whose labels do not accurately represent the actual
   time bounds of the calculation if you specify string date ranges that span
   more than the interval of the input data.

Results
=======

The result is a list of :py:class:`aospy.Calc` objects, one per
simulation.

.. ipython:: python

    calcs

Each :py:class:`aospy.Calc` object includes the paths to the output

.. ipython:: python

    calcs[0].path_out

and the results of each output type

.. ipython:: python

    calcs[0].data_out

.. note::

    Notice that the variable's name and description have been copied
    to the resulting Dataset (and hence also to the netCDF file saved
    to disk). This enables you to better understand what the physical
    quantity is, even if you don't have the original ``Var`` definition
    on hand.

Gridpoint-by-gridpoint
~~~~~~~~~~~~~~~~~~~~~~

Let's plot (using `matplotlib <http://matplotlib.org/>`_) the time
average at each gridcell of all four variables.  For demonstration
purposes, we'll load the data that was saved to disk using xarray
rather than getting it directly from the ``data_out`` attribute as
above.

.. ipython:: python

    from matplotlib import pyplot as plt

    fig = plt.figure()

    for i, calc in enumerate(calcs):
        ax = fig.add_subplot(2, 2, i+1)
	arr = xr.open_dataset(calc.path_out['av']).to_array()
	if calc.name != precip_conv_frac.name:
	    arr *= 86400  # convert to units mm per day
	arr.plot(ax=ax)
	ax.set_title(calc.name)
	ax.set_xticks(range(0, 361, 60))
	ax.set_yticks(range(-90, 91, 30))

    plt.tight_layout()

    @savefig plot_av.png width=100%
    plt.show()

We see that precipitation maximizes at the equator and has a secondary
maximum in the mid-latitudes. [#itcz]_ Also, the convective
precipitation dominates the total in the Tropics, but moving poleward
the gridscale condensation plays an increasingly larger fractional
role (note different colorscales in each panel). [#ls-conv]_

Regional averages
~~~~~~~~~~~~~~~~~

Now let's examine the regional averages.  We find that the global
annual mean total precipitation rate for this run (converting to units
of mm per day) is:

.. ipython:: python

    for calc in calcs:
        ds = xr.open_dataset(calc.path_out['reg.av'])
	if calc.name != precip_conv_frac.name:
	    ds *= 86400  # convert to units mm/day
	print(calc.name, ds, '\n')

As was evident from the plots, we see that most precipitation (80.8%)
in the tropics comes from convective rainfall, but averaged over the
globe the large-scale condensation is a more equal player (40.2% for
large-scale, 59.8% for convective).

Beyond this simple example
--------------------------

Scaling up
==========

In this case, we computed time averages of four variables, both at
each gridpoint (which we'll call 1 calculation) and averaged over two
regions, yielding (4 variables)*(1 gridcell operation + (2 regions)*(1
regional operation)) = 12 total calculations executed.  Not bad, but
12 calculations is few enough that we probably could have handled them
without aospy.

The power of aospy is that, with the infrastructure we've put in
place, we can now fire off additional calculations at any time.  Some
examples:

- Set ``output_time_regional_reductions=['ts', 'std', 'reg.ts',
  'reg.std']`` : calculate the timeseries ('ts') and standard
  deviation ('std') of annual mean values at each gridpoint and for
  the regional averages.
- Set ``output_time_intervals=range(1, 13)`` : average across years
  for each January (1), each February (2), etc. through December
  (12). [#seasonal]_

With these settings, the number of calculations is now (4
variables)*(2 gridcell operations + (2 regions)*(2 regional
operations))*(12 temporal averages) = 288 calculations submitted with
a single command.

Modifying your object library
=============================

We can also add new objects to our object library at any time.  For
example, suppose we performed a new simulation in which we modified
the formulation of the convective parameterization.  All we would have
to do is create a corresponding :py:class:`aospy.Run` object, and then
we can execute calculations for that simulation.  And likewise for
models, projects, variables, and regions.

As a real-world example, two of aospy's developers use aospy for in
their own scientific research, with multiple projects each comprising
multiple models, simulations, etc.  They routinely fire off thousands
of calculations at once.  And thanks to the highly organized and
metadata-rich directory structure and filenames of the aospy output
netCDF files, all of the resulting data is easy to find and use.

Example "main" script
=====================

Finally, aospy comes included with a "main" script for submitting
calculations that is pre-populated with the objects from the example
object library.  It also comes with in-line instructions on how to use
it, whether you want to keep playing with the example library or
modify it to use on your own object library.

It is located in "examples" directory of your aospy installation.
Find it via typing ``python -c "import os, aospy;
print(os.path.join(aospy.__path__[0], 'examples', 'aospy_main.py'))"``
from your terminal.

.. ipython:: python
    :suppress:

    from shutil import rmtree
    rmtree('example-output')
    rmtree('example-tar-output')

.. rubric:: Footnotes

.. [#idealized]

   An "idealized climate model" is a model that, for the sake of
   computational efficiency and conceptual simplicity, omits and/or
   simplifies various processes relative to how they are computed in
   full, production-class models.  The particular model used here is
   described in `Frierson et al 2006
   <https://doi.org/10.1175/JAS3753.1>`_.

.. [#aquaplanet]

   An "aquaplanet" is simply a climate model in which the the surface
   is entirely ocean, i.e. there is no land.  Interactions between
   atmospheric and land processes are complicated, and so an
   aquaplanet avoids those complications while still generating a
   climate (when zonally averaged, i.e. averaged around each latitude
   circle) that roughly resembles that of the real Earth's.

.. [#var-defs]

   Most climate models generate precipitation through two separate
   pathways: (1) direct saturation of a whole gridbox, which results
   in condensation and precipitation, and (2) a "convective
   parameterization."  The latter simulates the precipitation that,
   due to subgrid-scale variability, can be expected to occur at some
   fraction of the area within a gridcell, even though the cell as a
   whole isn't saturated.  The total precipitation is simply the sum
   of these "large-scale" and "convective" components.

.. [#land-mask]

   In this case, the model being used is an aquaplanet, so the mask
   will be simply all ocean.  But this is not generally the case --
   comprehensive climate and weather models include Earth's full
   continental geometry and land topography (at least as well as can
   be resolved at their particular horizontal grid resolution).

.. [#itcz]

   This equatorial rainband is called the Intertropical Convergence
   Zone, or ITCZ.  In this simulation, the imposed solar radiation is
   fixed at Earth's annual mean value, which is symmetric about the
   equator.  The ITCZ typically follows the solar radiation maximum,
   hence its position in this case directly on the equator.

.. [#ls-conv]

   This is a very common result.  The gridcells of many climate models
   are several hundred kilometers by several hundred kilometers in
   area.  In Earth's Tropics, most rainfall is generated by cumulus
   towers that are much smaller than this.  But in the mid-latitudes,
   a phenomenon known as baroclinic instability generates much larger
   eddies that can span several hundred kilometers.

.. [#seasonal]

   In this particular simulation, the boundary conditions are constant
   in time, so there is no seasonal cycle.  But we could use these
   monthly averages to confirm that's actually the case, i.e. that we
   didn't accidentally use time-varying solar radiation when we ran
   the model.
