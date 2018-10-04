*************************
Python Interface to NeXus
*************************
The Python interface to NeXus is provided by the `nexusformat 
<https://github.com/nexpy/nexusformat>`_ package, which is distributed 
separately from NeXpy. It can be used within a standard Python or IPython 
shell:: 

 $ python
 Python 3.6.1 |Anaconda custom (x86_64)| (default, May 11 2017, 13:04:09) 
 [GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.57)] on darwin
 Type "help", "copyright", "credits" or "license" for more information.
 >>> from nexusformat.nexus import *

.. note:: Although wildcard imports are usually discouraged in Python, there
          should not be any name clashes using it here because all the 
          imported functions and variables start with 'nx' or 'NX'.

Loading NeXus Data
==================
The entire tree structure of a NeXus file can be loaded by a single command::

 >>> a=nxload('sns/data/ARCS_7326_tof.nxs')

The assigned variable now contains the entire tree structure of the file, which 
can be displayed by printing the 'tree' property::

 >>> print(a.tree)
 root:NXroot
  @HDF5_Version = '1.8.2'
  @NeXus_version = '4.2.1'
  @file_name = 'ARCS_7326_tof.nxs'
  @file_time = '2010-05-05T01:59:25-05:00'
  entry:NXentry
    data:NXdata
      @axes = ['rotation_angle' 'tilt_angle' 'sample_angle' 'time_of_flight']
      @signal = 'data'
      data = float32(631x461x4x825)
      rotation_angle = float32(632)
        @units = 'degree'
      sample_angle = float32(5)
        @units = 'degree'
      tilt_angle = float32(462)
        @units = 'degree'
      time_of_flight = float32(826)
        @units = 'microsecond'
    run_number = '7326'
    sample:NXsample
      pulse_time = 2854.947473649946
        @units = 'microsecond'

Individual data items are immediately available from the command-line::

 >>> print(a.entry.run_number)
 7326

Note that only the tree structure and the values of smaller data sets are read
from the file to avoid using up memory unnecessarily. In the above example, only
the types and dimensions of the larger data sets are displayed in the tree.
Data is loaded only when it is needed, for plotting or calculations, either as 
a complete array, if memory allows, or as a series of slabs (see below).

.. note:: The variable NX_MEMORY defines the maximum size in MB of data that 
          will be read from a file. If the NXfield is larger than NX_MEMORY, the
          data will have to be read as a series of slabs. The default value is
          500.

Load Options
------------
There is a second optional argument to the load module that defines the access
mode for the existing data. For example, the following opens the file in 
read/write mode::

 >>> a=nxload('chopper.nxs', mode='rw')

The default mode is 'r', *i.e.*, readonly access.

.. warning:: If the file is opened in read/write mode, any changes are made 
             automatically to the file itself. In particular, any deletions of 
             file objects will be irreversible. Make sure you have a backup
             if you open a mission-critical file. In the :doc:`pythongui`, this 
             can be achieved using the 'Duplicate...' menu item.

Creating NeXus Data
===================
It is just as easy to create new NeXus data sets from scratch using Numpy 
arrays. The following example shows the creation of a simple function, which is 
then saved to a file::
 
 >>> import numpy as np
 >>> x=y=np.linspace(0,2*np.pi,101)
 >>> X,Y=np.meshgrid(x,y)
 >>> z=np.sin(X)*np.sin(Y)
 >>> a=NXdata(z,[y,x])
 >>> a.save('function.nxs')

This file can then be loaded again::

 >>> b=nxload('function.nxs')
 >>> print(b.tree)
 root:NXroot
  @HDF5_Version = '1.8.17'
  @file_name = 'function.nxs'
  @file_time = '2017-10-12T20:02:53.953375'
  @h5py_version = '2.7.0'
  @nexusformat_version = '0.4.10'
  entry:NXentry
    data:NXdata
      @axes = ['axis1' 'axis2']
      @signal = 'signal'
      axis1 = float64(101)
      axis2 = float64(101)
      signal = float64(101x101)

.. note:: The save() method automatically wraps any valid NeXus data in an 
          NXentry group, in order to produce a standard-compliant file. See
          `Saving NeXus Data`_ for more details.

NeXus Objects
=============
NeXus data is stored as a hierarchical tree structure, much like a computer file 
system. NeXus data structures consist of groups, with base class NXgroup, which 
can contain fields, with base class NXfield, and/or other groups.

NeXus Fields
------------
NeXus data values are stored in NeXus objects of class 'NXfield'. The NXfield
class wraps standard Numpy arrays, scalars, and Python strings so that
additional metadata (or attributes) and methods can be associated with them. 

There are three ways to create an NXfield.

# Direct assignment::

    >>> x = NXfield(np.linspace(0,2*np.pi,101), units='degree')

  The data value is given by the first positional argument, and may be a Python
  scalar or string, or a Numpy array. In this method, keyword arguments can be
  used to define NXfield attributes.

# Attribute assignment as the child of a NeXus group::

    >>> a.entry.sample.temperature=40.0

  The assigned values are automatically converted to an NXfield::

    >>> a.entry.sample.temperature
    NXfield(40.0)

# Dictionary assignment to the NeXus group::

    >>> a.entry.sample["temperature"]=40.0

  This is equivalent to the second method, but should be used if there is a 
  danger of a name clash with an NXfield method, *e.g.*, if the NXfield is 
  called 'plot'.
  
.. note:: When using the NeXpy GUI shell (see :doc:`pythongui`), it is possible 
          to use tab completion to check for possible name clashes with NXfield 
          methods. To avoid name clashes in scripts, use dictionary assignments.

The data in an NXfield can be of type integer, float, or character. The type is
normally inherited automatically from the data type of the Python object, 
although it is possible to define alternative (but compatible) datatypes. For 
example, a float64 array can be converted to float32 on assignment::

  >>> x=np.linspace(0,2*np.pi,101)
  >>> x.dtype
  dtype('float64')
  >>> a=NXfield(x,dtype='float32')
  >>> a.dtype
  dtype('float32')
  >>> b=NXfield('Some Text')
  >>> b.dtype, b.shape
  (dtype('O'), ())

.. note:: Numeric dtypes can be defined either as a string, *e.g.*, 'int16', 
          'float32', or using the Numpy dtypes, *e.g.*, np.int16, np.float32.

.. warning:: By default, Python strings are stored as variable-length strings in
             the HDF5 file. These use a special object dtype defined by h5py 
             (see the `h5py documentation 
             <http://docs.h5py.org/en/latest/special.html#variable-length-strings>`_).
             As of v0.3.0, if you wish to store fixed length strings, specify 
             a dtype of kind 'S', *e.g.*, 'S10' when creating the NXfield. Any
             dtype of kind 'U' (for unicode string in Python 3), will be 
             automatically converted to the variable-length string dtype since
             HDF5 cannot convert them.

.. warning:: If you wish to store an array of strings containing Unicode
             characters as fixed-length strings, convert them to byte strings
             first using UTF-8 encoding, *e.g.* (in Python 2.7)::
             
               >>> t = [u'a', u'b', u'c', u'd', u'é']
               >>> a=NXfield(np.array(np.core.defchararray.encode(t, 'utf-8')), 
                             dtype='S10')
               >>> a
               NXfield(array(['a', 'b', 'c', 'd', '\xc3\xa9'], dtype='|S10'))

Similarly, the shape and dimension sizes of an integer or float array is 
inherited from the assigned Numpy array. It is possible to initialize an NXfield
array without specifying the data values in advance, *e.g.*, if the data has to
be created in slabs::

  >>> a=NXfield(dtype=np.float32, shape=[2048,2048,2048])
  >>> a
  NXfield(shape=(2048, 2048, 2048), dtype=float32)

More details of handling large arrays are given below.

NeXus attributes
^^^^^^^^^^^^^^^^  
The NeXus standard allows additional attributes to be attached to NXfields to
contain metadata ::

 >>> a.entry.sample.temperature.units='K'

These have a class of NXattr. They can be defined using the 'attrs' dictionary 
if necessary to avoid name clashes::

 >>> a.entry.sample.temperature.attrs['units']='K'

Other common attributes include the 'signal' and 'axes' attributes used to 
define the plottable signal and independent axes, respectively, in a NXdata 
group.

When a NeXus tree is printed, the attributes are prefixed by '@'::

 >>> print(a.entry.sample.tree)
 sample:NXsample
   temperature = 40.0
     @units = 'K' 

Masked Arrays
^^^^^^^^^^^^^
Numpy has the ability to store arrays with masks to remove missing or invalid
data from computations of, *e.g.*, averages or maxima. Since Matplotlib is able 
to handle masked arrays and removes masked data from plots, this is a convenient 
way of preventing bad data from contaminating statistical analyses, while 
preserving all the data values, good and bad, *i.e.*, masks can be turned on and 
off. 

NeXpy uses the same syntax as Numpy for masking and unmasking data.

 >>> z = NXfield([1,2,3,4,5,6], name='z')
 >>> z[3:5] = np.ma.masked
 >>> z
 NXfield([1 2 3 -- -- 6])
 >>> z.mask
 array([False, False, False,  True,  True, False], dtype=bool)
 >>> z.mask[3] = np.ma.nomask
 >>> z
 NXfield([1 2 3 4 -- 6])
 
.. warning:: If you perform any operations on a masked array, those operations 
             are not performed on the masked values. It is not advisable
             to remove a mask if you have modified the unmasked values. 

If the NXfield does not have a parent group, the mask is stored within the field
as in Numpy arrays. However, if the NXfield has a parent group, the mask is 
stored in a separate NXfield that is generated automatically by the mask
assignment or whenever the masked NXfield is assigned to a group. The mask is
identified by the 'mask' attribute of the masked NXfield.

 >>> print(NXlog(z).tree)
 log:NXlog
 z = [1 2 3 4 -- 6]
  @mask = 'z_mask'
 z_mask = [False False False False  True False]

The mask can then be saved to the NeXus file if required.

.. warning:: In principle, the NXfield containing the mask can be modified 
             manually, but it is recommended that modifications to the mask use
             the methods described above.
             
Masks can also be set using the Projection panel in the :doc:`pythongui`.

Large Arrays
^^^^^^^^^^^^
If the size of an array is too large to be loaded into memory (as defined by 
NX_MEMORY), the NXfield can be created without any initial values, and then
filled incrementally as slabs::

 >>> entry.data.z = NXfield(shape=(1000,1000,1000), dtype=np.float32)
 >>> for i in range(1000):
         entry.data.z[i,:,:] = np.ones(shape=(1000,1000), dtype=np.float32)
             ...

If ``entry`` in the above example is already stored in a NeXus file (with write
access), then ``entry.data.z`` is automatically updated in the file. If it is 
not stored in a file, the field is stored in an HDF5 core memory file that will 
be copied to the NeXus file when it is saved.

When initializing the NXfield, it is possible to specify a number of HDF5 
attributes that specify how the data are stored. 

* Compression::

    >>> z = NXfield(shape=(1000,1000,1000), dtype=np.float32, compression='lzf')

  This specifies the compression filter used. For large arrays, the data are
  compressed with the ``gzip`` filter by default. 

* Chunk size::

    >>> z = NXfield(shape=(1000,1000,1000), dtype=np.float32, chunks=(1,100,100))

  If chunk sizes are not specified, HDF5 will choose default sizes.

* Maximum array shape::

    >>> z = NXfield(shape=(10,1000,1000), dtype=np.float32, maxshape=(1000,1000,1000))

  The initial shape is defined by the ``shape`` attribute, but it will be 
  automatically expanded up to a limit of ``maxshape`` if necessary.

* Fill value::

    >>> z = NXfield(shape=(1000,1000,1000), dtype=np.float32, fillvalue=np.nan)

  Slabs that are not initialized will contain the specified fill value. This is
  normally set to zero by default.
  
All these values can be adjusted at the command line until the first slab has
been written, whether to a file or in core memory, using the ``compression``, 
``chunks``, ``maxshape`` or ``fillvalue`` properties, *e.g.*

 >>> z = NXfield(shape=(1000,1000,1000), dtype=np.float32)
 >>> z.compression = 'lzf'

.. warning :: Setting the HDF5 attributes on an NXfield that is not stored
              in a NeXus file requires v0.3.0.

NeXus Groups
------------
NeXus groups are defined as subclasses of the NXgroup class, with the class name 
defining the type of information they contain, *e.g.*, the NXsample class 
contains metadata that define the measurement sample, such as its temperature or 
lattice parameters. The initialization parameters can be used to populate the 
group with other predefined NeXus objects, either groups or fields::

 >>> temperature = NXfield(40.0, units='K')
 >>> sample = NXsample(temperature=temperature)
 >>> print(sample.tree)
 sample:NXsample
   temperature = 40.0
     @units = 'K'

In this example, it was necessary to use the keyword form to add the NXfield 
'temperature' since its name is otherwise undefined within the NXsample group. 
However, the name is set automatically if the NXfield is added as an attribute 
or dictionary assignment::

 >>> sample = NXsample()
 >>> sample.temperature=NXfield(40.0, units='K')
 sample:NXsample
   temperature = 40.0
     @units = 'K'

The NeXus objects in a group (NXfields or NXgroups) can be accessed as  
dictionary items::

 >>> sample["temperature"] = 40.0
 >>> sample.keys()
 ['temperature']
 
.. note:: It is also possible to reference objects by their complete paths with
          respect to the root object, *e.g.*, root['/entry/sample/temperature'].

If a group is not created as another group attribute, its internal name defaults
to the class name without the 'NX' prefix. This can be useful in automatically
creating nested groups with minimal typing::

 >>> a=NXentry(NXsample(temperature=40.0),NXinstrument(NXdetector(distance=10.8)))
 >>> print(a.tree)
 entry:NXentry
   instrument:NXinstrument
     detector:NXdetector
       distance = 10.8
   sample:NXsample
     temperature = 40.0

.. seealso:: Existing NeXus objects can also be inserted directly into groups.
             See :mod:`nexusformat.nexus.tree.NXgroup.insert`

NXdata Groups
^^^^^^^^^^^^^
NXdata groups contain data ready to be plotted. That means that the group should
consist of an NXfield containing the data and one or more NXfields containing
the axes. NeXus defines a method of associating axes with the appropriate
dimension, but NeXpy provides a simple constructor that implements this method
automatically. This was already demonstrated in the example above, reproduced
here::

 >>> import numpy as np
 >>> x=y=np.linspace(0,2*np.pi,101)
 >>> X,Y=np.meshgrid(x,y)
 >>> z=np.sin(X)*np.sin(Y)
 >>> a=NXdata(z,[y,x])

The first positional argument is an NXfield or Numpy array containing the data,
while the second is a list containing the axes, again as NXfields or Numpy
arrays. In this example, the names of the arrays have not been defined within an
NXfield so default names were assigned::

 >>> print(a.tree)
 data:NXdata
   @axes = ['axis1' 'axis2']
   @signal = signal
   axis1 = float64(101)
   axis2 = float64(101)
   signal = float64(101x101)

.. note:: The plottable signal and axes are identified by the 'signal'
          and 'axes' attributes of the NXdata group. The 'axes' attribute 
          defines the axes as a list of NXfield names The NXdata constructor 
          sets these attributes automatically.

.. warning:: Numpy stores arrays by default in C, or row-major, order, *i.e.*, 
             in the array 'signal(axis1,axis2)', axis2 is the fastest to vary. 
             In most image formats, *e.g.*, TIFF files, the x-axis is assumed
             to be the fastest varying axis, so we are adopting the same
             convention and plotting as 'signal(y,x)'. The :doc:`pythongui` 
             allows the x and y axes to be swapped.

Names can be assigned explicitly when creating the NXfield through the 'name' 
attribute::

 >>> phi=NXfield(np.linspace(0,2*np.pi,101), name='polar_angle')
 >>> data=NXfield(np.sin(phi), name='intensity')
 >>> a=NXdata(data,(phi))
 >>> print(a.tree)
 data:NXdata
   @axes = 'polar_angle'
   @signal = 'intensity'
   intensity = float64(101)
   polar_angle = float64(101)

.. note:: In the above example, the x-axis, 'phi', was defined as a tuple in the
          second positional argument of the NXdata call. It could also have been
          defined as a list. However, in the case of one-dimensional signals, it
          would also have been acceptable just to call NXdata(data, phi), 
          *i.e.*, without embedding the axis in a tuple or list. 

It is also possible to define the plottable signal and/or axes using the 
'nxsignal' and 'nxaxes' properties, respectively::

 >>> phi=np.linspace(0,2*np.pi,101)
 >>> a=NXdata()
 >>> a.nxsignal=NXfield(np.sin(phi), name='intensity')
 >>> a.nxaxes=NXfield(phi, name='polar_angle')
 >>> print(a.tree)
 data:NXdata
   @axes = 'polar_angle'
   @signal = 'intensity'
   intensity = float64(101)
   polar_angle = float64(101)


NeXus Links
-----------
NeXus allows groups and fields to be assigned to multiple locations through the
use of links. These objects have the class NXlink and contain the attribute 
'target', which identifies the parent object. It is also possible to link to
fields in another NeXus file (see 'External Links' below).

For example, the polar angle and time-of-flight arrays may logically be stored 
with the detector information in a NXdetector group that is one of the 
NXinstrument subgroups::

 >>> print(entry.instrument.tree)
 instrument:NXinstrument
   detector:NXdetector
    distance = float32(128)
      @units = 'metre'
    polar_angle = float32(128)
      @units = 'radian'
    time_of_flight = float32(8252)
      @target = '/entry/instrument/detector/time_of_flight'
      @units = 'microsecond'

However, they may also be needed as plotting axes in a NXdata group::

 >>> print(entry.data.tree)
 data:NXdata
   @axes = ['polar_angle' 'time_of_flight']
   @signal = data
   data = uint32(128x8251)
   polar_angle = float32(128)
     @target = '/entry/instrument/detector/polar_angle'
     @units = 'radian'
   time_of_flight = float32(8252)
     @target = '/entry/instrument/detector/time_of_flight'
     @units = 'microsecond'
 
Links allow the same data to be used in different contexts without using more
memory or disk space.

.. note:: In earlier verions, links were required to have the same name as their
          parents, but this restriction has now been lifted. 
        
In the Python API, the user who is only interested in accessing the data does
not need to worry if the object is parent or child. The data values and NeXus 
attributes of the parent to the NXlink object can be accessed directly through
the child object. The parent object can be referenced directly, if required,
using the 'nxlink' attribute::

 >>> entry.data.time_of_flight
 NXlink('/entry/instrument/detector/time_of_flight')
 >>> entry.data.time_of_flight.nxdata
 array([   500.,    502.,    504., ...,  16998.,  17000.,  17002.], dtype=float32) 
 >>> entry.data.time_of_flight.units
 'microsecond'
 >>> entry.data.time_of_flight.nxlink
 NXfield(dtype=float32,shape=(8252,))

.. note:: The absolute path of the data with respect to the root object of the 
          NeXus tree is given by the nxpath property::

           >>> entry.data.time_of_flight.nxpath
           '/entry/data/time_of_flight'
           >>> entry.data.time_of_flight.nxlink.nxpath
           '/entry/instrument/bank1/time_of_flight'

Creating a Link
^^^^^^^^^^^^^^^
Links can be created using the target object as the argument assigned
to another group::

 >>> print(root.tree)
 root:NXroot
   entry:NXentry
     data:NXdata
     instrument:NXinstrument
       detector:NXdetector
         polar_angle = float64(192)
           @units = 'radian'
 >>> root.entry.data.polar_angle=NXlink(root.entry.instrument.detector.polar_angle)

It is also possible to create links using the makelink method, which takes the 
parent object and, optionally, a new name as arguments::

 >>> root.entry.data.makelink(root.entry.instrument.detector.polar_angle)
 >>> print(root.tree)
 root:NXroot
   entry:NXentry
     data:NXdata
       polar_angle = float64(192)
         @target = '/entry/instrument/detector/polar_angle'
         @units = 'radian'
     instrument:NXinstrument
       detector:NXdetector
         polar_angle = float64(192)
           @target = '/entry/instrument/detector/polar_angle'
           @units = 'radian'

.. note:: After creating the link, both the parent and target objects have an 
          additional attribute, 'target', showing the absolute path of the 
          parent.

.. seealso:: :mod:`nexusformat.nexus.tree.NXgroup.makelink`

External Links
^^^^^^^^^^^^^^
It is also possible to link to a NeXus field that is stored in another file.
This is accomplished using a similar syntax to internal links.

 >>> root.entry.data.data = NXlink('/counts', file='external_counts.nxs')
 
In the case of external links, the first argument is the absolute path of the 
linked object within the external file, while the second argument is the 
absolute or relative file path of the external file.

By default, the target file path is converted to a relative path with respect
to the parent file. If it is required to store the absolute file path, add the
keyword argument, ``abspath=True``.

 >>> root.entry.data.data = NXlink('/counts', 
                                   file='/home/user/external_counts.nxs',
                                   abspath=True)
 
.. warning:: If the files are moved without preserving their relative file 
             paths, the parent file will still open but the link will be broken.

Plotting NeXus Data
===================
NXdata, NXmonitor, and NXlog groups all have a plot method, which automatically 
determines what should be plotted::

 >>> data.plot()

.. image:: /images/simple-plot.png
   :align: center
   :width: 80%

Note that the plot method uses the NeXus attributes within the groups to
determine automatically which NXfield is the signal, what its rank and
dimensions are, and which NXfields define the plottable axes. The same command
will work for one-dimensional or two-dimensional data. If you plot
higher-dimensional data, the top two-dimensional slice is plotted. Alternative
two-dimensional slices can be specified using slice indices on the NXdata group.

If the data is one-dimensional, it is possible to overplot more than one data
set using 'over=True'. By default, each plot has a new color, but conventional
Matplotlib keywords can be used to change markers and colors::

 >>> data.plot(log=True)
 >>> data.plot('r-')
 >>> data.plot(over=True, log=True, color='r')

If the NXdata group contains RGB(A) image data, *i.e.*, the signal is a 
three-dimensional array, in which the fastest varying dimension, which should be 
of size 3 or 4, contains the RGB(A) values for each two-dimensional pixel, then
the image can be plotted using the 'image=True'.

 >>> data.plot(image=True)

By convention, the first pixel of an image is in the upper-left corner, rather 
than the lower-left used in other two-dimensional plots.

..note:: The plot method also works on NXroot and NXentry groups, if they are 
         able to identify plottable data. If the ``default`` attribute is set, 
         the default NXentry and/or NXdata groups are used. Otherwise, the first
         valid NXdata group found in an iterative search is used.
 
Additional Plot Methods
-----------------------
As a convenience, additional plot methods can be used instead of adding extra
keywords.

 >>> data.oplot()
 >>> data.logplot()
 >>> data.implot()
 
These are equivalent to setting the 'over', 'log', and 'image' keywords to True
when invoking the plot method.

Manipulating NeXus Data
=======================
Arithmetic Operations
---------------------
NXfield
^^^^^^^
NXfields usually consist of arrays of numeric data with associated metadata, the 
NeXus attributes (the exception is when they contain character strings). This 
makes them similar to Numpy arrays, and this module allows the use of NXfields 
in numerical operations as if they were Numpy ndarrays::

 >>> x = NXfield((1.0,2.0,3.0,4.0))
 >>> print(x+1)
 [ 2.  3.  4.  5.]
 >>> print(2*x)
 [ 2.  4.  6.  8.]
 >>> print(x/2)
 [ 0.5  1.   1.5  2. ]
 >>> print(x**2)
 [  1.   4.   9.  16.]
 >>> x.reshape((2,2))
 NXfield([[ 1.  2.]
 [ 3.  4.]])
 >>> y = NXfield((0.5,1.5,2.5,3.5))
 >>> x+y
 NXfield(name=x,value=[ 1.5  3.5  5.5  7.5])
 >>> x*y
 NXfield(name=x,value=[  0.5   3.    7.5  14. ])
 >>> (x+y).shape
 (4,)
 >>> (x+y).dtype
 dtype('float64')

Such operations return valid NXfield objects containing the same attributes 
as the first NXobject in the expression. The 'reshape' and 'transpose' methods 
also return NXfield objects.

NXfields can be compared to other NXfields (this is a comparison of their Numpy 
arrays)::

 >>> y=NXfield(np.array((1.5,2.5,3.5)),name='y')
 >>> x == y
 True

NXfields are technically not a sub-class of the ndarray class, but they are cast
as Numpy arrays when required by Numpy operations, returning either another 
NXfield or, in some cases, an ndarray that can easily be converted to an 
NXfield::

 >>> x = NXfield((1.0,2.0,3.0,4.0)) 
 >>> x.size
 4
 >>> x.sum()
 10.0
 >>> x.max()
 4.0
 >>> x.mean()
 2.5
 >>> x.var()
 1.25
 >>> x.reshape((2,2)).sum(1)
 array([ 3.,  7.])
 >>> np.sin(x)
 array([ 0.84147098,  0.90929743,  0.14112001, -0.7568025 ])
 >>> np.sqrt(x)
 array([ 1.        ,  1.41421356,  1.73205081,  2.        ])
 >>> print(NXdata(np.sin(x), (x)).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'signal'
   signal = [ 0.84147098  0.90929743  0.14112001 -0.7568025 ]
   x = [ 1.  2.  3.  4.]

NXdata
^^^^^^
Similar operations can also be performed on whole NXdata groups. If two NXdata
groups are to be added, the rank and dimension sizes of the main signal array
must match (although the names could be different)::

 >>> y=NXfield(np.sin(x),name='y')
 >>> y
 NXfield(name=y,value=[ 0.99749499  0.59847214 -0.35078323])
 >>> a=NXdata(y,x)
 >>> print(a.tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   x = [ 1.5  2.5  3.5]
   y = [ 0.99749499  0.59847214 -0.35078323]
 >>> print((a+1).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   x = [ 1.5  2.5  3.5]
   y = [ 1.99749499  1.59847214  0.64921677]
 >>> print((2*a).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   x = [ 1.5  2.5  3.5]
   y = [ 1.99498997  1.19694429 -0.70156646]
 >>> print((a+a).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   x = [ 1.5  2.5  3.5]
   y = [ 1.99498997  1.19694429 -0.70156646]
 >>> print((a-a).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   x = [ 1.5  2.5  3.5]
   y = [ 0.  0.  0.]
 >>> print((a/2).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   x = [ 1.5  2.5  3.5]
   y = [ 0.49874749  0.29923607 -0.17539161]

If data errors are included in the NXdata group (with an additional array named 
'errors'), then the errors are propagated according to the operand::

 >>> print(a.tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   errors = [ 0.99874671  0.77360981  0.59226956]
   x = [ 1.5  2.5  3.5]
   y = [ 0.99749499  0.59847214  0.35078323]
 >>> print((a+a).tree)
 data:NXdata
   @axes = 'x'
   @signal = 'y'
   errors = [ 1.41244114  1.09404949  0.83759564]
   x = [ 1.5  2.5  3.5]
   y = [ 1.99498997  1.19694429  0.70156646]

Some statistical operations can be performed on the NXdata group.

NXdata.sum(axis=None):
    Returns the sum of the NXdata signal data. If the axis is not specifed, the
    total is returned. Otherwise, it is summed along the specified axis. The 
    result is a new NXdata group containing a copy of all the metadata contained 
    in the original NXdata group::

     >>> x=np.linspace(0, 3., 4)
     >>> y=np.linspace(0, 2., 3)
     >>> X,Y=np.meshgrid(x,y)
     >>> a=NXdata(X*Y,(y,x))
     >>> print(a.tree)
     data:NXdata
       @axes = ['axis1' 'axis2']
       @signal = 'signal'
       axis1 = [ 0.  1.  2.  3.]
       axis2 = [ 0.  1.  2.]
       signal = float64(3x4)
     >>> a.nxsignal
     NXfield([[ 0.  0.  0.  0.]
      [ 0.  1.  2.  3.]
      [ 0.  2.  4.  6.]])
     >>> a.sum()
     18.0
     >>> a.sum(0).nxsignal
     NXfield([ 0.  3.  6.  9.])
     >>> a.sum(1).nxsignal
     NXfield([  0.   6.  12.])   

NXdata.average(axis=None):
    Returns the average of the NXdata signal data. This is identical to the sum
    method, but the result is divided by the number of data elements in the 
    summation::

     >>> a.average()
     1.5
     >>> a.average(0).nxsignal
     NXfield([ 0.,  1.,  2.,  3.])
     >>> a.average(1).nxsignal
     NXfield([ 0. ,  1.5,  3. ])   

NXdata.moment(order=1):
    Returns an NXfield containing the first moment of the NXdata group assuming 
    the signal is one-dimensional. Currently, only the first moment has been 
    defined::
    
     >>> x=np.linspace(0, 10., 11)
     >>> y=np.exp(-(x-3)**2)
     >>> a=NXdata(y,x)
     >>> a.moment()
     3.0000002539776141


Slicing
-------
NXfield
^^^^^^^
A slice of an NXfield can be obtained using the usual Python indexing syntax::

 >>> x=NXfield(np.linspace(0,2*np.pi,101))
 >>> print(x[0:51])
 [ 0.          0.06283185  0.12566371 ...,  3.01592895  3.0787608 3.14159265]

If either of the indices are floats, then the limits are set by the values 
themselves (assuming the array is monotonic)::

 >>> print(x[0.5:1.5])
 [ 0.50265482  0.56548668  0.62831853 ...,  1.38230077  1.44513262 1.50796447]

NXdata
^^^^^^
It is also possible to slice whole NXdata groups. In this case, the slicing
works on the multidimensional NXfield, but the full NXdata group is returned
with both the signal data and the associated axes limited by the slice
parameters. If either of the limits along any one axis is a float, the limits
are set by the values of the axis::

 >>> a=NXdata(np.sin(x),x)
 >>> a[1.5:2.5].x
 NXfield(name=x,value=[ 1.57079633  1.72787596  1.88495559 ...,  2.19911486  2.35619449])

Unless the slice reduces one of the axes to a single item, the rank of the data
remains the same. To project data along one of the axes, and so reduce the rank
by one, the data can be summed along that axis using the sum() method::

 >>> x=y=np.linspace(0,2*np.pi,41)
 >>> X,Y=np.meshgrid(x,y)
 >>> a=NXdata(np.sin(X)*np.sin(Y), (y,x))
 >>> print(a.tree)
 data:NXdata
   @axes = ['axis1' 'axis2']
   @signal = 'signal'
   axis1 = float64(41)
   axis2 = float64(41)
   signal = float64(41x41)
 >>> print(a.sum(0).tree)
 data:NXdata
   @axes = ['axis2']
   @signal = 'signal'
   @summed_bins = 41
   axis1 = 3.141592653589793
     @maximum = 6.283185307179586
     @minimum = 0.0
     @summed_bins = 41
   axis2 = float64(41)
   signal = float64(41)
   title = 'data'

It is also possible to slice whole NXdata groups. In this case, the slicing
works on the multidimensional NXfield, but the full NXdata group is returned
with both the signal data and the associated axes limited by the slice
parameters. If either of the limits along any one axis is a float, the limits
are set by the values of the axis::

 >>> a=NXdata(np.sin(x),x)
 >>> a[1.5:2.5].x
 NXfield(name=x,value=[ 1.57079633  1.72787596  1.88495559 ...,  2.19911486  2.35619449])

Unless the slice reduces one of the axes to a single item, the rank of the data
remains the same. To project data along one of the axes, and so reduce the rank
by one, the data can be summed along that axis using the sum() method. This
employs the Numpy array sum() method::

 >>> x=y=NXfield(np.linspace(0,2*np.pi,41))
 >>> X,Y=np.meshgrid(x,y)
 >>> a=NXdata(np.sin(X)*np.sin(Y), (y,x))
 >>> print(a.tree)
 data:NXdata
   @axes = ['axis1' 'axis2']
   @signal = 'signal'
   axis1 = float64(41)
   axis2 = float64(41)
   signal = float64(41x41)
 >>> print(a.sum(0).tree)
 data:NXdata
   @axes = 'axis2'
   @signal = 'signal'
   axis2 = float64(41)
   signal = float64(41)
     @long_name = 'Integral from 0.0 to 6.28318530718 '

NXdata.project(axes, limits):
    The project() method projects the data along a specified 1D axis or 2D axes 
    summing over the limits, which are specified as a list of tuples for each 
    dimension. If the axis is not to be limited, then specify the limit as 
    *None*. The data should be at least two-dimensional and the values are 
    assumed to be floating point. 

    >>> x=np.linspace(0, 3., 4)
    >>> y=np.linspace(0, 2., 3)
    >>> X,Y=np.meshgrid(x,y)
    >>> a=NXdata(X*Y,(y,x))
    >>> print(a.tree)
    data:NXdata
      @axes = ['axis1' 'axis2']
      @signal = 'signal'
      axis1 = [ 0.  1.  2.]
      axis2 = [ 0.  1.  2.  3.]
      signal = float64(3x4)
    >>> print(a.signal)
    [[ 0.  0.  0.  0.]
     [ 0.  1.  2.  3.]
     [ 0.  2.  4.  6.]]
    >>> print(a.project([0],[(None,None),(0.5,2.5)]).tree)
    data:NXdata
      @axes = 'axis1'
      @signal = 'signal'
      axis1 = [ 0.  1.  2.]
      axis2 = 1.5
        @maximum = 2.0
        @minimum = 1.0
      signal = [ 0.  3.  6.]

    The :doc:`pythongui` provides a menu-based approach to simplify the plotting 
    of data projections.

Saving NeXus Data
=================
Every NeXus object, whether it is a group or a field, has a save() method as 
illustrated in `Creating NeXus Data`_.::

 >>> root.save(filename='example.nxs')

NXroot Groups
-------------
If the NeXus object is a NXroot group, the save() method saves the whole NeXus 
tree. The filename can only be omitted if the tree is being saved to a file that 
was loaded with read/write access. In this case, the format argument is ignored.
If the tree was loaded with readonly access, any modifications must be saved to
a new file specified by the filename argument.

Other Objects
-------------
If the object is not a NXroot group, a new file will be created containing the
selected object and its children. A filename *must* be specified. Saving 
non-NXroot data allows parts of a NeXus tree to be saved for later use, *e.g.*, 
to store an NXsample group that will be added to other files. The saved NeXus 
object is wrapped in an NXroot group and an NXentry group (with name 'entry'), 
if necessary, in order to produce a valid NeXus file.
     