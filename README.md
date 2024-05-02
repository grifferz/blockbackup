# blockbackup suite

Set of tools for incremental backup of block devices

## Features

- incremental backup of block device
- simple deduplication using hard links
- compressing backup on the fly using zstd
- making xxh128 sums of chunks before compression
- full restore at decent speed
- partial restore (slower but any chunk can be restored separately)
- only bash and basic system tools used
- free and open source

## Dependencies
- bash 4.x+
- `xxh128`
- `zstd`

On Debian/Ubuntu:

```
# apt install xxhash zstd
```

# How it works

__Blockbackup__ divides the whole block device into chunks. Each chunk is
saved, compressed (optional) and xxh128 sum is calculated (optional).
__Blockbackup__ also compares every chunk with the same chunk from earlier
backup and makes hard link if they are the same, so no additional disk space is
used.  Because of hard links, it looks like every backup is a full backup; any
backup in the middle could be safely removed; restore from any previous point
in time is easy.

For every backup __blockbackup__ creates a new directory named based on the
current date/time.  After every successful backup the symlink _lastsync_ is
created/updated so that it points to the last backup. This link also points to
the reference directory for comparision.

Destination directory structure:
```
    dest-directory/
    ├── 2016-06-26_211549
    │   ├── 00000
    │   │   ├── 00000000.zst
    │   │   ├── 00000000.xxh128
    │   │   ├── 00000001.zst
    │   │   ├── 00000001.xxh128
    │   │   ├── ........
    │   │   ├── 00000999.zst
    │   │   └── 00000999.xxh128
    │   ├── 00001
    │   │   ├── 00001000.zst
    │   │   ├── 00001000.xxh128
    │   │   ├── 00001001.zst
    │   │   ├── 00001001.xxh128
    │   ................
    └── lastsync -> 2016-06-26_211549
```

When enabled, chunks are also compressed. There are 2 modes of operation:
- compression on the fly (-z). In this mode chunks are compressed while blocks
  are being read.
- compression after whole disk is imaged (-za). This option is intended to
  decrease access disk duration to as short as possible. Whole compression is
  done after reading all data from disk.

Note on chunk size: default is 1MB which is a reasonably good value. For better
performance try using larger chunks, but it may impact deduplication
efficiency.

__Blockrestore__:
- has chunk size autodetect
- zip/nozip autodetect
- two modes of operation:
 - normal (slow) mode where disk offset is calculated for every chunk. By using
   this mode it's possible to restore only part of backup (depending on
   directory contents).
 - fast mode, where all chunks are sorted and then piped to one _dd_ process, so
   it's fast, but can't handle incomplete backups with some files missing.


# Command line options

## blockbackup

    blockbackup -s device -d directory [-z|-za] [-c chunk] [-nx] [-nu] [-np]

### `-s|--source`
Source device.

### `-d|--dest`
Destination folder.

For each run __blockbackup__ creates in this directory a subdirectory named by
date and time. After every successful run it also creates the symlink
*lastsync* pointing to last created subdirectory. This link is used as the
reference for deduplication on the next run.

### `-z|--zip`
Compress during reading (if chunk is new).

### `-za|--zip-after`
Read and compare whole device, run compression at the end.

### `-c|--chunk`
Divide whole block device into chunks of size _chunk_. Default 1MiB.

### `-nx|--no-xxh`
Don't generate xxh128 sum for every chunk.

### `-sc|--start-chunk N`
Start dumping the device from chunk _N_. Default is 0.

### `-ec|--end-chunk N`
End dump at chunk N of device. Default is to dump the whole of the device.

### `-nu|--no-update`
Don't update *lastsync* link at the end.

### `-np|--no-progress`
Don't report progress of long-running operations such as linking and
compression. With a 1MiB chunk size a 1GiB block result will otherwise result
in 1,000 lines of output for dumping out the black and ossibly another 1,000
lines for compression.

### `-ie|--ignore-errors`
Don't stop on errors while reading the device. While using this option we won't
detect end of device so `--end-chunk` must also be specified.

### Choosing compression level
When choosing to compress by `zstd` the default compression level is 3. The
environment variable `ZSTD_CLEVEL` can be set to change this, e.g.:

```
# ZSTD_CLEVEL=1 blockbackup …
```

### Multithreaded compression
When choosing to compress with `zstd` by default this will be single-threaded.
To choose a different number of simultaneous threads, set the `ZSTD_NBTHREADS`
envionment variable, e.g.:

```
# ZSTD_NBTHREADS=4 blockbackup …
```

Setting `ZSTD_NBTHREADS=0` will cause `zstd` to attempt to auto-detect the
number of CPUs.

## blockrestore

    blockrestore -s directory -d device [-c chunk] [-f]

### `-s|--source`
Source directory. This should be the dated directory produced by _blockbackup_
(or specify the *lastsync* symlink).

### `-d|--dest`
Destination block device.

### `-c|--chunk`
Force using chunks size. Normally chunk size is autodetected from first chunk
found in the source directory.

### `-f|--fast`
Use fast restore method. Source directory _must_ be complete with all chunks
present or the result will be corrupted. But it's fast. 😀

# Legal

    Copyright (C) 2016-2018 Marek Wodzinski
                  2024-     Andy Smith

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
