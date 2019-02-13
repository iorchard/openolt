:orphan:

How to Build OpenOLT on debian jessie
======================================

The purpose of this document is to guide how to build OpenOLT agent on 
debian jessie.


build environment
-------------------

* Hostname: openolt-build
* Role: openolt build machine
* CPU: 2 cores 
* Memory: 8 GB
* HDD: 50 GiB
* NIC: eth0 (10.3.0.54/8)

Prerequisites
--------------

The following proprietary source code is required to build the OpenOLT 
agent.

* SW-BCM68620_2_6_0_1.zip - Broadcom BAL source and Maple SDK
* sdk-all-6.5.7.tar.gz - Broadcom Qumran SDK
* ACCTON_BAL_2.6.0.1-V201804301043.patch - Accton/Edgecore's patch
* OPENOLT_BAL_2.6.0.1.patch - A patch to Broadcom software to allow
  compilation with C++ based openolt

The user to build OpenOLT agent should have a sudo privilege.
In this document, I assume 'orchard' user has a sudo privilege.


Prerequisites
---------------

To build openolt agent, we need g++ and other utilities.

Install prerequisite packages.::

    orchard@openolt-build:~$ sudo apt update && sudo apt upgrade -y
    orchard@openolt-build:~$ sudo apt install -y g++ make git

Save the prerequisite files in openolt_build directory.::

    orchard@openolt-build:~$ mkdir openolt_build
    orchard@openolt-build:~$ cd openolt_build
    orchard@openolt-build:~/openolt_build$ ls
    ACCTON_BAL_2.6.0.1-V201804301043.patch
    OPENOLT_BAL_2.6.0.1.patch
    SW-BCM68620_2_6_0_1.zip
    sdk-all-6.5.7.tar.gz


Download the openolt source code.::

    orchard@openolt-build:~/openolt_build$ git clone \
                                    https://github.com/iorchard/openolt.git
    Cloning into 'openolt'...
    remote: Counting objects: 4, done
    remote: Total 547 (delta 0), reused 547 (delta 0)
    Receiving objects: 100% (547/547), 338.77 KiB | 0 bytes/s, done.
    Resolving deltas: 100% (237/237), done.

Copy the prerequisite files in openolt/agent/download directory.::

    orchard@openolt-build:~/openolt_build$ cp \
        SW-BCM68620_2_6_0_1.zip \
        sdk-all-6.5.7.tar.gz \
        ACCTON_BAL_2.6.0.1-V201804301043.patch \
        OPENOLT_BAL_2.6.0.1.patch \
        openolt/agent/download/

Apply the patch file to build on debian jessie.::

    orchard@openolt-build:~/openolt_build$ cd openolt
    orchard@openolt-build:~/openolt_build/openolt$ patch -p0 <\
                                                    makefile.in.jessie.patch

Execute configure.::

     orchard@openolt-build:~/openolt_build/openolt$ cd agent
     orchard@openolt-build:~/openolt_build/openolt/agent$ ./configure

Execute the prerequisite make to build.::

    orchard@openolt-build:~/openolt_build/openolt/agent$ make \
            OPENOLTDEVICE=asfvolt16 prereq

Change orchard user's current group to docker since docker is used to build.::

     orchard@openolt-build:~/openolt_build/openolt/agent$ make docker
     orchard@openolt-build:~/openolt_build/openolt/agent$ newgrp docker

Build openolt agent.::

     orchard@openolt-build:~/openolt_build/openolt/agent$ make \
            OPENOLTDEVICE=asfvolt16

Building openolt agent takes a very long time.
So you might want to run the above command with nohup.::

     orchard@openolt-build:~/openolt_build/openolt/agent$ nohup make \
            OPENOLTDEVICE=asfvolt16 &

Then you can forget about it for now and come to see how it is built later.

Lastly make an openolt agent debian package. We need to do something 
beforehand.

Without the change, there will be an error to make deb package.::

    orchard@openolt-build:~/openolt_build/openolt/agent$ make \
                                                OPENOLTDEVICE=asfvolt16 deb
    ...
    dpkg-shlibdeps: error: no dependency information found for /usr/local/lib/libgpr.so.6 (used by debian/asfvolt16/tmp/libgrpc++.so.1)
    Hint: check if the library actually comes from a package.
    dh_shlibdeps: dpkg-shlibdeps -Tdebian/asfvolt16.substvars debian/asfvolt16/tmp/openolt debian/asfvolt16/tmp/libgrpc.so.6 debian/asfvolt16/tmp/libgrpc++.so.1 returned exit code 2
    debian/rules:21: recipe for target 'binary' failed
    make[1]: *** [binary] Error 2
    make[1]: Leaving directory '/home/orchard/openolt_build/openolt/agent/mkdebian'
    dpkg-buildpackage: error: fakeroot debian/rules binary gave error exit status 2
    Makefile:259: recipe for target 'deb' failed
    make: *** [deb] Error 2

The openolt package includes libgrpc++so.1 which depends on libgpr.so.6 so
the package info should include the dependency information for the package 
which has libgpr.so.6 but it is not installed from a package but from
the source compilation.
There is no libgpr package for debian jessie.
That's why dh_shlibdeps shows the error "no dependency information found".

As a workaround, I need to enable --ignore-missing-info option for 
dh_shlibdeps.

dpkg-shlibdeps manpage::

    $ man dpkg-shlibdeps
    ...
       --ignore-missing-info
              Do not fail if dependency  information  can't  be  found  for  a
              shared  library.   Usage  of  this  option  is  discouraged, all
              libraries should provide  dependency  information  (either  with
              shlibs  files,  or  with symbols files) even if they are not yet
              used by other packages.

Let's change debian/rules file to make it work.::

    orchard@openolt-build:~/openolt_build/openolt/agent$ diff -u mkdebian/debian/rules.bak mkdebian/debian/rules
    --- mkdebian/debian/rules.bak	2019-02-12 14:35:04.220000000 +0900
    +++ mkdebian/debian/rules	2019-02-12 14:46:17.596000000 +0900
    @@ -33,8 +33,8 @@
     	cp -a $(CURDIR)/debian/init.d $(DEB_DH_INSTALL_SOURCEDIR)/tmp
     	cp -a $(CURDIR)/debian/logrotate.d $(DEB_DH_INSTALL_SOURCEDIR)/tmp
     
    -#override_dh_shlibdeps:
    -#	dh_shlibdeps --dpkg-shlibdeps-params=--ignore-missing-info -l$(ONLP_LIB_PATH):$(OFDPA_LIB_PATH)
    +override_dh_shlibdeps:
    +	dh_shlibdeps --dpkg-shlibdeps-params=--ignore-missing-info
     
     # avoid auto strip for debug ofagentapp.dbg
     #


Now building a debian package should work.::

     orchard@openolt-build:~/openolt_build/openolt/agent$ make \
            OPENOLTDEVICE=asfvolt16 deb


The deb file is openolt.deb in build/ directory.


To clean all, run make distclean as sudo previlege.::

     orchard@openolt-build:~/openolt_build/openolt/agent$ sudo make distclean

