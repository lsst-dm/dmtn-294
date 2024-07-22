##############################################
File Formats and Layouts for Cell-based Coadds
##############################################

.. abstract::

   Rubin's deep coadds will be built in on a grid of small cells, in which each cell has an approximately constant PSF.  Cells will have "inner regions" that can be stitched together to form the full coadd, but they will also have outer regions that overlap (neighboring cells will have their own versions of some of the same pixels), in order to allow convolutions and other operations that require padding to be performed rigorously cell by cell.  This creates a problem for how to store a coadd in an on-disk FITS file: we want a layout that can be easily interpreted by third-party readers, but we also need to support compression and efficient subimage reads of at least the inner cell region.  This technical note will summarize various possibilities and their advantages and disadvantages.

Add content here
================

See the `Documenteer documentation <https://documenteer.lsst.io/technotes/index.html>`_ for tips on how to write and configure your new technote.
