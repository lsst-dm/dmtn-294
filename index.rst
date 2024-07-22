##############################################
File Formats and Layouts for Cell-based Coadds
##############################################

.. abstract::

   Rubin's deep coadds will be built on a grid of small cells, in which each cell has an approximately constant PSF.
   Cells will have "inner regions" that can be stitched together to form the full coadd, but they will also have outer regions that overlap (neighboring cells will have their own versions of some of the same pixels), in order to allow convolutions and other operations that require padding to be performed rigorously cell by cell.
   This creates a problem for how to store a coadd in an on-disk FITS file: we want a layout that can be easily interpreted by third-party readers, but we also need to support compression and efficient subimage reads of at least the inner cell region.
   This technical note will summarize various possibilities and their advantages and disadvantages.

Goals and Requirements
======================

Rubin's cell-based coadds will need to store five or more image planes that share a single coordinate system and pixel grid:

- a floating point main data image;
- an integer bitmask;
- a floating point variance plane, with per-pixel variance estimates that include photon noise from sources;
- a floating point "interpolation fraction" image that records fractional missing data in each pixel;
- at least one floating point Monte Carlo noise realization.

It will need to store at least one PSF model image for each cell; these will be smaller than the other per-cell images but the grid will have the same shape.
To account for chromatic PSF effects we may also store images that correspond to the derivatives of the PSF with respect to some proxy for object SED, in at least some bands.

In addition, we'll need to store considerable structured metadata, including a WCS, information about the grid structure, coadded aperture corrections and wavelength-dependent throughput, the visit and detector IDs of the epochs that contributed to each cell, and additional information about the pixel uncertainty (some measure of typical covariance, and an approximately constant variance that either averages or does not include photon noise from sources).

We will assume throughout this note that we're writing FITS files: regardless of any other format Rubin might support, we have to support FITS as well, so it's just more work to do anything else.

These files will be one per "patch", which we assume to be an approximately 4k x 4k image, divided into cells with an inner region that is approximately 150x150 and 50 pixel padding on all sides.
We will only consider layouts that save the entire coadd, including the outer cell regions.

We need these files to be readable over both POSIX filesystems and S3 and webdav object stores, and we need to be able to read subimages efficiently over all of these storage systems (i.e. we cannot afford to read the entire file just to read a subimage).
Writing to just POSIX filesystems is adequate, as there's no problem with writing to local temporary storage and then uploading separately in our pipeline architecture.
We do not expect to need the ability to extend existing files.

We expect to compress all image planes with at least lossless compression, and would like to have the capability to perform lossy compression on some planes as well.
If we do lossy compression, we would quite likely want to do our own quantization (our `afw` library already has code for this that we prefer over CFITSIO's).
DESC has expressed an interest in applying lossy compression to any coadd files they transfer to NERSC, if we have not done so already.

Constraints from FITS
=====================

Because we need to be able to read subimages efficiently, file-level compression is not an option: our only options are FITS image "tile compression" and perhaps lesser-known FITS 4.0's binary table compression.
Combined with our need to subimage reads over object stores, this already puts us beyond what third-party FITS libraries can do:

- CFITSIO can do image tile compression, including efficient subimage reads that take advantage of the tile structure, but only on POSIX filesystems.
  It can also do subset reads of FITS binary tables, again only on POSIX filesystems.
  This imposes the same limitation on its Python bindings in the ``fitsio`` package.

- Astropy can do image tile compression with efficient subimage reads on POSIX filesystems, and it can do uncompressed reads (including efficient subimage reads) against both POSIX and object stores via `fsspec`, but it delegates to vendored CFITSIO code for its tile compression code and does not support the combination of these two.
  Astropy cannot do subset reads of FITS binary tables, even on POSIX filesystems.

- I am not aware of any library that can do FITS binary table compression at all.

As a result, I expect us to need to write some low-level code of our own to do subimage reads over object stores, though this is not specific to cell-based coadds: we need this for all of our image data products.
Whether to contribute more general code upstream (probably to Astropy) or focus only on reading the specific files we write ourselves is an important open question; the general solution would be very useful the community, but its scope is unquestionably larger.

Finally, the WCS of any single cell (or the inner-cell stitched image) will be simple and can be exactly represented in FITS headers (unlike the WCSs of our single-epoch images).
An additional FITS WCS could be used to describe the position of a single-cell image within a larger grid.
It is highly desirable that any images we store in our FITS files have both to allow third-party FITS readers to fully interpret them.

Image Data Layouts
==================

This section will cover potential ways to lay out the image data - the 5+ planes that share a common grid, as well as the PSF images - across multiple FITS HDUs.

Binary Table With Per-Cell Rows
-------------------------------

TODO

Per-HDU Cells
-------------

TODO

Data Cubes
----------

TODO

Exploded Images
---------------

TODO

Stitched Images
---------------

TODO

Hybrid Options
--------------

TODO


Metadata Layouts
================

Cell coadd metadata falls into three categories:

- global information common to all cells (identifiers, WCS, grid structure);
- per-cell information with a fixed schema (including coadded aperture corrections and wavelength-dependent throughput);
- visit, detector and other IDs for the observations that contribute to each cell.

While some global information will go into FITS headers (certainly the WCS and some identifiers), we do not want to assume that all global metadata can be neatly represented in the limited confines of a FITS header.
A single-row binary table is another option, but we will likely instead adopt the approach recently proposed for other Rubin image data products on RFC-1030: embedding a JSON document as a byte array in a FITS extension HDU.

A binary table with per-cell rows is a natural fit for the fixed-schema per-cell information, especially if the image data layout already involves a binary table with per-cell rows.
But if we're embedding a JSON document in the FITS file anyway, it might make more sense to store this information in JSON as well; this will let us share code, documentation, and serialization for more complex objects with other Rubin image data products, and that includes sharing the machinery for managing schema changes schema documentation.

The tables of observations that contribute to each cell is also a natural binary table, but not one with per-cell rows (it's more natural as a cell-visit-detector join table), but once again embedded JSON is an equally viable option.


See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.
