---
Title: Creating a Debian package from scratch
Date: 2018-05-21
---
At [work](https://grnet.gr), we use exclusively Debian across our fleet. We try
to keep up with the latest stable distribution, without relying on third-party
repositories. Also, we try to package every piece of software we're installing
on our boxes, including our IaaS cloud software [Synnefo](https://synnefo.org)
and tools that are not found in upstream Debian repos (i.e.
[check_mk](https://mathias-kettner.de/check_mk.html)). So, it is not uncommon
that we have to create a Debian deb package from scratch.

After dealing with some custom packages, I tried to create and build a Debian
package without using any tools, except `dch(1)` and `dpkg-buildpackage(1)`.
The following is surely not the recommended way to create a package from
scratch, but I was quite curious about the internals and decided to dig a bit
deeper. Everything was done on a jessie box.

Before starting you'll need to install some packages:

```
apt install dpkg-deb devscripts build-essential
```

First of all, I cd'ed into the repo of the software that I want to package. In
our case, it's a dummy repo with some Markdown files that will end up in
`/opt`.

`cd path/to/repo` and I created a tarball of my repo, which ideally should be
your 'upstream' tarball:

```
git archive -o /tmp/deb/koko_1.0.orig.tar.gz HEAD
```

I'm going to assume that the upstream version of our package is 1.0.

Our git tree looks like this:

```
$ tree .
.
└── lala
    ├── file0.html
    ├── file1.html
    └── file2.html

1 directory, 3 files
```

At work, we use a builder VM with multiple chroots, where all packaging is
being done. So, let's copy the "upstream" tarball there and ssh in.

```
scp /tmp/deb/koko_1.0.orig.tar.gz our.fancy.builder.vm:
ssh our.fancy.builder.vm
```
Now, we need a directory where all package files will live. This directory must
have a name of `[package_name]-[upstream_version]`.

```
mkdir koko-1.0
```
and now we're going to extract the tarball in there:

```
tar xfv koko_1.0.tar.gz -C koko-1.0
```
All stuff needed for Debian's tools to finally build the package, live inside
the debian directory:

```
cd koko-1.0
mkdir debian
```
All packages, have a `debian/compat` file, which contains the minimum supported
version of debhelper. Let's set it to 9 (jessie ships version 9).

```
echo 9 > debian/compat
```
Of course, all packages contain a `debian/control` file. This file contains the
most vital information about the source package: Name, priority, maintainer,
dependencies, descriptions. This info is also shown when you run `apt-cache
show [package]`. See here for more info about control files and what each line
means.

```
cat debian/control
Source: koko
Section: misc
Priority: optional
Maintainer: Nikos Kormpakis <nkorb@some.where>
Build-Depends: debhelper (>= 9)
Standards-Version: 3.9.6

Package: koko
Architecture: all
Description: A dummy package.  
 A dummy package, created for learning purposes, that just installs
 some files inside /opt.
```

Now, we have to define the format of our source package. Our package will be a
non-native Debian package. See here for the difference between native and
non-native packages. The package format is defined in `debian/source/format`.
We'll use version 3.0, the latest one.

```
mkdir debian/source
echo "3.0 (quilt)" > debian/source/format
```

Let's add our first entry to our changelog at `debian/changelog`. In this file
we will define our package version number and describe what changed in this
revision. This file will be shipped to the machine that will install the
package, at `/usr/share/doc/koko/changelog.Debian.gz`. Debian provides us a
tool called `dch(1)`, which eases us with the edit of the changelog file. If
you prefer, you can also create manually the file:

```
dch --create -v 1.0-1 --package koko 
```

The file looks like this:

```
koko (1.0-1) jessie; urgency=medium

  * Initial release.

 -- Nikos Kormpakis <nkorb@some.where>  Sun, 23 Jul 2017 21:43:37 +0300 
```

But why did we use version `1.0-1` instead of `1.0`? Remember when we talked about
native and non-native packages? Debian Policy defines a standard way to define
a version of a package. As stated in Section 5.6.12 for more) the format of the
version string is:

```
The format is: [epoch:]upstream_version[-debian_revision]
```

The tl;dr about versioning is (ignoring epoch for now):

If you're building a non-native package (99% of the time) you must define a
`debian_revision`. The `debian_revision` starts from `-1` and increases by one
each time a new version of the package is releases, without changing the
upstream tarball.  If a new upstream tarball is imported, `debian_revision`
started again from `-1`.  So, must be clear now why we used `1.0-1` instead of
`1.0`. `1.0` would imply that we're building a native package.

After all that fuzz, let's define what directories must be created by our package:

```
echo "opt/foo" > debian/koko.dirs
```
This means that during install, the package will create `/opt` and `/opt/foo`
and install our files inside `/opt/foo/lala`.

Now, let's define which files will be copied during installation:

```
echo "lala opt/foo" > debian/koko.install
```

The above means that install everything you will find inside source directory
lala into `/opt/foo`.

Last (but not least!) step: We need a `debian/rules` file. This file, (which is
a Makefile) will be used by `dpkg-buildpackage(1)` to create the actually
package.  For now, we'll use a bare simple rules file. For more gory details,
check Debian's Maintainer Guide, Section 4.4.

```
#!/usr/bin/make -f
%:
        dh $@
```

Please do not use spaces but only tabs. `dpkg-buildpackage(1)` will complain!

Finally, `debian/rules` must be marked as executable:

```
chmod +x debian/rules
```

Now, we're ready to build our package!

```
dpkg-buildpackage -sa -us -uc
dpkg-buildpackage: source package koko 
dpkg-buildpackage: source version 1.0-1
dpkg-buildpackage: source distribution jessie
dpkg-buildpackage: source changed by Nikos Kormpakis <nkorb@some.where>
dpkg-buildpackage: host architecture amd64
 dpkg-source --before-build koko-1.0 
 debian/rules clean
dh clean
   dh_testdir
   dh_auto_clean
   dh_clean
 dpkg-source -b koko-1.0
dpkg-source: info: using source format `3.0 (quilt)'
dpkg-source: info: building koko using existing ./koko_1.0.orig.tar.gz
dpkg-source: info: building koko in koko_1.0-1.debian.tar.xz
dpkg-source: info: building koko in koko_1.0-1.dsc
 debian/rules build
dh build
   dh_testdir
   dh_auto_configure
   dh_auto_build
   dh_auto_test
 debian/rules binary
dh binary
   dh_testroot
   dh_prep
   dh_installdirs
   dh_auto_install
   dh_install
   dh_installdocs
   dh_installchangelogs
   dh_perl
   dh_link
   dh_compress
   dh_fixperms
   dh_installdeb
   dh_gencontrol
   dh_md5sums
   dh_builddeb
dpkg-deb: building package `koko' in `../koko_1.0-1_all.deb'.
 dpkg-genchanges -sa >../koko_1.0-1_amd64.changes
dpkg-genchanges: including full source code in upload
 dpkg-source --after-build koko-1.0
dpkg-buildpackage: full upload (original source is included)
```

After all that stuff, your directory must look something like this:

```
ls -l
total 732
drwxr-xr-x 4 nkorb nkorb    4096 Jul 23 23:12 koko-1.0
-rw-r--r-- 1 nkorb nkorb     748 Jul 23 23:12 koko_1.0-1.debian.tar.xz
-rw-r--r-- 1 nkorb nkorb     854 Jul 23 23:12 koko_1.0-1.dsc
-rw-r--r-- 1 nkorb nkorb  359668 Jul 23 23:12 koko_1.0-1_all.deb
-rw-r--r-- 1 nkorb nkorb    1584 Jul 23 23:12 koko_1.0-1_amd64.changes
-rw-r--r-- 1 nkorb nkorb  360569 Jul 23 23:12 koko_1.0.orig.tar.gz
```

So, let's install the package we built.

```
# dpkg -i koko_1.0-1_all.deb 
Selecting previously unselected package koko.
(Reading database ... 295273 files and directories currently installed.)
Preparing to unpack koko_1.0-1_all.deb ...
Unpacking koko (1.0-1) ...
Setting up koko (1.0-1) ...

tree /opt/foo
/opt/foo
└── lala
    ├── file0.html
    ├── file1.html
    └── file2.html

1 directory, 3 files
```

Exactly as we expected! :) Many credits to the writers of the [Debian Packaging
Intro](https://wiki.debian.org/Packaging/Intro)!

Finally, there are some links about Debian packaging, if you're interested:

[Debian New Maintainers' Guide](https://www.debian.org/doc/manuals/maint-guide/)

[Debian Packaging Tutorial by Lucas Nussbaum](https://www.debian.org/doc/manuals/packaging-tutorial/packaging-tutorial.en.pdf)

[Debian Policy](https://www.debian.org/doc/debian-policy/)

Have fun packaging!
