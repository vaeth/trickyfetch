# trickyfetch

(C) Martin Väth (martin at mvath.de).
The license of this project is the GNU Public License GPL-2.
SPDX-License-Identifier: GPL-2.0-only

This project is for the Gentoo portage system to cleanup or
organize your `$DISTDIR`.
You can also use it to temporarily enable/disable `getdelta` or to
log your fetches.

### Installation

There is an ebuild for this project in the mv overlay (available over layman).

### Description

There are several scripts in the net or in the gentoo repository which should
cleanup your `$DISTDIR`, for instance `eclean` from `app-portage/gentoolkit`.
However, these scripts either remove too much or too less
(because, e.g. they do not reliably follow your `USE` flags).

Moreover, they cannot straightforwardly be used if you want to create
a "minimal" `$DISTDIR` for another system (which e.g. has no internet
connection and limited disc space.)
Also, you can usually not find out easily with such scripts if your `$DISTDIR`
is no longer up-to-date, because e.g. there were silent package bumps.

A solution to all these problems is to use a wrapper over `FETCHCOMMAND`
to log the files which are actually fetched.

The script `trickyfetch` of this project is such a wrapper:
To install it, simply copy `bin/trickyfetch` (with executable permissions)
into a directory from `$PATH`, and `etc/trickyfetch.conf` into `/etc` or into
another directory from `$PATH` and then define in your
`/etc/portage/make.conf` (which was previously called `/etc/make.conf`):

`FETCHCOMMAND="trickyfetch \"\${URI}\" \"\${DISTDIR}\" \"\${FILE}\""`

If you already had set `FETCHCOMMAND`, you can put this instead into
`LOCALFETCHCOMMAND` (the `trickyfetch` script will use this similarly as
portage would use your `FETCHCOMMAND`; for using `getdelta.sh`, see below).

This change will not modify your "ordinary" work with portage.
However, the main idea is that now you can cleanup your `$DISTDIR` with
the following steps:

1. Create a subdirectory `$DISTDIR/.obsolete` and move all regular (i.e. not
   starting with `.`) files of `$DISTDIR` into this subdirectory:
   - `cd` _$DISTDIR_  (substite to the correct value for _$DISTDIR_, of course)
   - `mkdir .obsolete`
   - `mv -i -- * .obsolete`

2. Make portage fetch all missing files from DISTDIR:
   - `PORTAGE_CHECKSUM_FILTER='-* size' emerge system world`

   If you want to be more careful (and use __portage-2.2__ or newer),
   use instead better
   - `PORTAGE_CHECKSUM_FILTER='-* size' emerge -feD --with-bdeps=y @system @world`

There is also a tiny `bin/obsolete` script in this project which does
exactly the two steps above (with the __portage-2.2__ variant).
You might also want to copy this script to your path if you use it often.

That's it: Afterwards, your `$DISTDIR` directory will contain all required
files, and the `$DISTDIR/.obsolete` subdirectory contains the obsolete files.

All the magic happens in step 2: Instead of actually fetching the files from
the net, the `trickyfetch` script will just move the files from the
`$DISTDIR/.obsolete` subdirectory into `$DISTDIR`.

Of course, files which cannot be found in that subdirectory will be
fetched from the net, as usual.

Optionally, you might want to set the environment variable
-	`LOCALFETCHCOMMAND=false`

in step 2 (or when running the obsolete script):
This way, there is never any access to the net, even if files are missing
or their size/checksum in the Manifest is wrong. (Of course, if you do this,
`emerge` will report corresponding errors and might rename corresponding
files due to checksum errors.)

The `trickyfetch` script also has a lot of additional features:

1. It supports `app-portage/getdelta` and can temporarily switch it off.
2. It supports logging for the files actually attempted to fetch from the net.
3. It supports `LOCALFETCHCOMMAND` to define your own way of fetching data.
4. It supports `wget` as a fallback (`WGET`, `WGET_EXTRA_OPTS`)
5. It helps you to keep certain files permanently and "organize" your
   `$DISTDIR`. This is also supported by the `bin/distlinks` script provided
   by this project.

These points must be explained in more detail:

### 1. Support for app-portage/getdelta

This is optional; support for this can be enabled or disabled
in `/etc/trickyfetch.conf`

Note that current versions of `app-portage/getdelta` are broken, see
http://bugs.gentoo.org/show_bug.cgi?id=239439

(Perhaps the one of the mv overlay is still working, although
it hasn't been tested now since ages...)

If you have installed `app-portage/getdelta`, the `trickyfetch` script
will use it automatically, i.e. in contrast to the description of
`app-portage/getdelta` you need not modify anything if you use `trickyfetch`.
(Provided, of course, that you enabled the support at all in
`/etc/trickyfetch.conf`).

If you want to disable __getdelta__ (even if you enabled the support in
`/etc/trickyfetch.conf`), either set the environment variable
`NODELTA=*` (either locally or in your `make.conf`), or create
the file `$DISTDIR/.nodelta`.
Note that creating/removing that file has the advantage that you can change
"on the fly" (during portage is fetching) whether __getdelta__ should be used
for further packages or not.

The variable `NODELTA` mentioned above contains a (space-separated)
list of masks; if the name of the file which should be fetched
matches a mask listed in `NODELTA`, then __getdelta__ is not used.

In addition to `NODELTA` you can also use `EXCLUDE_DELTA`:
If `EXCLUDE_DELTA` contains the name of a file, then the masks
(with the same meaning as in `NODELTA`) are read from this file.
If `EXCLUDE_DELTA` contains the name of a directory, the masks are
read from all non-hidden files in this directory or subdirectories.

I recommend to set `EXCLUDE_DELTA` in your `make.conf` and to set
`NODELTA` only locally for specific commands.

If your `GENTOO_MIRROR` contains more than one mirror, __getdelta__ is not used
if you are fetching from a later mirror than the first: This avoids that
__getdelta__ is used after it already failed for the first mirror.
(This test can be faster if you have `app-portage/eix` installed).

If you do not have __getdelta__ installed (or if __getdelta__ is excluded
or failed), a default `wget` command is used.

### 2. Support for logging

This can be setup in `/etc/trickyfetch.conf`.
See that file for details.

### 3. `LOCALFETCHCOMMAND` support for your own scripts

If you define `LOCALFETCHCOMMAND`, this will be called after logging,
but instead of any net access (i.e. getdelta or wget will not be called).
`LOCALFETCHCOMMAND` takes the role of __portage__'s original `FETCHCOMMAND`
(and must be set completely analogously if you want to use it).

Note that you can make `LOCALFETCHCOMMAND` point to your own script in which
you can in turn call `trickyfetch` - if you define `LOCALFETCHCOMMAND=''`
(or unset it) this will not cause a recursion, but causes `trickyfetch`
to proceed with __getdelta__.

As mentioned earlier, in the context of e.g. the `obsolete` script, you might
want to define temporarily
-	`LOCALFETCHCOMMAND=false`

to avoid any internet access when that script is run, but in this case
you might have to manually care later about renamed files by portage.

### 4. `WGET`, `WGET_EXTRA_OPTS`

In `WGET` you can define the path to `wget` and "standard" options to `wget`.
Of course, you can use another tool instead of `wget`; however, this tool
has to support the options `-P` and `-O` of `wget` which are passed immediately
after `$WGET`.
`WGET_EXTRA_OPTS` is meant to be set temporarily in special situations
(e.g. when you need to pass options like `--no-check-certificate`
which you do not want to pass regularly, of course).
Both variables are `eval`'ed, that is, you can use quoting or even write
a shell function in `WGET` to handle the option `-P`/`-O` and its argument.
Be aware that due to this `eval` even `WGET_EXTRA_OPTS` can be misused to
inject arbitrary shell commands into the fetch process!

### 5. Keep certain files permanently and "organize" your `$DISTDIR`

Directories of the form  `$DISTDIR/.save`* (you can have several of those)
are treated similarly to `$DISTDIR/.obsolete` by trickyfetch, but with a
lower priority. In other words, trickyfetch works as follows:
It first checks if the file can be found in `$DISTDIR/.obsolete`,
then it checks if it can be found in some `$DISTDIR/.save`*
(alphabetical order).
The first matching file is moved out of this subdirectory into `$DISTDIR`.
Only if no such file can be found, the file is fetched from the internet.

Originally, the point here was to move files into `$DISTDIR/.save` which
you think you need in the near future.
However, this is not a permanent solution, since `trickyfetch` will move
the files from `$DISTDIR/.save` to `$DISTDIR`,
once portage needs to fetch them.

For some files, e.g. for fetch-restricted files, you might want a more
permanent solution. This is also supported by `trickyfetch`:
trickyfetch will always create symlinks to the files in the special directory
`$DISTDIR/.restricted`.
Thus, you can e.g.  move fetch-restricted files in this subdirectory, and
these files need not be fetched by `trickyfetch`.

A different kind of permanent solution using the `$DISTDIR/.save`* directories
can be obtained with the `bin/distlinks` script.

The usage of this `distlinks` script is somewhat more complex and explained
in the following. This approach is reasonable, e.g. if you have a
"main machine" from which you want to copy only the relevant portions
of `$DISTDIR` to other machines (e.g. with limited disk space and no
internet connection).

The usage of the `bin/distlinks` script requires some preparation
(besides copying it into your `$PATH`, of course):
You must move `$DISTDIR` one directory level down, e.g. if you originally had

`DISTDIR=/usr/portage/distdir`

you should now create a subdirectory like

`DISTDIR=/usr/portage/distdir/portage-distdir`

and move the original content of `/usr/portage/distdir` into that subdirectory.
Do not forget to set the new correct DESTDIR in your make.conf!

The idea is that within the original `/usr/portage/distdir` (in the following
called `$PARENT`) you can add further directories like
-	`$PARENT/overlays`
-	`$PARENT/laptop`
-	`$PARENT/local-projects`

into which you can move files from `$DISTDIR` which should never be deleted.
The purpose of the `distlinks` script is to organize corresponding
`$DISTDIR/.save.`* subdirectories of symbolic links.
For instance, if you have the subdirectories of the above example, the call
-	`distlinks` (which is a shortcut for `distlinks create`)

will create the subdirectories
-	`$DISTDIR/.save.overlays`
-	`$DISTDIR/.save.laptop`
-	`$DISTDIR/.save.local-projects`

and fill them with symbolic links to the corresponding files in `$PARENT/...`
(and by default it will cleanup existing such symbolic links from
`$DISTDIR` or `$DISTDIR/.obsolete`).

This way, `trickyfetch` will use the files from `$PARENT` (by moving them out
of `$DISTDIR/.save.`* if necessary), and you can see which of these files
have actually been fetched after the last call of "distlinks create"
by looking at the content of `$DISTDIR/.save.`*

Note that for efficiency reasons, the `trickyfetch` script does not treat
subdirectories of `$DISTDIR/.save`* (or `$DISTDIR/.restricted`).
However, the `distlinks` script also allows subdirectories.
For instance, you can have
-	`$PARENT/overlay/local-overlay`
-	`$PARENT/overlay/layman-overlays/mv`

The `distlinks` script will create links from `$DISTDIR/.save.overlays/`*
to all files contained in some of the above directories. It will not warn
about name collisions arising in this way - the first matching name is used.

Note that the symbolic links in `$DISTDIR/.save`* must be absolute, since the
links should still be valid, if they are moved into `$DISTDIR`.
The `distlinks` script will create such absolute links
(generating them from the `$DISTDIR` path).

The `distlinks` script can also remove the links and empty `$DISTDIR/.save`*
subdirectories. Some directories are special for `distlinks`:
-	`$PARENT/empty`
-	`$PARENT/fetch`

The content of these directories is not linked into `$DISTDIR/.save.`*

The `$PARENT/empty` directory is created/removed by with appropriate
permissions, so that you can simply `mv` or `cp -a` this directory
to create a new one.
The `$PARENT/fetch` directory is meant to be used as storage directory
for live ebuilds. To use it, put into your make.conf something like
```
ESCM_STORE_DIR=$DISTDIR/../fetch
ECVS_TOP_DIR=$ESCM_STORE_DIR
EBZR_STORE_DIR=$ESCM_STORE_DIR
ECVS_STORE_DIR=$ESCM_STORE_DIR
EGIT_STORE_DIR=$ESCM_STORE_DIR
EHG_STORE_DIR=$ESCM_STORE_DIR
EMTN_STORE_DIR=$ESCM_STORE_DIR
ESVN_STORE_DIR=$ESCM_STORE_DIR
```
The `distlinks` and `obsolete` script runs slightly faster if
`app-portage/eix` is installed.

To find out how to use `distlinks` exactly, use
-	`distlinks -h`
