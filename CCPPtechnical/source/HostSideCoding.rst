.. _Host-side Coding:

**************************************************
Host Side Coding
**************************************************

This chapter describes the connection of a host model with the pool of :term:`CCPP Physics` :term:`schemes <scheme>` through the :term:`CCPP Framework`.

==================================================
Variable Requirements on the Host Model Side
==================================================

All variables required to communicate between the host model and the physics must be allocated by the host model. Variables needed to communicate between physics schemes can be allocated by the framework (e.g., Suite Variables); However, host models can still choose to allocate physics interstitial variables if they desire. The framework also controls several mandatory (control) variables ``errflg``, ``errmsg``, ``ccpp_suite``, ``group_name``, ``lb``, ``ub``, ``mythread``, ``nthreads``, and ``nphys_thread``, as explained in :numref:`Section %s <CCPPMandatory>`

At present, only two types of variable definitions are supported by the CCPP Framework:

* Standard Fortran variables (character, integer, logical, real) defined in a module or in the main program. For character variables, a fixed length is required. All others can have a kind attribute of a kind type defined by the host model. For variables of type ``real`` and ``complex`` without a ``kind`` attribute, the CCPP Framework will automatically assign the CCPP default floating point kind ``kind_phys`` to the variable's metadata. Pointers are not allowed as passable CCPP variables (though they may still be used internally by individual schemes).
* Derived data types (DDTs) defined in a module or the main program. While the use of DDTs as arguments to physics schemes in general is discouraged (see :numref:`Section %s <IOVariableRules>`), it is perfectly acceptable for the host model to define the variables requested by physics schemes as components of DDTs and pass these components to CCPP by using the correct local_name (e.g., ``myddt%thecomponentIwant``; see :numref:`Section %s <VariableTablesHostModel>`.)

.. _VariableTablesHostModel:

==================================================
Metadata for Variables in the Host Model
==================================================

To establish the link between host model variables and physics scheme variables, the host model must provide metadata information similar to those presented in :numref:`Section %s <MetadataRules>`. The host model can have multiple metadata files (``.meta``), each with the required ``[ccpp-table-properties]`` section and the related ``[ccpp-arg-table]`` sections. The host model Fortran files contain three-line snippets to indicate the location for insertion of the metadata information contained in the corresponding section in the ``.meta`` file.

.. _SnippetMetadata:

.. code-block:: fortran

   !!> \section arg_table_example_vardefs
   !! \htmlinclude example_vardefs.html
   !!

For each variable required by the pool of CCPP Physics schemes, one and only one entry must exist on the host model side. The connection between a variable in the host model and in the physics scheme is made through its ``standard_name``.

The following requirements must be met when defining metadata for variables in the host model (see also :ref:`Listing 6.1 <example_vardefs>`
and :ref:`Listing 6.2 <example_vardefs_meta>` for examples of host model metadata).

* The ``standard_name`` must match that of the target variable in the physics scheme.
* The shape and size of the variable (as defined in the host model Fortran code) must match that of the target variable. 
* The type and kind may differ between host-model and scheme, as the framework will automatically add these conversions.
* The attributes ``units``, ``dimensions``, ``type`` and ``kind`` in the host model metadata must match those in the physics scheme metadata.
* The attribute ``active`` is used to allocate variables under certain conditions.  It must be written as a Fortran expression that equates to ``.true.`` or ``.false.``, using the CCPP standard names of variables. ``active`` attributes for all variables are ``.true.`` by default. See :numref:`Section %s <ActiveAttribute>` for details.
* The ``intent`` attribute is not a valid attribute for host model metadata and will be ignored, if present.
* The ``local_name`` of the variable must be set to the name the host model cap uses to refer to the variable.
* Metadata sections describing module variables must be placed inside the module.
* Metadata sections describing components of DDTs must be placed immediately before the type definition and have the same name as the DDT.

.. _example_vardefs:

.. code-block:: fortran

       module example_vardefs

         implicit none

   !!> \section arg_table_example_vardefs
   !! \htmlinclude example_vardefs.html
   !!

         integer, parameter           :: r15 = selected_real_kind(15)
         integer                      :: ex_int
         real(kind=8), dimension(:,:) :: ex_real1

   !!> \section arg_table_example_ddt
   !! \htmlinclude example_ddt.html
   !!

         type ex_ddt
           logical                   :: l
           real(r15), dimension(:,:) :: r
         end type ex_ddt

         type(ex_ddt) :: ext

       end module example_vardefs


*Listing 6.1: Example host model file with reference to metadata. In this example, only the definition of a DDT* ``ex_ddt`` is included. The allocation of a variable of type ``ex_ddt`` occurs externally, see See :numref:`Section %s <DJS TODO>`.*

.. _example_vardefs_meta:

.. code-block:: fortran

   ########################################################################
   [ccpp-table-properties]
     name = arg_table_example_vardefs
     type = host
     dependencies =

   [ccpp-arg-table]
     name = arg_table_example_vardefs
     type = host
   [r15]
     standard_name = working_precision 
     long_name = working precision
     units = none
     dimensions = ()
     type = integer
   [ex_int]
     standard_name = example_int
     long_name = ex. int
     units = none
     dimensions = ()
     type = integer
   [ex_real]
     standard_name = example_real
     long_name = ex. real
     units = m
     dimensions = (horizontal_dimension,vertical_layer_dimension)
     type = real
     kind = kind=8

   ########################################################################
   [ccpp-table-properties]
     name = arg_table_example_ddt
     type = ddt
     dependencies =

   [ccpp-arg-table]
     name = arg_table_example_ddt
     type = ddt
   [ext%1]
     standard_name = example_flag
     long_name = ex. flag
     units = flag
     dimensions =
     type = logical
   [ext%r]
     standard_name = example_real
     long_name = ex. real
     units = kg
     dimensions = (horizontal_dimension,vertical_layer_dimension)
     type = real
     kind = r15
   [ext%r(;,1)]
     standard_name = example_slice
     long_name = ex. slice
     units = kg
     dimensions = (horizontal_dimension)
     type = real
     kind = r15


*Listing 6.2: Example host model metadata file (* ``.meta`` *).*


.. _ActiveAttribute:

,,,,,,,,,,,,,,,,
Active Attribute
,,,,,,,,,,,,,,,,

The CCPP must be able to detect when arrays need to be allocated, and when certain tracers must be
present in order to perform operations or tests in the auto-generated caps (e.g. unit conversions,
Association checks for optional scheme variables, etc.). This is accomplished with the attribute ``active`` in the
metadata for the host model variables (e.g., ``GFS_typedefs.meta`` for the :term:`UFS Atmosphere` or the :term:`SCM`).

Several arrays in the host model (e.g., ``GFS_typedefs.F90`` in the UFS Atmosphere or the SCM) are
allocated based on certain conditions, for example:

.. code-block:: fortran

    !--- needed for Thompson's aerosol option
    if(Model%imp_physics == Model%imp_physics_thompson .and. Model%ltaerosol) then
      allocate (Coupling%nwfa2d (IM))
      allocate (Coupling%nifa2d (IM))
      Coupling%nwfa2d   = clear_val
      Coupling%nifa2d   = clear_val
    endif

Other examples are the elements in the tracer array, where their presence depends on the corresponding
index being larger than zero. For example:

.. code-block:: fortran

    integer              :: ntwa            !< tracer index for water friendly aerosol
    ...
    Model%ntwa             = get_tracer_index(Model%tracer_names, 'liq_aero', ...)
    ...
    if (Model%ntwa>0) then
      ! do something with qgrs(:,:,Model%ntwa)
    end if

The ``active`` attribute is a conditional statement that, if true, will allow the corresponding variable
to be allocated.  It must be written as a Fortran expression that equates to ``.true.`` or ``.false.``,
using the CCPP standard names of variables. Active attributes for all variables are ``.true.`` by default.

To see how the active attribute is deployed with the group caps for optional scheme variables, see :numref:`Section %s <OptionalVariables>`.


If a developer adds a new variable that is only allocated under certain conditions, or changes the conditions
under which an existing variable is allocated, a corresponding change must be made in the metadata for the
host model variables (``GFS_typedefs.meta`` for the UFS Atmosphere or the SCM). See variables ``nwfa2d``
and ``qgrs`` in :ref:`Listing 6.2 <example_vardefs_meta>` for an example.

========================================================
CCPP Variables in the SCM and UFS Atmosphere Host Models
========================================================

While the use of standard Fortran variables is preferred, in the current implementation of the CCPP in the UFS Atmosphere and in the SCM almost all data is contained in DDTs for organizational purposes. In the case of the SCM and UFS Atmosphere, DDTs are defined in both ``GFS_typedefs.F90`` and ``CCPP_typedefs.F90``.  The current implementation of the CCPP in both :term:`host models <host model>` uses the following set of DDTs:

* ``GFS_init_type`` 		variables to allow proper initialization of GFS physics
* ``GFS_statein_type``	prognostic state data provided by dycore to physics
* ``GFS_stateout_type``	prognostic state after physical parameterizations
* ``GFS_sfcprop_type``	surface properties read in and/or updated by climatology, obs, physics
* ``GFS_coupling_type``	fields from/to coupling with other components, e.g., land/ice/ocean
* ``GFS_control_type``	control parameters input from a namelist and/or derived from others
* ``GFS_grid_type``		grid data needed for interpolations and length-scale calculations
* ``GFS_tbd_type``		data not yet assigned to a defined container
* ``GFS_cldprop_type``	cloud properties and tendencies needed by radiation from physics
* ``GFS_radtend_type``	radiation tendencies needed by physics
* ``GFS_diag_type``		fields targeted for diagnostic output to disk
* ``GFS_data_type``	combined type of all of the above except ``GFS_control_type``
* ``GFS_interstitial_type``     fields used to communicate variables among schemes in the :term:`slow physics` :term:`group` required to replace interstitial code that resided in ``GFS_{physics, radiation}_driver.F90`` in IPD
* ``GFDL_interstitial_type`` fields used to communicate variables among schemes in the :term:`fast physics` group

The DDT descriptions provide an idea of what physics variables go into which data type.  ``GFS_diag_type`` can contain variables that accumulate over a certain amount of time and are then zeroed out. Variables that require persistence from one timestep to another should not be included in the ``GFS_diag_type`` nor the ``GFS_interstitial_type`` DDTs. Similarly, variables that need to be shared between groups cannot be included in the ``GFS_interstitial_type`` DDT. Although this memory management is somewhat arbitrary, new variables provided by the host model or derived in an interstitial scheme should be put in a DDT with other similar variables.

Each DDT contains a create method that allocates the data defined using the metadata. For example, the ``GFS_stateout_type`` contains:

.. code-block:: fortran

 type GFS_stateout_type

    !-- Out (physics only)
    real (kind=kind_phys), pointer :: gu0 (:,:)   => null()  !< updated zonal wind
    real (kind=kind_phys), pointer :: gv0 (:,:)   => null()  !< updated meridional wind
    real (kind=kind_phys), pointer :: gt0 (:,:)   => null()  !< updated temperature
    real (kind=kind_phys), pointer :: gq0 (:,:,:) => null()  !< updated tracers

    contains
      procedure :: create  => stateout_create  !<   allocate array data
  end type GFS_stateout_type

In this example, ``gu0``, ``gv0``, ``gt0``, and ``gq0`` are defined in the host-side metadata section, and when the subroutine ``stateout_create`` is called, these arrays are allocated and initialized to zero.  With the CCPP, it is possible to not only refer to components of DDTs, but also to slices of arrays with provided metadata as long as these are contiguous in memory. An example of an array slice from the ``GFS_stateout_type`` looks like:

.. code-block:: fortran

  ########################################################################
  [ccpp-table-properties]
     name = GFS_stateout_type
     type = ddt
     dependencies =

   [ccpp-arg-table]
     name = GFS_stateout_type
     type = ddt
   ...
   [gq0]
     standard_name = tracer_concentration
     long_name = tracer concentration
     units = kg kg-1
     dimensions = (horizontal_dimension,vertical_layer_dimension,number_of_tracers)
     type = real
     kind = kind_phys
   ...
   [gq0(:,:,index_of_snow_mixing_ratio_in_tracer_concentration_array)]
     standard_name = snow_mixing_ratio_of_new_state
     long_name = ratio of mass of snow water to mass of dry air plus vapor (without condensates) updated by physics
     units = kg kg-1
     dimensions = (horizontal_dimension,vertical_layer_dimension)
     type = real
     kind = kind_phys

Array slices can be used by physics schemes that only require certain values from an array.

.. _CCPP_API:

========================================================
CCPP API
========================================================

The CCPP Application Programming Interface (API) is comprised of a set of clearly defined methods used to communicate variables between the host model and the physics and to run the physics. The API is automatically generated by the CCPP capgen script (see :numref:`Chapter %s <CCPPCapgen>`) and contains the subroutines ``ccpp_register``, ``ccpp_init``, ``ccpp_physics_init``, ``ccpp_physics_timestep_init``, ``ccpp_physics_run``, ``ccpp_physics_timestep_finalize``, and ``ccpp_physics_finalize`` (described below).

.. _CCPPMandatory:

,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
CCPP Mandatory (control) variables for Host and Scheme Coupling
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
 
Mandatory variables required by the CCPP framework are stored in a ``control`` metadata table. These variables are provided by the host model and passed directly through the CCPP API into the physics schemes. There are nine control variables:

* Error code for handling in CCPP (``errmsg``).
* Error message associated with the error code (``errflg``).
* CCPP suite name (``suite_name``)
* CCPP group name (``group_name``)
* Lower bound of horizontal_dimension (``lb``)
* Upper bound of horizontal_dimension (``ub``)
* Number of openMP threads(``nthreads``)
* Current openMP thread number(``mythread``)
* Number of openMP threads used by physics(``nphys_threads``)

.. code-block:: fortran

  module ccpp_driver

    use {host_name}_ccpp_cap, only: ccpp_register,               &
                                    ccpp_init,                   &
                                    ccpp_physics_init,           &
                                    ccpp_physics_timestep_init,  &
                                    ccpp_physics_run,            &
                                    ccpp_physics_timestep_final, &
                                    ccpp_physics_final,          &
                                    ccpp_final
    use iso_fortran_env,    only: error_unit
    implicit none
    ! CCPP control variables                                                                                                                                                                  
    integer :: mythread
    integer :: nthreads
    integer :: nphys_threads
    integer :: lb
    integer :: ub
    integer :: errflg
    character(len=512) :: errmsg
    character(len=256) :: ccpp_suite='undefined'
    character(len=256) :: group_name='undefined'
  end module ccpp_driver

*Listing 6.3: Example host model file containing mandatory CCPP control variables.

  ########################################################################
  [ccpp-table-properties]
    name = CCPP_driver
    type = control
    dependencies =

  [ccpp-arg-table]
    name = CCPP_driver
    type = control
  [ suite_name ]
    standard_name = suite_name
    long_name = name of the CCPP suite to dispatch to
    units = none
    dimensions = ()
    type = character
    kind = len=256
  [ group_name ]
    standard_name = group_name
    long_name = name of the CCPP group to dispatch to
    units = none
    dimensions = ()
    type = character
    kind = len=256
  [ lb ]
    standard_name = horizontal_loop_begin
    long_name = start of horizontal range for this phase
    units = index
    dimensions = ()
    type = integer
  [ ub ]
    standard_name = horizontal_loop_end
    long_name = end of horizontal range for this phase
    units = index
    dimensions = ()
    type = integer
  [ mythread ]
    standard_name = thread_number
    long_name = current thread number
    units = index
    dimensions = ()
    type = integer
  [ nthreads ]
    standard_name = number_of_threads
    long_name = total number of OpenMP threads
    units = count
    dimensions = ()
    type = integer
  [ nphys_threads ]
    standard_name = number_of_physics_threads
    long_name = thread budget for physics-internal OpenMP
    units = count
    dimensions = ()
    type = integer
  [ errmsg ]
    standard_name = ccpp_error_message
    long_name = error message for CCPP error handling
    units = none
    dimensions = ()
    type = character
    kind = len=512
  [ errflg ]
    standard_name = ccpp_error_code
    long_name = error flag for CCPP error handling
    units = 1
    dimensions = ()
    type = integer

*Listing 6.4: Mandatory variables that* **must** **be** **provided** *by the Host model*

Two of the variables are mandatory and must be passed to every physics scheme: ``errmsg`` and ``errflg``. The variables ``loop_cnt``, ``loop_max``, ``blk_no``, and ``thrd_no`` can be passed to the schemes if required, but are not mandatory. They are, however, required for the auto-generated caps to pass the correct data to the physics and to realize the subcycling of schemes. The ``cdata`` structure is only used to hold these six variables, since the host model variables are directly passed to the physics without the need for an intermediate data structure.

Note that ``cdata`` is not restricted to being a scalar but can be a multidimensional array, depending on the needs of the host model. For example, a model that uses a one-dimensional array of blocks for better cache-reuse and OpenMP threading to process these blocks in parallel may require ``cdata`` to be a two-dimensional array of size "number of blocks" x "number of OpenMP threads".

,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
Initializing and Finalizing the CCPP
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,

At the beginning of each run, the ``cdata`` structure needs to be set up. Similarly, at the end of each run, it needs to be terminated. This is done with subroutines ``ccpp_init`` and ``ccpp_finalize``. These subroutines should not be confused with ``ccpp_physics_init`` and ``ccpp_physics_finalize``, which were described in :numref:`Chapter %s <SuiteGroupCaps>`.

Note that optional arguments are denoted with square brackets.

.. _SuiteInitSubroutine:

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Suite Initialization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The :term:`suite` initialization step consists of allocating (if required) and initializing the ``cdata`` structure(s), it does not call the CCPP Physics or any auto-generated code. The simplest example is a suite initialization step that consists of initializing a scalar ``cdata`` instance with ``cdata%blk_no = 1`` and ``cdata%thrd_no = 1``.

A more complicated example is when multiple ``cdata`` structures are in use, namely one for the the CCPP phases that require access to all data of an MPI task (a scalar that is initialized in the same way as above), and one for the ``run`` phase, where chunks of blocked data are processed in parallel by multiple OpenMP threads, as shown in Listing :ref:`Listing 6.6 <SuiteInitComplicated>`.

.. _SuiteInitComplicated:

.. code-block:: fortran

   ...

   type(ccpp_t),                              target :: cdata_domain
   type(ccpp_t), dimension(:,:), allocatable, target :: cdata_block

   ! ccpp_suite is set during the namelist read by the host model
   character(len=256) :: ccpp_suite
   integer            :: nthreads

   ...

   ! Get and set number of OpenMP threads (module
   ! variable) that are available to run physics
   nthreads = omp_get_max_threads()

   ! For physics running over the entire domain,
   ! block and thread number are not used
   cdata_domain%blk_no  = 1
   cdata_domain%thrd_no = 1

   ! Allocate cdata structure for blocks and threads
   allocate(cdata_block(1:nblks,1:nthreads))

   ! Assign the correct block and thread numbers
   do nt=1,nthreads
     do nb=1,nblks
       cdata_block(nb,nt)%blk_no = nb
       cdata_block(nb,nt)%thrd_no = nt
     end do
   end do

*Listing 6.6: A more complex suite initialization step that consists of allocating and initializing multiple ``cdata`` structures.*

Depending on the implementation of CCPP in the host model, the suite name for the suite to be executed must be set in this step as well (omitted in Listing :ref:`Listing 6.6 <SuiteInitComplicated>`).

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Suite Finalization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The suite finalization consists of deallocating any ``cdata`` structures, if applicable, and optionally resetting scalar ``cdata`` instances as in the following example for the UFS:

.. code-block:: fortran

 deallocate(cdata_block)
 ! Optional
 cdata_domain%blk_no = -999
 cdata_domain%thrd_no = -999
 ...

,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
Running the Physics
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,

The physics is invoked by calling subroutine ``ccpp_physics_run``. This subroutine is part of the CCPP API and is auto-generated. This subroutine is capable of executing the physics with varying granularity, that is, a single group, or an entire suite can be run with a single subroutine call. Typical calls to ``ccpp_physics_run`` are below,where ``suite_name`` is mandatory and ``group_name`` is optional:

.. code-block:: fortran

 call ccpp_physics_run(cdata, suite_name, [group_name], ierr=ierr)

,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
Initializing and Finalizing the Physics
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,

Many (but not all) physical :term:`parameterizations <parameterization>` need to be initialized, which includes functions such as reading lookup tables, reading input datasets, computing derived quantities, broadcasting information to all MPI ranks, etc. Initialization procedures are done for the entire domain, that is, they are not subdivided by blocks and need access to all data that an MPI task owns. Similarly, many (but not all) parameterizations need to be finalized, which includes functions such as deallocating variables, resetting flags from *initialized* to *non-initialized*, etc. Initialization and finalization functions are each performed once per run, before the first call to the physics and after the last call to the physics, respectively. They may not contain thread-dependent or block-dependent information.

The initialization and finalization can be invoked for a single group, or for the entire suite. In both cases, subroutines ``ccpp_physics_init`` and ``ccpp_physics_finalize`` are used and the arguments passed to those subroutines determine the type of initialization.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Subroutine ``ccpp_physics_init``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This subroutine is part of the CCPP API and is auto-generated. A typical call to ``ccpp_physics_init`` is:

.. code-block:: fortran

 call ccpp_physics_init(cdata, suite_name, [group_name], ierr=ierr)

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Subroutine ``ccpp_physics_finalize``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This subroutine is part of the CCPP API and is auto-generated. A typical call to ``ccpp_physics_finalize`` is:

.. code-block:: fortran

 call ccpp_physics_finalize(cdata, suite_name, [group_name], ierr=ierr)

,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,
Initializing and Finalizing the time step
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,,

The time step initialization typically consists of updating quantities that depend on the valid time, for example solar insulation angle, aerosol emission rates and other values obtained from climatologies. Like the physics initialization and finalization steps, the time step intialization and finalization steps need access to the entire data of an MPI task and may not contain thread-dependent or block-dependent information.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Subroutine ``ccpp_physics_timestep_init``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This subroutine is part of the CCPP API and is auto-generated.A typical call to ``ccpp_physics_timestep_init`` is:

.. code-block:: fortran

 call ccpp_physics_timestep_init(cdata, suite_name, [group_name], ierr=ierr)

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Subroutine ``ccpp_physics_timestep_finalize``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This subroutine is part of the CCPP API and is auto-generated.  A typical call to ``ccpp_physics_timestep_finalize`` is:

.. code-block:: fortran

 call ccpp_physics_timestep_finalize(cdata, suite_name, [group_name], ierr=ierr)

========================================================
Host Caps
========================================================

The purpose of the host model *cap* is to abstract away the communication between the host model and the CCPP Physics schemes. While CCPP calls can be placed directly inside the host model code (as is done for the relatively simple SCM), it is recommended to separate the *cap* in its own module for clarity and simplicity (as is done for the UFS Atmosphere). While the details of implementation will be specific to each host model, the host model *cap* is responsible for the following general functions:

* Allocating memory for variables needed by physics

  * All variables needed to communicate between the host model and the physics, and all variables needed to communicate among physics schemes, need to be allocated by the host model. The latter, for example for interstitial variables used exclusively for communication between the physics schemes, are typically allocated in the *cap*.

* Allocating and initializing the ``cdata`` structure(s) and setting the suite name (suite initialization)

* Providing interfaces to call the CCPP

  * The *cap* must provide functions or subroutines that can be called at the appropriate places in the host model time integration loop and that internally call ``ccpp_physics_init``, ``ccpp_physics_timestep_init``, ``ccpp_physics_run``, ``ccpp_physics_timestep_finalize`` and ``ccpp_physics_finalize``, and handle any errors returned. :ref:`Listing 6.7 <example_ccpp_host_cap>` provides an example where the host cap consists of three subroutines ``physics_init`` (which consists of the suite initialization and CCPP physics init phase), ``physics_run`` (which internally performs the CCPP time step init, run, and time step finalize phases), and ``physics_finalize`` (which consists of the suite finalization and CCPP physics finalize phase).

.. _example_ccpp_host_cap:

.. code-block:: fortran

 module example_ccpp_host_cap

  use ccpp_types,         only: ccpp_t
  use ccpp_static_api,    only: ccpp_physics_init,              &
                                ccpp_physics_timestep_init,     &
                                ccpp_physics_run,               &
                                ccpp_physics_timestep_finalize, &
                                ccpp_physics_finalize

   implicit none
   ! CCPP data structure
   type(ccpp_t), save, target :: cdata
   public :: physics_init, physics_run, physics_finalize
 contains

  subroutine physics_init(ccpp_suite_name)
    character(len=*), intent(in) :: ccpp_suite_name
    integer :: ierr
    ierr = 0

    ! Initialize cdata
    cdata%blk_no = 1
    cdata%thrd_no = 1

    ! Initialize CCPP physics (run all _init routines)
    call ccpp_physics_init(cdata, suite_name=trim(ccpp_suite_name),      &
                           ierr=ierr)

  end subroutine physics_init

  subroutine physics_run(ccpp_suite_name, group)
    ! Optional argument group can be used to run a group of schemes      &
    ! defined in the SDF. Otherwise, run entire suite.
    character(len=*),           intent(in) :: ccpp_suite_name
    character(len=*), optional, intent(in) :: group

    integer :: ierr
    ierr = 0

    if (present(group)) then
       call ccpp_physics_timestep_init(cdata,                            &
                             suite_name=trim(ccpp_suite_name),           &
                             group_name=group, ierr=ierr)
       call ccpp_physics_run(cdata, suite_name=trim(ccpp_suite_name),    &
                             group_name=group, ierr=ierr)
       call ccpp_physics_timestep_finalize(cdata,                        &
                             suite_name=trim(ccpp_suite_name),           &
                             group_name=group, ierr=ierr)
    else
       call ccpp_physics_timestep_init(cdata,                            &
                             suite_name=trim(ccpp_suite_name), ierr=ierr)
       call ccpp_physics_run(cdata, suite_name=trim(ccpp_suite_name),    &
                             ierr=ierr)
       call ccpp_physics_timestep_finalize(cdata,                        &
                             suite_name=trim(ccpp_suite_name), ierr=ierr)
    end if

  end subroutine physics_run

  subroutine physics_finalize(ccpp_suite_name)
    character(len=*), intent(in) :: ccpp_suite_name
    integer :: ierr
    ierr = 0

    ! Finalize CCPP physics (run all _finalize routines)
    call ccpp_physics_finalize(cdata, suite_name=trim(ccpp_suite_name),  &
                               ierr=ierr)

    ! Reset cdata
    cdata%blk_no = -999
    cdata%thrd_no = -999

  end subroutine physics_finalize

 end module example_ccpp_host_cap

*Listing 6.7: Fortran template for a CCPP host model cap. After each call to ``ccpp_physics_*``, the host model should check the return code ``ierr`` and handle any errors (omitted for readability).*

Readers are referred to the actual implementations of the cap functions in the CCPP-SCM and the UFS for further information. For the SCM, the cap functions are implemented in:

* ``ccpp-scm/scm/src/scm.F90``
* ``ccpp-scm/scm/src/scm_type_defs.F90``
* ``ccpp-scm/scm/src/scm_setup.F90``
* ``ccpp-scm/scm/src/scm_time_integration.F90``

For the UFS, the cap functions can be found in ``ufs-weather-model/FV3/ccpp/driver/CCPP_driver.F90``.


