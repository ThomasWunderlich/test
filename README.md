# Migrating from MREPO to REPOSYNC
Kent C. Brodie

# BACKGROUND
We are a RedHat shop (in my case, many CentOS servers, and some RedHat as well).   To support the system updates around all of that, I currently use MREPO, an open source repository mirroring tool created by Dag Wieers.   MREPO is an excellent  YUM repository manager that has the ability to house, manage, and mirror multiple repositories.   Sadly for many, MREPO’s days are numbered.   Today I’m going to cover both how to migrate from MREPO to REPOSYNC, and WHY.  REPOSYNC is a simpler alternative that is built into RedHat and CentOS systems.

For me,  MREPO has thus far done the job well.   It allows you to set up and synchronize multiple repositories all on the same single server.   In my case, I have been mirroring RedHat 6/7, and Centos 6/7 and it has always worked great.  I’ve had this setup for years, dating back to RedHat 5. 

While mirroring CentOS with MREPO is fairly trivial, mirroring RedHat updates requires a little extra magic:  MREPO uses a clever “registration” process to register a system to RedHat’s RHN (Red Hat Network) service, so that the fake “registered server” can get updates.

Let’s say you have MREPO and wanted to set up a RedHat 6 repository.  The key part of this process uses the “gensystemid” command, something like this:

```gensystemid -u RHN_username -p RHN_password --release=6Server --arch=x86_64 /srv/mrepo/src/6Server-x86_64/```

This command actually logs into RedHat’s RHN, and “registers” the server with RHN.  And now that this fake-server component of MREPO is now allowed to access RedHat’s updates, it can begin mirroring the repository.  If you log into RedHat’s RHN, you will see a “registered server” that looks something like this:

# IF IT AIN’T BROKE, DON’T FIX IT, RIGHT?

So what’s the issue?  For you RedHat customers, if you’re still using RHN in any capacity, you hopefully have also seen this notice by now:

Putting this all together:  If you’re using MREPO to get updates for RedHat servers, that process is going to totally break in just over 7 months.  **MREPO’s functionality for RedHat updates depends on RedHat’s RHN, which goes away July 31st.**

Finally, while MREPO is still used widely, it is worth noting that it appears continued development of MREPO ceased over four years ago.    There have been a scattering of forum posts out there that mention trying to get MREPO to work with RedHat’s new subscription-management facility, but I never found a documented solution that works.

# WHAT IS REPOSYNC?

Reposync is a command-line utility that’s included with RedHat-derived systems as part of the yum-utils RPM package.   The beauty of reposync is its simplicity.  At the core, an execution of reposync will examine all of the repositories that the system you’re running it on has available, and downloads all of the included packages to local disk.  Technically, reposync has no configuration.  You run it, and then it downloads stuff.  MREPO on the other hand, requires a bit of configuration and customization per repository.

# HOW DOES REPOSYNC SOLVE THE PROBLEM OF RHN DISAPPEARING?
You simply have to think about the setup differently.   In our old model, we had one server that acted as the master repository for all things, whether it was RedHat 6, CentOS 7, whatever.  This one system was “registered”, multiple times, to mirror RPMS for multiple operating system variants and versions.

In the new model, we have to divide things up.   You will need one dedicated server per operating system version.   This is because any given server can only download RPMs specific to the operating system version that server is running.  (Fortunately with today’s world of hosting virtual machines, this isn’t an awful setup, it’s actually quite elegant).    In my case, I needed a dedicated server for each of:   RedHat 6, RedHat 7, CentOS 6, and CentOS 7.

For the RedHat servers, the elegant part of this solution deals with the fact that you no longer need to use “fake” system registration tools (aka gensystemid).  You simply register each of the repository servers using RedHat’s preferred system registration:   the “subscription-manager register” command that RedHat provides (with the retirement of RHN coming, the older rhn_register command is going bye-bye).   MREPO, at present, does not really have a way to do this using RedHat’s “new” registration mechanism.

# SIMPLE REPOSYNC EXAMPLE
The best way for you to see how reposync works is to try it out.   For this example, I highly recommend starting with a fresh new server.    Because I want to show the changes that occur with subscribing the server to extra channel(s), I am using RedHat 6.     (You are welcome to use CentOS but note the directory names created will be different and by default the server will already be subscribed to the ‘extras’ channel)
