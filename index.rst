##############################################
File Formats and Layouts for Cell-based Coadds
##############################################

.. abstract::

   Rubin's deep coadds will be built on a grid of small cells, in which each cell has an approximately constant PSF.
   Cells will have "inner regions" that can be stitched together to form the full (patch-sized) coadd, but they will also have outer regions that overlap (neighboring cells will have their own versions of some of the same pixels), in order to allow convolutions and other operations that require padding to be performed rigorously cell by cell.
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

We will also need to store at least one PSF model image for each cell; these will be smaller than the other per-cell images but the grid will have the same shape.
To account for chromatic PSF effects we may also store images that correspond to the derivatives of the PSF with respect to some proxy for object SED, in at least some bands.

In addition, we'll need to store considerable structured metadata, including a WCS, information about the grid structure, coadded aperture corrections and wavelength-dependent throughput, the visit and detector IDs of the epochs that contributed to each cell, and additional information about the pixel uncertainty (some measure of typical covariance, and an approximately constant variance that either averages or does not include photon noise from sources).

It is highly desirable that non-Rubin-specific third-party image viewers be able to understand the WCS, display coordinates for pixels indicated by users, and accurately overlay marks on the image at the locations of detected objects or other targets.

We will assume throughout this note that we're writing FITS files: regardless of any other format Rubin might support, we have to support FITS as well, so it's just more work to do anything else.

These files will be one per "patch", which we assume to be an approximately 4k × 4k image, divided into cells with an inner region that is approximately 150×150 with 50 pixel padding on all sides.
We will only consider layouts that save the entire coadd, including the outer cell regions.

We need these files to be readable over both POSIX filesystems and S3 and webdav object stores, and we need to be able to read subimages efficiently over all of these storage systems (i.e. we cannot afford to read the entire file just to read a subimage).
Writing to just POSIX filesystems is adequate, as there's no problem with writing to local temporary storage and then uploading separately in our pipeline architecture.
We do not expect to need the ability to extend existing files.

We expect to compress all image planes with at least lossless compression, and need like to have the capability to perform lossy compression on some planes as well.
For lossy compression, we would quite likely want to do our own quantization (our `afw` library already has code for this that we prefer over CFITSIO's),
and would want to use subtractive dithering (adding and subtracting deterministic pseudorandom numbers to avoid quantization biases).
DESC has expressed an interest in applying lossy compression to any coadd files they transfer to NERSC, if we have not done so already.

Constraints from FITS
=====================

Because we need to be able to read subimages efficiently, file-level compression is not an option: our only options are FITS image "tile compression" and perhaps lesser-known FITS 4.0's binary table compression.
Combined with our need to subimage reads over object stores, this already puts us beyond what third-party FITS libraries can do:

- CFITSIO can do image tile compression and binary table compression, including efficient subimage reads that take advantage of the tile structure, but only on POSIX filesystems (or blocks of contiguous memory that correspond to the on-disk layout).
  It can also do subset reads of FITS binary tables, again only on POSIX filesystems.
  CFITSIO can only permits customization of the quantization used in image tile compression via undocumented internal interfaces (which is what we use in ``afw``).
  This imposes the same limitation on its Python bindings in the ``fitsio`` package.

- Astropy can do image tile compression with efficient subimage reads on POSIX filesystems, and it can do uncompressed reads (including efficient subimage reads) against both POSIX and object stores via `fsspec`, but it delegates to vendored CFITSIO code for its tile compression code and does not support the combination of these two.
  Astropy cannot do subset reads of FITS binary tables, even on POSIX filesystems.
  Astropy also does not provide a way to customize the quantization during lossy compression.

As a result, I expect us to need to write some low-level code of our own to do subimage reads over object stores, though this is not specific to cell-based coadds: we need this for all of our image data products.
We also may need new low-level code to write tile-compressed images with custom quantization, since the code we have in ``afw`` for this is not general enough to work with all of the file layouts described here.
Whether to contribute more general code upstream (probably to Astropy) or focus only on reading the specific files we write ourselves is an important open question; the general solution would be very useful the community, but its scope is unquestionably larger.

Finally, the WCS of any single cell (or the inner-cell stitched image; they differ only by an integer offset) will be simple and can be exactly represented in FITS headers (unlike the WCSs of our single-epoch images).
An additional FITS WCS could be used to describe the position of a single-cell image within a larger grid (this is just an integer offset).
It is highly desirable that any images we store in our FITS files have both to allow third-party FITS readers to fully interpret them.

Image Data Layouts
==================

This section will cover potential ways to lay out the image data - the 5+ planes that share a common grid, as well as the PSF images - across multiple FITS HDUs.

Binary Table With Per-Cell Rows
-------------------------------

Since FITS binary tables can have array columns with arbitrary shapes, the most intuitive layout for cell-based image data is probably a single binary table HDU that has each image plane as a separate 2-d array column.
A FITS WCS can be stored with each row, using "Green Bank" convention to translate header keys to table columns; this may be sufficient for third-party readers to fully interpret our files and even stitch together cells on the fly (in the case of image viewers).

It's not clear whether any extant FITS viewers could *already* do this in an intuitive way, but by relying on an existing convention it'd be reasonable for them to add it, even if they were otherwise disinclined to add support for observatory-specific file formats.

The Green Bank convention makes no mention of having multiple images as different columns in the same table, however, so we may want to consider putting different image planes in different binary table HDUs, each with corresponding rows, and probably some columns - like the WCS ones - duplicated.
The extra storage cost of another header and some duplicated columns is insignificant, and it may be desirable to put multiple cells from the same plane closer together on disk than different planes of the same cell, especially since we expect that subimage reads of some planes (i.e. the image data image) will be much more popular than others (e.g. noise realizations).

PSF images fit naturally in this layout, as there's nothing wrong with having some array columns having a different shape.
There might be some confusion with a Green Bank convention WCS, though, if we go with a single table with different columns for different image planes as well as the PSF; there's no way to mark which image columns the WCS applies to.

The main problem with binary table layouts is compression support.
The latest FITS standard does include (in section 10.3) a discussion of "tiled table compression", which is limited to lossless compression algorithms, though the standard endorses the idea of extending this in the future.
It might still be possible to do our own non-dithered quantization and use ``BZERO`` and ``BSCALE`` to record the transformation from integer back to floating-point, and external FITS readers that understand both the Green Bank convention and tiled table compression could reasonably be expected to interpret this as intended.
But this does not provide a way to do subtractive dithering (which we consider necessary for lossy compression) or a way to quantize more than one floating-point image-like column in a single table, unless they had the same quantization parameters (and our images and variances, at least, would not).

Adopting a binary table layout would thus effectively mean any lossy-compressed columns we write would be unavailable to third-party FITS readers until and unless we could standardize some approach to quantization and get it implemented in other libraries.
This seems doable given the positive language in the standard about that possibility, but not on a short timescale.

Per-HDU Cells
-------------

Another simple file layout is to put each image plane for each cell in a completely separate FITS image HDU.
This is entirely compatible with FITS tile compression (though we'd almost certainly compress the entire HDU as one tile) and our goals for using FITS WCS.
Stitching images from different HDUs into a coherent whole is probably a bit more likely for a third-party FITS viewer to support than images from different binary tables; it definitely is supported by some viewers in the absence of overlaps, if certain header cards are included.
But a flat list of HDUs for all cells and image planes provides a lot less organizational structure than a binary table (especially a single binary table) for third-party tools to interpret.

Each HDU comes with an extra 3-9 KB of overhead (1-2 header blocks, and padding out the full HDU size to a multiple of 2880 bytes) that cannot be compressed, which is not ideal, but probably not intolerable unless we get unexpectedly good compression ratios or shrink the cell size: an uncompressed 250×250 single-precision floating point image is 250KB, so those overheads should be at most 4% or so.
The overheads would be significant for the PSF images, which we expect to be 25-40 pixels on a side (2.5-6 KB uncompressed).

Subimage reads would be similarly non-ideal but perhaps tolerable.
Because each HDU is so small, it'd be plenty efficient to read full HDUs, but only those for the cells that overlap the region of interest.
Seeking to the right HDUs (or requesting the appropriate byte ranges, in the object store case) is easily solved by putting a table of byte offsets in the primary HDU header, though this isn't something third-party FITS readers could leverage.
That would make for a simple solution to the problem of doing subimage reads over object stores (including compression): we could use the address table to read the HDUs we are interested in in their entirety into a client-side memory location that looks like a full in-memory FITS file holding just those HDUs, and then delegate to CFITSIO's "memory file" interfaces to let it do the decompression.

Data Cubes
----------

In this layout, we'd have one 3-d or 4-d image extension HDU for each plane, with each cell's image a 2-d slice of that higher-dimensional array, and the other dimensions corresponding to a 1-d or 2-d index of that cell in its grid.

This avoids the problem with per-HDU overheads, and it makes an address table much less important, as there are many fewer HDUs.
It also allows compression tiles that comprise multiple cells.

It does not allow us to represent the on-sky locations of cells using FITS WCS, however, and this is probably enough to rule it out.

This approach is neutral w.r.t. the problem of compressed subimage reads against object stores: any solution that worked for a regular, non-cell image would work for this one.

Exploded Images
---------------

This approach is similar to the data cube layout, with one HDU for each image plane, but instead of using additional dimensions to represent the grid, we just stitch all cells into a single larger 2-d image.
This doesn't put the cells onto a consistent meaningful coordinate system, however, because we'd be stitching the outer regions together, not the inner regions, and that means all of the logical pixels in the overlap regions appear more than once (albeit with subtly different PSFs, noise, etc, due to different input epochs, in most cases).

That makes the full image *somewhat* interpretable by humans, though far from ideal - it's a bit similar to the common approach of displaying "untrimmed" raw images with the overscan regions of amplifiers in between the data sections.
This is a slight advantage over the data cube layout, but it seems to be the only one: it suffers from the same incompatibility with FITS WCS, and is similarly neutral for the compressed subimage reads problem.

Stitched Images
---------------

We expect many accesses to our coadd files to be uninterested in the redundant overlap pixel values - instead, many science users will be interested in stitching together the inner cells to form a mostly-seamless patch-level image.
This suggests that we might want to store that stitched inner-cell image for each plane directly as a single HDU, and shunt the overlap pixel values to a different HDU.
Aside from being actively problematic for visualization, the overlap regions are not needed for science analyses that either ignore the PSF or use it via forward modeling (they are important for methods that involve convolution or deconvolutions of the data).

For the inner-cell image, this is ideal: FITS WCS can be used exactly the way it was intended, and third-party FITS viewers will be completely usable without any extra effort.

For the overlap regions, we'd end up with a repeat of our original problem, but with lower stakes: for each original cell, we'd have 4 overlap-region images (top, bottom, left, right) that need to be packed into a binary table, data cube, or stitched image of their own (with this stitching being analogous to the exploded coadd case, since there'd be no meaningful overall coordinate system).

.. figure:: /_static/cell-stitching.png
   :name: cell-stitching
   :target: ../_images/cell-stitching.png

   Left: an "exploded" image with inner cell regions (squares) and a subdivision of the overlap regions (T, B, L, R) labeled.

   Right: a stitched inner-cell image and packings of the overlap regions.
   Each color represents a different FITS HDU; the green inner region would be a single 2-d image, while the blue and purple overlap regions could 2-d images or 3- or 4-d data cubes.

Assuming we don't care about FITS WCS support for the overlap regions, the main problem with this approach is complexity in two places:

- When writing, we'd need to quantize the outer cell image first, and then slice the image into its inner-cell and overlap-region sections, and only then compress the quantized values.
  This isn't something third-party FITS compress-while-writing libraries can do, but if we're doing our own quantization anyway, it'd be straightforward to include.
  Compression would not be compatible with storing the overlap regions in a binary table, but we would have the freedom to compress more than one cell at a time (and even compress more cells at a time in the overlap regions than in the inner regions).

- Access to the outer cells on read would require more complex code and a few more seeks: first read the inner region, then read the four overlap regions (though these can almost certainly be read in pairs), and then put them all together.
  This is not something we'd expect third-party general-purpose readers to ever do, but it's not a terribly complicated specification or a huge burden for, say, someone who wanted to write a Rubin-specific reader in a language other than Python.

This option is also neutral to the problem of compressed subimage reads against object stores.

Dual-interpretation HDUs
------------------------

Tile-compressed FITS image HDUs are actually implemented as binary table HDUs with special columns and headers, and FITS libraries typically allow them to be read directly as binary tables as well as decompressed into images, even if the latter is generally the default.
The FITS standard explicitly permits these binary tables to have additional columns (to be ignored when reading the HDU as a compressed image), which means we can actually combine the exploded image (or data cube) and binary table representations in a single HDU, with a couple of caveats:

- we have to compress each cell independently (this is the most way to compress anyway), in order for the image tile-compression binary table form to have one row for each cell;
- each image plane can only have one image HDU.

This lets us associate WCS information and other metadata with each cell by adding table columns that correspond to standard FITS header values, at the cost of duplicating it for every plane (very little of the per-cell information we'd put in these columns would change from plane to plane).
It's unlikely third-party FITS readers would fully interpret these columns without additional effort - they would most likely be ignored when treating the HDU as an image to be decompressed, while the ``COMPRESSED_DATA`` column would not be recognized as the image-like column expected by the Green Bank convention in the binary table form.
Nevertheless, mixing two well-established interpretations of binary tables to together fully represent our data model is arguably better than inventing a new one.

Hybrid Options
--------------

We can in theory use a different approach for each image plane, though for the most part the arguments are the same for all image planes.
The PSF images are the big exception: they are already intrinsically in their own coordinate system that doesn't have a meaningful WCS, and they are unlikely to ever be lossy compressed (I actually don't have any intuition for whether they'd have good lossless compression ratios).

This makes binary table, exploded image, or data cube storage of PSFs quite attractive, even if other image planes are stored in other ways.

Another attractive hybrid concept is storing the inner cell regions of one or more planes two times, in different ways:

- as a stitched image with aggressively lossy compression, for visualization only;
- along with the outer regions with less aggressively lossy or lossless compression in some format that is natural for third-party readers but not visualization (e.g. binary tables or exploded images).

With sufficiently aggressive quantization, the visualization copy's storage cost may be negligible relative to the full copy.

Metadata Layouts
================

Cell coadd metadata falls into three categories:

- global information common to all cells (identifiers, WCS, grid structure);
- per-cell information with a fixed schema (including coadded aperture corrections and wavelength-dependent throughput);
- visit, detector and other IDs for the observations that contribute to each cell.

While some global information will go into FITS headers (certainly the WCS and some identifiers), we do not want to assume that all global metadata can be neatly represented in the limited confines of a FITS header.
A single-row binary table is another option, but we will likely instead adopt the approach recently proposed for other Rubin image data products on RFC-1030: embedding a JSON document as a byte array in a FITS extension HDU.

A binary table with per-cell rows is a natural fit for the fixed-schema per-cell information, especially if the image data layout already involves a binary table with per-cell rows.
But if we're embedding a JSON document in the FITS file anyway, it might make more sense to store this information in JSON as well; this will let us share code, documentation, and serialization for more complex objects with other Rubin image data products, and that includes sharing the machinery for managing schema changes and schema documentation.

The tables of observations that contribute to each cell is also a natural binary table, but not one with per-cell rows (it's more natural as a cell-visit-detector join table), but once again embedded JSON is an equally viable option.
