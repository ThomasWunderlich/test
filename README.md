# Migrating from MREPO to REPOSYNC
Kent C. Brodie

# BACKGROUND
We are a RedHat shop (in my case, many CentOS servers, and some RedHat as well).   To support the system updates around all of that, I currently use MREPO, an open source repository mirroring tool created by Dag Wieers.   MREPO is an excellent  YUM repository manager that has the ability to house, manage, and mirror multiple repositories.   Sadly for many, MREPO’s days are numbered.   Today I’m going to cover both how to migrate from MREPO to REPOSYNC, and WHY.  REPOSYNC is a simpler alternative that is built into RedHat and CentOS systems.

For me,  MREPO has thus far done the job well.   It allows you to set up and synchronize multiple repositories all on the same single server.   In my case, I have been mirroring RedHat 6/7, and Centos 6/7 and it has always worked great.  I’ve had this setup for years, dating back to RedHat 5. 

While mirroring CentOS with MREPO is fairly trivial, mirroring RedHat updates requires a little extra magic:  MREPO uses a clever “registration” process to register a system to RedHat’s RHN (Red Hat Network) service, so that the fake “registered server” can get updates.

Let’s say you have MREPO and wanted to set up a RedHat 6 repository.  The key part of this process uses the “gensystemid” command, something like this:

```gensystemid -u RHN_username -p RHN_password --release=6Server --arch=x86_64 /srv/mrepo/src/6Server-x86_64/```

This command actually logs into RedHat’s RHN, and “registers” the server with RHN.  And now that this fake-server component of MREPO is now allowed to access RedHat’s updates, it can begin mirroring the repository.  If you log into RedHat’s RHN, you will see a “registered server” that looks something like this:
