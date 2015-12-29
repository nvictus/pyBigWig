[![PyPI version](https://badge.fury.io/py/pyBigWig.svg)](https://badge.fury.io/py/pyBigWig) [![Travis-CI status](https://travis-ci.org/dpryan79/pyBigWig.svg?branch=WriterIntegration)](https://travis-ci.org/dpryan79/pyBigWig.svg?branch=master)

# pyBigWig
A python extension, written in C, for quick access to and creation of bigWig files. This extension uses [libBigWig](https://github.com/dpryan79/libBigWig) for local and remote file access.

Table of Contents
=================

  * [Installation](#installation)
  * [Usage](#usage)
    * [Load the extension](#load-the-extension)
    * [Open a bigWig file](#open-a-bigwig-file)
    * [Access the list of chromosomes and their lengths](#access-the-list-of-chromosomes-and-their-lengths)
    * [Print the header](#print-the-header)
    * [Compute summary information on a range](#compute-summary-information-on-a-range)
    * [Retrieve values for individual bases in a range](#retrieve-values-for-individual-bases-in-a-range)
    * [Retrieve all intervals in a range](#retrieve-all-intervals-in-a-range)
    * [Add a header to a bigWig file](#add-a-header-to-a-bigwig-file)
    * [Adding entries to a bigWig file](#adding-entries-to-a-bigwig-file)
    * [Close a bigWig file](#close-a-bigwig-file)
  * [A note on coordinates](#a-note-on-coordinates)
  * [Galaxy](#galaxy)

# Installation
You can install this extension directly from github with:

    pip install git+git://github.com/dpryan79/pyBigWig.git

# Usage
Basic usage is as follows:

## Load the extension

    >>> import pyBigWig

## Open a bigWig file

This will work if your working directory is the pyBigWig source code directory.

    >>> bw = pyBigWig.open("test/test.bw")

Note that if the file doesn't exist you'll see an error message and `None` will be returned. Be default, all files are opened for reading and not writing. You can alter this by passing a mode containing `w`:

    >>> bw = pyBigWig.open("test/output.bw", "w")

Note that a file opened for writing can't be queried for its intervals or statistics, it can *only* be written to. If you open a file for writing then you will next need to add a header (see the section on this below).

## Access the list of chromosomes and their lengths

`bigWigFile` objects contain a dictionary holding the chromosome lengths, which can be accessed with the `chroms()` accessor.

    >>> bw.chroms()
    dict_proxy({'1': 195471971L, '10': 130694993L})

You can also directly query a particular chromosome.

    >>> bw.chroms("1")
    195471971L

The lengths are stored a the "long" integer type, which is why there's an `L` suffix. If you specify a non-existant chromosome then nothing is output.

    >>> bw.chroms("c")
    >>> 

## Print the header

It's sometimes useful to print a bigWig's header. This is presented here as a python dictionary containing: the version (typically `4`), the number of zoom levels (`nLevels`), the number of bases described (`nBasesCovered`), the minimum value (`minVal`), the maximum value (`maxVal`), the sum of all values (`sumData`), and the sum of all squared values (`sumSquared`). The last two of these are needed for determining the mean and standard deviation.

    >>> bw.header()
    {'maxVal': 2L, 'sumData': 272L, 'minVal': 0L, 'version': 4L, 'sumSquared': 500L, 'nLevels': 1L, 'nBasesCovered': 154L}

## Compute summary information on a range

BigWig files are used to store values associated with positions and ranges of them. Typically we want to quickly access the average value over a range, which is very simple:

    >>> bw.stats("1", 0, 3)
    [0.2000000054637591]

Suppose instead of the mean value, we instead wanted the maximum value:

    >>> bw.stats("1", 0, 3, type="max")
    [0.30000001192092896]

Other options are "min" (the minimum value), "coverage" (the fraction of bases covered), and "std" (the standard deviation of the values).

It's often the case that we would instead like to compute values of some number of evenly spaced bins in a given interval, which is also simple:

    >>> bw.stats("1",99,200, type="max", nBins=2)
    [1.399999976158142, 1.5]

`nBins` defaults to 1, just as `type` defaults to `mean`.

If the start and end positions are omitted then the entire chromosome is used:

    >>> bw.stats("1")
    [1.3351851569281683]

## Retrieve values for individual bases in a range

While the `stats()` method **can** be used to retrieve the original values for each base (e.g., by setting `nBins` to the number of bases), it's preferable to instead use the `values()` accessor.

    >>> bw.values("1", 0, 3)
    [0.10000000149011612, 0.20000000298023224, 0.30000001192092896]

The list produced will always contain one value for every base in the range specified. If a particular base has no associated value in the bigWig file then the returned value will be `nan`.

    >>> bw.values("1", 0, 4)
    [0.10000000149011612, 0.20000000298023224, 0.30000001192092896, nan]

## Retrieve all intervals in a range

Sometimes it's convenient to retrieve all entries overlapping some range. This can be done with the `intervals()` function:

    >>> bw.intervals("1", 0, 3)
    ((0, 1, 0.10000000149011612), (1, 2, 0.20000000298023224), (2, 3, 0.30000001192092896))

What's returned is a list of tuples containing: the start position, end end position, and the value. Thus, the example above has values of `0.1`, `0.2`, and `0.3` at positions `0`, `1`, and `2`, respectively.

If the start and end position are omitted then all intervals on the chromosome specified are returned:

    >>> bw.intervals("1")
    ((0, 1, 0.10000000149011612), (1, 2, 0.20000000298023224), (2, 3, 0.30000001192092896), (100, 150, 1.399999976158142), (150, 151, 1.5))

## Add a header to a bigWig file

If you've opened a file for writing then you'll need to give it a header before you can add any entries. The header contains all of the chromosomes, **in order**, and their sizes. If your chromosome has two chromosomes, chr1 and chr2, of lengths 1 and 1.5 million bases, then the following would add an appropriate header:

    >>> bw.addHeader([("chr1", 1000000), ("chr2", 1500000)])

bigWig headers are case-sensitive, so `chr1` and `Chr1` are different. Likewise, `1` and `chr1` are not the same, so you can't mix Ensembl and UCSC chromosome names. After adding a header, you can then add entries.

By default, up to 10 "zoom levels" are constructed for bigWig files. You can change this default number with the `maxZooms` optional argument. A common use of this is to create a bigWig file that simply holds intervals and no zoom levels:

    >>> bw.addHeader([("chr1", 1000000), ("chr2", 1500000)], maxZooms=0)

## Adding entries to a bigWig file

Assuming you've opened a file for writing and added a header, you can then add entries. Note that the entries **must** be added in order, as bigWig files always contain ordered intervals. There are three formats that bigWig files can use internally to store entries. The most commonly observed format is identical to a [bedGraph](https://genome.ucsc.edu/goldenpath/help/bedgraph.html) file:

    chr1	0	100	0.0
    chr1	100	120	1.0
    chr1	125	126	200.0

These entries would be added as follows:

    >>> bw.addEntries(["chr1", "chr1", "chr1"], [0, 100, 125], ends=[5, 120, 126], values=[0.0, 1.0, 200.0])

Each entry occupies 12 bytes before compression.

The second format uses a fixed span, but a variable step size between entries. These can be represented in a [wiggle](http://genome.ucsc.edu/goldenpath/help/wiggle.html) file as:

    variableStep chrom=chr1 span=20
    500	-2.0
    600	150.0
    635	25.0

The above entries describe (1-based) positions 501-520, 601-620 and 636-655. These would be added as follows:

    >>> bw.addEntries("chr1", [500, 600, 635], values=[-2.0, 150.0, 25.0], span=20)

Each entry of this type occupies 8 bytes before compression.

The final format uses a fixed step and span for each entry, corresponding to the fixedStep [wiggle format](http://genome.ucsc.edu/goldenpath/help/wiggle.html):

    fixedStep chrom=chr1 step=30 span=20
    -5.0
    -20.0
    25.0

The above entries describe (1-based) bases 901-920, 931-950 and 961-980 and would be added as follows:

    >>> bw.addEntries("chr1", 900, values=[-5.0, -20.0, 25.0], span=20, step=30)

Each entry of this type occupies 4 bytes.

Note that pyBigWig will try to prevent you from adding entries in an incorrect order. This, however, requires additional over-head. Should that not be acceptable, you can simply specify `validate=False` when adding entries:

    >>> bw.addEntries(["chr1", "chr1", "chr1"], [100, 0, 125], ends=[120, 5, 126], values=[0.0, 1.0, 200.0], validate=False)

You're obviously then responsible for ensuring that you **do not** add entries out of order. The resulting files would otherwise largley not be usable.

## Close a bigWig file

A file can be closed with a simple `bw.close()`, as is commonly done with other file types. For files opened for writing, closing a file writes any buffered entries to disk, constructs and writes the file index, and constructs zoom levels. Consequently, this can take a bit of time.

# A note on coordinates

Wiggle and BigWig files use 0-based half-open coordinates, which are also used by this extension. So to access the value for the first base on `chr1`, one would specify the starting position as `0` and the end position as `1`. Similarly, bases 100 to 115 would have a start of `99` and an end of `115`. This is simply for the sake of consistency with the underlying bigWig file and may change in the future.

# Galaxy

pyBigWig is also available as a package in [Galaxy](http://www.usegalaxy.org). You can find it in the toolshed as `package_python_2_7_pybigwig_0_1_9`. The [IUC](https://wiki.galaxyproject.org/IUC) is currently hosting the XML definition of this on [github](https://github.com/galaxyproject/tools-iuc/tree/master/packages/package_python_2_7_10_pybigwig_0_1_9).
