# COPC – Cloud Optimized Point Cloud

## Use Case

[Cloud Optimized GeoTIFF](https://www.cogeo.org/) has shown the utility and convenience 
of taking a dominant container format for geospatial raster data and optionally 
augmenting its organization to allow incremental "range-read" support over HTTP with it. 
With the mantra of "It's just a TIFF" allowing ubiquitous usage of the data content 
combined with the flexibility of supporting partial reads over the internet, COG has 
found a sweet spot. Its reward is the ongoing rapid conversion of significant raster data 
holdings to COG-organized content to enable convenient cloud consumption of the data 
throughout the GIS industry.

What is the COG for point clouds? It would need to be similar in fit and scope to 
COG:

* Support incremental partial reads over HTTP
* Provide good compression
* Allow dimension-selective reads
* Provide all metadata and supporting information
* Support an [EPT](https://entwine.io/entwine-point-tile.html)-style octree organization for 
  data streaming

## "Just a LAZ"

LAZ (LASZip) is the ubiquitous geospatial point cloud format. It is an augmentation of 
[ASPRS LAS](https://github.com/ASPRSorg/LAS) that utilizes an arithmetic encoder to efficiently 
compress the point content. It has seen a number of revisions, but the latest supports 
dimension-selective access and provides all of the metadata support that normal LAS provides.
Importantly, multiple software implementations ([laz-rs](https://github.com/laz-rs/laz-rs), [laz-perf](https://github.com/hobu/laz-perf), and [LASzip](https://github.com/laszip/laszip)) provide LAZ 
compression and decompression, and laz-perf includes compilation to JavaScript which is 
used by all JavaScript clients when consuming LAZ content. 

## Put EPT in LAZ

The EPT content organization supports LAZ in its current "exploded" organization. Exploded 
in this context means that each chunk of data at each octree level is stored as an individual 
LAZ file (or simple blob, or a zstd-compressed blob). One consequence of the exploded organization 
is large EPT trees of data can mean collections of *millions* of files. In non-cloud situations, 
EPT's cost when moving data or deleting it can be significant. Like the tilesets of late 2000s raster 
map tiles, lots of little files are a problem.

LAZ provides a feature that allows us to concatenate the individual LAZ files
into a single, large LAZ file. This is the concept of a dynamically-sized chunk
table. It is a feature that [Martin Isenburg](https://twitter.com/rapidlasso)
envisioned for quad-tree organized data, but it could work the same for an
octree. Additionally, this chunk table provides the lookups needed for an
HTTP-based client to compute where to directly access and incrementally read
data. 

## Implementation details

### ept.json

This file shall follow the format of EPT specified [here](https://entwine.io/entwine-point-tile.html#ept-json). The `dataType` parameter must be set to `copc` and the `version` parameter must be at least `"2.0.0.0"`.

### ept-data

This folder shall contain a single file named `octree.laz`. This file must abide by the LAZ 1.4 format. Each "node" of the octree shall be stored in the LAZ file as its own compressed chunk (see [here](https://www.cs.unc.edu/~isenburg/lastools/download/laszip.pdf)), with chunks and points within the chunk having no meaningful ordering unless otherwise defined.

### ept-hierarchy

This folder shall contain one or more json files detailing information about the octree nodes, where string keys of `D-X-Y-Z` map to an object. For example, for the root octree node `ept-heirarchy/0-0-0-0.json`:

```json
{
  "0-0-0-0": {
    point_count: 65341,
    chunk_size: 6000,
    chunk_offset: 0
  },
  "1-0-0-0": {
    point_count: 438,
    chunk_size: 40,
    chunk_offset: 6000
  },
  "2-0-1-0": {
    point_count: 322,
    chunk_size: 30,
    chunk_offset: 6040
  },
  "2-0-1-2": {
    point_count: 4332,
    chunk_size: 400,
    chunk_offset: 6070
  },
  "1-0-0-1": {
    point_count: 56209,
    chunk_size: 5000,
    chunk_offset: 11070
  },
  "3-0-0-0": {},
    ...
}
```

#### `point_count`

This integer attribute defines the total number of points stored in this node.

#### `chunk_offset`

This integer value defines the starting byte within the LAZ file for this chunk, relative to the beginning of point data. Therefore, for the first chunk in the LAZ file, the absolute starting byte location within the file will be calculated from `(header: offset to point data) + (chunk_offset) = file offset`, so within the json file, the first chunk will always have a value of `0`.

Note that this value **must** be aligned correctly with the LAZ's internal chunk table. Therefore, removing or adding points within the `octree.laz` file is illegal.

#### `chunk_size`

This value is the byte length of the current chunk relative to `chunk_offset`. Therefore, for any given node at position `i`, 

```
chunk_offset[i] = chunk_offset[i-1] + chunk_size[i-1]
file_offset[i] = chunk_offset[i] + header.offset_to_point_data
```

Each hierarchy node must contain at least the above attributes, except for an empty object `{}` which indicates this node's metadata and its subtree resides in its octree file. For example, in the above example, there must be an additional file `ept-hierarchy/3-0-0-0.json`:

```json
{
  "3-0-0-0": {
    point_count: ...,
    chunk_size: ...,
    chunk_offset: ...
  },
  "4-0-0-0": {
    point_count: ...,
    chunk_size: ...,
    chunk_offset: ...
  },
  "4-0-0-1": {
    point_count: ...,
    chunk_size: ...,
    chunk_offset: ...
  },
  "4-0-0-2": {
    point_count: ...,
    chunk_size: ...,
    chunk_offset: ...
  },
  "5-0-2-2": {
    point_count: ...,
    chunk_size: ...,
    chunk_offset: ...
  }
}
```

See [ept-hierarchy](https://entwine.io/entwine-point-tile.html#ept-hierarchy) for additional details.

### ept-addons

This optional folder allows for the addition of new dimensions to a COPC dataset. These new dimensions are stored independently of the COPC dataset, therefore write-access to the COPC dataset is not necessary to create an addon.

For each additional dimension, there must be one binary data file (optionally compressed with zstandard) and one metadata file defining the dimension attributes. For example, to add an additional dimension `ClusterID` defined as a `int64_t` value, we would have:

`ept-addons/ClusterID.json`
```json

```


Fill in the details of how this all works here

* How is metadata organized and stored?
* Why is it LAZ 1.4 only?
* What about extra dimensions to support preserving scan ordering?
* 

## Software implementations

Which software implements COPC? 
How does it work? 
What options matter are available? 

