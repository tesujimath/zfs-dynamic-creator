zfs-dynamic-creator
===================

Incron based script to create ZFS filesystems automatically on mkdir

This script is used to maintain dynamic ZFS filesystems, i.e. those
created automatically in response to mkdir.  My initial use-case was
to create a ZFS filesystem for each run of an Illumina HiSeq
sequencing machine, in response to the sequencing machine creating a
directory on the fileserver.

**IMPORTANT NOTE**: The approach taken by this script does not provide
a general solution.  It relies critically on a known period of
quiescence in the affected filesystem tree, during which the
operations below are performed.  If there is filesystem activity there
during this period, data loss is likely.  The approach here is
inherently racy, and no amount of tweaking of steps 1 to 6 below can
fix this.  That might not matter in your scenario, but be careful not
to deploy this script naively.

The approach is as follows:

1. Incron watches a directory for changes according to incrontab rules.
   Any directory created, optionally filtered with a name matching
   filter, triggers steps 2 to 6, optionally after a delay for a fixed
   time to wait for a quiescent period.

2. zfs create of new filesystem to match

3. rsync across of any files which got created during the delay

4. renames of the old/new directory/filesystem

5. deletion of the old directory

6. optional update of `/etc/exports`, and call to `exportfs -r`.

There are two modes of operation.

* called with 4 arguments (rootfs, rootdir, filename, event), as by incrontab,
  it watches the rootdir directory for changes, and creates ZFS
  filesystems in response, as above.

* called with 2 argument (rootfs, rootdir), it performs various
  functions as per the options, e.g. setting up /etc/exports to match
  what filesystems have been created in the past.

`rootdir` is a path in the filesystem, and `rootfs` the full name of the
corresponding ZFS filesystem.

The incron entry should arrange to run this script at a time when
the newly created directory is quiescent, as there is a nasty race
condition implicit in performing the steps above.  The `--delay` option
is useful for this.

A suitable entry in a file in incron.d would be this:

    /path/to/rootdir IN_CREATE /path/to/zfs-dynamic-creator --add-export --delay=30 zpool/path/to/rootfs $@ $# $%

An incron.d entry to do directory name filtering usng Python regexps
would be e.g. (using Illumina HiSeq format directory names):

    /path/to/rootdir IN_CREATE /path/to/zfs-dynamic-creator --add-export --delay=300 --only-dirs-matching=^\\d\\d\\d\\d\\d\\d_\\w\\d\\d\\d\\d\\d_\\d\\d\\d\\d_\\w\\w\\w\\w\\w\\w\\w\\w\\w\\w$ zpool/path/to/rootfs $@ $# $%

All actions are logged using syslog.

For the full options, run the script with --help.

Simon Guest, 13/6/2014
