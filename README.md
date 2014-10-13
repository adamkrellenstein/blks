# Changelog

* Version 1—initial publication


# Dependencies

* [python3](http://python.org)
* [python-bitstring](http://pythonhosted.org/bitstring/) (for Python 3)


# Installation

`blks` is not designed to be installed―it is a single file to be run [by a Python 3 interpreter].

	$ chmod +x blks
	$ ./blks --help

or

	$ python3 blks --help
	

# Description

`blks` is very small program designed to produce a **description of a file as divided into a particular number of blocks** and to **compare a file with such a description**. That is, it may **write** out a list of blocks that a file may be divided into, and it may **read** a file and try to find those blocks within it.

`blks` may be used to compare two files block‐wise, and asynchronously (so that the two files may actually be one file at two different times). The files may be large, and they may be binary. `blks` may be used to give a quick impression of how similar two files are, and where in their bulks they differ.

`blks` identifies blocks of a file in two ways: by a short excerpt of the [binary] data from the head of the block, and by the hash of the block as a whole.

`blks` is simple and memory‐efficient.


# Usage

* `blks` has two primary modes: **write** and **read**. `blks write TARGET K` prints to stdout a CSV table that describes the K blocks; `blks read TARGET` reads such a CSV from stdin and prints to stdout a new CSV which identifies the location of each of the described blocks, if it was indeed found in the second file. If a block that exists in the first file cannot be found in the second, then the ‘location’ of that block is ‘None’.

* For command‐line options and switches, see `blks --help`, `blks write --help` and `blks read --help`.

* `blks` works bit‐wise, so that if two files differ by insertions or deletions of bits but not bytes, the common data are still found.

* `blks write` uses SHA‐1 by default, for fast, entropic, non‐cryptographic hashing (as `git` does).

* `blks read` has no difficulty finding unequally sized blocks.

* Input to `blks read` must be a CSV with the following properties:

	* It must begin with a header line whose fourth element is the hash algorithm used in the CSV’s generation and named in accordance with the scheme adopted by the `hashlib` Python module.

	* It must have at least four fields.

	* The first two fields must be natural numbers.

	* The third field must be written in standard base64.

	* The fourth field must be written in hexadecimal.

	* The fields themselves cannot contain any commas.

	Furthermore:

	* The length of each of these fields is unspecified.

	* The number of lines is unspecified.

* The input (output) CSVs are parsed (written) as follows:

	* Locations and sizes in the CSVs are [to be given] in bits, not bytes.

	* The first block is block 1, not 0.

	* Null excerpts are wildcards: slow but acceptable. (Searching without excerpts can be *very* (read ‘impossibly’) slow even for rather small files. It can also be perfectly reasonable, i.e. if the searched‐for share follows hard upon the share last discovered.)

	* Null hashes assert absolutely the integrity of the corresponding share (and so may be used to force its inclusion in the decoding process).

	* Block numbers are not zero‐padded (in output).

* The hash algorithms available are given by `import hashlib; hashlib.algorithms_available`.

* Exit codes accord to the following standard: [FreeBSD Library Functions Manual: SYSEXITS(3)](http://www.freebsd.org/cgi/man.cgi?query=sysexits).

* Note that `blks read` won’t tell you (directly) if there is data beyond the last share (or before the first). The following script will do that:

	#!/bin/bash

	result=$(blks read $foo $k < "$catalogue")

	if ! [ $(echo "$result" | head -n 2 | tail -n 1 | cut -f 3) == 0 ]; then echo "Data precede first share."; fi

	last_share_location=$(echo "$result" | tail -n 1 | cut -f 3)
	if ! [ "$last_share_location" == "None" ]; then
		last_share_size=$(echo "$result" | tail -n 1 | cut -f 2)
		if [ $(($last_share_location + $last_share_size)) -lt $(stat -c %s "$foo") ]; then echo "Data succeed last share."; fi
	fi


# Examples

* Examine how the parts of file change over time.

	See file ‘foo’ divided into five blocks:

		$ blks.py write foo 5 > foo.blks
		$ cat foo.blks
		number,size,excerpt,sha1
		1,6558520,IQqPs+VHS5twff49aI+4,50e52a5bf4b1807ff745cff0469cd5ef507dc434
		2,6558520,/8g5qTotnezlVKT+0Kwl,d9aec61f0a54531f0f84f2b4cabf5a096e47bb95
		3,6558520,sGvPIln2O1UYvAkSwWB5,aa4130427bda62b8268cae0cba45d3cb2b15099f
		4,6558520,DbiY8hLv3zDKKD0Jf23v,257f08b0f1fa249183bbc5ab6a3b78e808c5ce72
		5,6558496,yY+KoBW3GbftcBlLtuAV,b55006652e4caaf8c3d893685e4b217061ea0d3b

	Change ‘foo’ somewhere in the fourth block (e.g. using `hexedit(1)`):

		$ hexedit foo

	Try to find each of the original blocks in the new version of ‘foo’:

		$ blks.py read foo < foo.blks
		number,size,location
		1,6558520,0
		2,6558520,6558520
		3,6558520,13117040
		4,6558520,None
		5,6558496,26234080

* Compare files ‘spam’ and ‘eggs’ block‐wise, with resolution equal to the size of ‘spam’ divided by three.

		$ blks write spam 6 | blks read eggs
		number,size,location
		1,5465432,None
		2,5465432,5465432
		3,5465432,10930864
		4,5465432,16396296
		5,5465432,None
		6,5465416,27327160

	So ‘spam’ and ‘eggs’ differ exactly in the first and fifth blocks.

* Divide ‘bar’ into nine‐hundred parts, identifying blocks by their first three bytes, and hashing with MD5:

		$ blks.py write bar 9 --excerpt-size=3 --hash-algorithm=md5 
		number,size,excerpt,md5
		1,36440,u7u7,262dfe38bbf17c60f7f5e7676221fcfb
		2,36440,Soaz,f914b3adbdeb4037018f8e5622b09cc8
		3,36440,QJcW,9a5ba4bf0f05e7ba0d995e5d61836742
		[…]
		898,36440,NQ43,6f4114a389717e4137436309ae5ac6e7
		899,36440,MhH5,9d5bdffdf74d97299de0da94e788ce35
		900,33016,LRYW,e59c3fe308f344d64a68da54cf5b2d4f

* Pipe the output of `blks read` to `column` for pretty‐printing, checking for block transposition.

		$ blks read qux --check-transposition < qux.blks | column -t -s ','
		number  size     location
		1       8345896  0
		2       8345896  8345896
		3       8345888  16691792
