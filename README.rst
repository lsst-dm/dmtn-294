.. image:: https://img.shields.io/badge/dmtn--294-lsst.io-brightgreen.svg
   :target: https://dmtn-294.lsst.io
.. image:: https://github.com/lsst-dm/dmtn-294/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-294/actions/

##############################################
File Formats and Layouts for Cell-based Coadds
##############################################

DMTN-294
========

Rubin's deep coadds will be built in on a grid of small cells, in which each cell has an approximately constant PSF.  Cells will have "inner regions" that can be stitched together to form the full coadd, but they will also have outer regions that overlap (neighboring cells will have their own versions of some of the same pixels), in order to allow convolutions and other operations that require padding to be performed rigorously cell by cell.  This creates a problem for how to store a coadd in an on-disk FITS file: we want a layout that can be easily interpreted by third-party readers, but we also need to support compression and efficient subimage reads of at least the inner cell region.  This technical note will summarize various possibilities and their advantages and disadvantages.

**Links:**

- Publication URL: https://dmtn-294.lsst.io
- Alternative editions: https://dmtn-294.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-294
- Build system: https://github.com/lsst-dm/dmtn-294/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-294
   cd dmtn-294
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-294.lsst.io whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-294.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
