# unixfreeware
Modern Freeware for IRIX 6

&nbsp;&nbsp;


# Package Building
***NOTE:***  *This is a quick attempt at "bootstrapping" packages.  This should be automated either by shell scripts or Ansible (once Python 3.7+ is compiled)*

## Basic Layout
The basic layout of the build system is as follows:

&nbsp;&nbsp;
###/devel/unixfreeware/src/pkg_name/pkg_name-version
This is the first directory where source packages are put for compilation.  It is organized into package name/package name-version.  This is because we may look at multiple packages for either specific compatibilities and features, or simply, we might go through a number of versions to find the one that compiles for IRIX.

Inside this directory, we configure and build the package.  See building setup below.

The most basic configuration options should be as follows using `sed` as an example:
`./configure --prefix=/devel/unixfreeware/builds/sed-4.5`

This simply configures the package to build with the output being sent to the builds directory.

If you need to do additional package options, you can do them similar to this for `sed`:
`./configure --prefix=/devel/unixfreeware/builds/sed-4.5 --disable-dependency-tracking`

Once done, typically you will also run `gmake -jN` where N is the number of processors +1 on the system.  Then run:

`su -`

followed by `gmake install`


&nbsp;&nbsp;
###/devel/unixfreeware/builds/<pkgname-version>
This is where the built output of the package lives.  For ease of use, the `/builds/` directory may be very open security wise so that normal users can write to it.  This is fine, we can fix this during the package creation, but some packages have had issues installing as root.



&nbsp;&nbsp;
###/devel/unixfreeware/dist/<pkgname-version>
This is where the package building happens.  This relies on a modified `nekotool` script, renamed to `ufwtool`.  A few of the variable names have changed.  In the future this needs to be much more automated.

To run this, `cd /devel/unixfreeware/dist` and then run `mkdir pkgname-version` (i.e. sed-4.5), then `cd pkgname-version`.

Run:  `export UFWPKGNAME=pkgname`
Run:  `export UFWPKGVERSION=pkgversion`
Run:  `/usr/nekoware/bin/ufwtool`

It will build scripts and ask if you want to run the `swpkg` tool.  Say No here.  You will now have the following directories:
1. dist  
2. patches  
3. relnotes  
4. src  

dist will have two files: `ufw_pkgname.idb` and `ufw_pkgname.spec`.  The .spec file you can leave alone for now - we don't mess with how `ufwtool` makes that.

the `ufw_pkgname.idb` file will hold the list of files and where they go.

For now, use the `swpkg` tool to build this.

#### swpkg
Run this tool and go to File -> Open -> Both .idb and .spec

You will be prompted to load each file one at a time.  You might need to edit the version information in the first screen (this is a bug that ufwtool doesn't always seem to get right).

In the screen to select the source files, select the files (you can do this in bulk) and choose to add them.  By default all files will be tagged as `root:root` and will be `0755` for binries.

There are multiple types of options for packaging.  The most common and default will be the `eoe` which is the actual package install, but there is also a source code option, documentation option, etc.  Typically I have just been doing the default until we get to a point we can break this out easier.

Save both files (File-> Save -> Both) and then exit.


&nbsp;&nbsp;
#### Fixing the .idb
Now open the .idb file in vi or vim if you have it installed on nekoware (for now) and you will see lines like this:  `d 0755 root sys devel/unixfreeware/builds/gcc-4.7.4 devel/unixfreeware/builds/gcc-4.7.4 ufw_gcc.sw.eoe`

The above line is from the GCC original idb file before being fixed.

**MAKE SURE** that you run the following command in vi:
`:%s,root sys devel/unixfreeware/builds/pkgname-version,root sys usr/unixfreeware,g`

In the above example we would do:
`:%s,root sys devel/unixfreeware/builds/gcc-4.7.4,root sys usr/unixfreeware,g`

The reason why we do this is because column 5 and 6 are identical when you select the list of items in `swpkg`, so we have to match column 5 with columns 3 and 4 so that we *don't* also manipulate column 6.

The columns are as follows:
1. (d)irectory or (f)ile  
2. owner  
3. group  
4. destination  
5. source  
6. component (software, documentation, source, etc.)  


**NOTE:** The above can easily be automated through a shell script, I just haven't gotten to it yet and if we do it would really speed up the package building process.


#### Building the package with gendist
Now that we have the components in place, we can run `gendist`.  `gendist` MUST be run as root and has some quirks.

FIRST, make sure the root user has a pristine environment - changing the path especially will mess up `gendist`.

SECOND, the destination directory, **MUST** be owned by `root:root` and **MUST** be set to permission `777`.

Create the directory (using sed-4.5 as the example):
`mkdir -p /devel/dist/mips3/sed-4.5/`

The reason for this other `dist` directory is because this is where the dist artifacts go.  It is kind of annoying because there are two dist directories, one for creating the files to build the package and then one (by convention?) where the dist artifacts are built.  We can change this if necessary.

The other thing is you will notice `mips3`.  This is because I accidentally started creating `gcc` as mips4 which ruled out a lot of architectures for IRIX, but I decided to add this identifier in in case we do ever make a `mips4` based system or `mips4` based packages for specific use cases.  If we don't we can drop this directory.

Once you have created the directory and set the permissions, run the following as `root` from inside the the `/devel/unixfreeware/dist/pkgname-version` directory:  

`/usr/sbin/gendist -sbase / -idb dist/ufw_sed.idb -spec dist/ufw_sed.spec -dist /devel/dist/mips3/sed-4.5/ -verbose -nocompress -all`

What this does is runs `gendist` specifying the necessary files.  We are telling it where the dist destination is.  We are telling it to build all components, be verbose and to not compress things.

Compression is something we can look at.  Inside the artifacts what `gendist` does is builds a special zip file with the file markers of where the block sizes are.  If you choose to not compress the files, then you don't get the block markers and it's easier to update the final file if we need to.  If we choose not to do this or want to save disk space, we can enable compression.

Once done, assuming it was successful, you will see a number of files in the `/deve/dist/pkgname-version` directory:  
1. `ufw_pkgname`  
2. `ufw_pkgname.idb`  
3. `ufw_pkgname.man`  
4. `ufw_pkgname.opt`  
5. `ufw_pkgname.sw`   

Some of these files will be very few bytes and be empty files essentially.  That's fine.  If you chose to mark some components as these other files they would be bigger.

You have created a distribution!

To package it into a meaningful distributable way:
`tar cvf /devel/unixfreeware/releases/mips3/tardists/ufw_pkgname-version-YYYYMMDD.tardist *`


&nbsp;&nbsp;
###/devel/unixfreeware/releases/<pkgname-version>
This stores the final `.tardist` artifacts from the build



&nbsp;&nbsp;
####/devel/dist/{mips3, mips4}
This is where the built artifacts are for the packages *before* they are packaged for distribution.
