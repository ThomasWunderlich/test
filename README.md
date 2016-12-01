# Migrating from MREPO to REPOSYNC
Kent C. Brodie

## BACKGROUND
We are a RedHat shop (in my case, many CentOS servers, and some RedHat as well).   To support the system updates around all of that, I currently use MREPO, an open source repository mirroring tool created by Dag Wieers.   MREPO is an excellent  YUM repository manager that has the ability to house, manage, and mirror multiple repositories.   Sadly for many, MREPO’s days are numbered.   Today I’m going to cover both how to migrate from MREPO to REPOSYNC, and WHY.  REPOSYNC is a simpler alternative that is built into RedHat and CentOS systems.

For me,  MREPO has thus far done the job well.   It allows you to set up and synchronize multiple repositories all on the same single server.   In my case, I have been mirroring RedHat 6/7, and Centos 6/7 and it has always worked great.  I’ve had this setup for years, dating back to RedHat 5.

While mirroring CentOS with MREPO is fairly trivial, mirroring RedHat updates requires a little extra magic:  MREPO uses a clever “registration” process to register a system to RedHat’s RHN (Red Hat Network) service, so that the fake “registered server” can get updates.

Let’s say you have MREPO and wanted to set up a RedHat 6 repository.  The key part of this process uses the “gensystemid” command, something like this:

```gensystemid -u RHN_username -p RHN_password --release=6Server --arch=x86_64 /srv/mrepo/src/6Server-x86_64/```

This command actually logs into RedHat’s RHN, and “registers” the server with RHN.  And now that this fake-server component of MREPO is now allowed to access RedHat’s updates, it can begin mirroring the repository.  If you log into RedHat’s RHN, you will see a “registered server” that looks something like this:

![Redhat RHN registered server screen](https://github.com/ThomasWunderlich/test/blob/master/registeredserver.png "Redhat RHN registered server screen")

## IF IT AIN’T BROKE, DON’T FIX IT, RIGHT?

So what’s the issue?  For you RedHat customers, if you’re still using RHN in any capacity, you hopefully have also seen this notice by now:

Putting this all together:  If you’re using MREPO to get updates for RedHat servers, that process is going to totally break in just over 7 months.  **MREPO’s functionality for RedHat updates depends on RedHat’s RHN, which goes away July 31st.**

Finally, while MREPO is still used widely, it is worth noting that it appears continued development of MREPO ceased over four years ago.    There have been a scattering of forum posts out there that mention trying to get MREPO to work with RedHat’s new subscription-management facility, but I never found a documented solution that works.

## WHAT IS REPOSYNC?

Reposync is a command-line utility that’s included with RedHat-derived systems as part of the yum-utils RPM package.   The beauty of reposync is its simplicity.  At the core, an execution of reposync will examine all of the repositories that the system you’re running it on has available, and downloads all of the included packages to local disk.  Technically, reposync has no configuration.  You run it, and then it downloads stuff.  MREPO on the other hand, requires a bit of configuration and customization per repository.

# HOW DOES REPOSYNC SOLVE THE PROBLEM OF RHN DISAPPEARING?
You simply have to think about the setup differently.   In our old model, we had one server that acted as the master repository for all things, whether it was RedHat 6, CentOS 7, whatever.  This one system was “registered”, multiple times, to mirror RPMS for multiple operating system variants and versions.

In the new model, we have to divide things up.   You will need one dedicated server per operating system version.   This is because any given server can only download RPMs specific to the operating system version that server is running.  (Fortunately with today’s world of hosting virtual machines, this isn’t an awful setup, it’s actually quite elegant).    In my case, I needed a dedicated server for each of:   RedHat 6, RedHat 7, CentOS 6, and CentOS 7.

For the RedHat servers, the elegant part of this solution deals with the fact that you no longer need to use “fake” system registration tools (aka gensystemid).  You simply register each of the repository servers using RedHat’s preferred system registration:   the “subscription-manager register” command that RedHat provides (with the retirement of RHN coming, the older rhn_register command is going bye-bye).   MREPO, at present, does not really have a way to do this using RedHat’s “new” registration mechanism.

## SIMPLE REPOSYNC EXAMPLE
The best way for you to see how reposync works is to try it out.   For this example, I highly recommend starting with a fresh new server.    Because I want to show the changes that occur with subscribing the server to extra channel(s), I am using RedHat 6.     (You are welcome to use CentOS but note the directory names created will be different and by default the server will already be subscribed to the ‘extras’ channel)

For my example, perform the following steps to set up a basic reposync environment:  
*	Install a new server with RedHat6.   A “Basic” install is best.
*	Register the server with RedHat via the subscription-manager command.   
*	Do NOT yet add this server to any other RedHat channels
*	Do NOT yet install any extra repositories like EPEL.
*	Install the following packages via YUM:     yum-utils, httpd
*	Remove /etc/httpd/conf.d/welcome.conf (The repository will not have an index web page, so by removing this, you’re not redirected to a default apache error document)
*	Ensure the system’s firewall is set so that you can reach this server via a web browser

The simplest form of the reposync command will download all packages from all channels your system is subscribed to, and place them in a directory of your choosing.

The following command will download thousands of packages and build a full local RedHat repository, including updates:

```/usr/bin/reposync  --download_path=/var/www/html```

The resulting directory structure will look like this:

```
/var/www/html
`-- rhel-6-server-rpms
    `-- Packages
```

If you point your web browser to http://repohost/rhel-6-server-rpms/Packages, you should see all of your packages.    

Use RedHat’s management portal to add this same server to RedHat’s “Optional packages” channel.  For my example, I also installed the EPEL repository to my yum environment  (reference: https://fedoraproject.org/wiki/EPEL).  

With the server now ‘subscribed’ to more stuff (RedHat’s optional channel and EPEL), a subsequent reposync command like performed above now generates the following:

```
/var/www/html
|-- epel
|-- rhel-6-server-optional-rpms
|   `-- Packages
`-- rhel-6-server-rpms
    `-- Packages
```

Note: EPEL is a simpler repo, it does not use a “Packages” subdirectory.

Hopefully this all makes sense now.  The reposync command examines what repositories your server belongs to, and downloads them.    **Reminder**:  you need one ‘reposync’ server for each major operating system version you have, because each server can only download RPMs and updates specific to the version of the operating system the server is running.

# WHAT’S NEXT?
One more step, actually.  A repository directory full of RPM files is only the first of two pieces.  The second is the metadata.  The repository metadata is set up using the “createrepo” command.  The output of this will be a “repodata” subdirectory containing critical files YUM requires to sort things out when installing packages.

Using a simple example and our first repository from above, let’s create the metadata:

```/usr/bin/createrepo /var/www/html/rhel-6-server-rpms/Packages```

After which our directory structure now looks like this:

```
/var/www/html
|-- epel
|-- rhel-6-server-optional-rpms
|   `-- Packages
`-- rhel-6-server-rpms
    `-- Packages
        `-- repodata
```

You will need to repeat the createrepo command for each of the individual repositories you have.   Each time you use reposync, it should be followed by a subsequent execution of createrepo.   The final step in all of this to keep current is the addition of cron job entries that usually run reposync and createrepo every night.

## THINGS I LEARNED
Both reposync and createrepo have several command options.   Here are some key options that I found useful and explanations as to when or why to use them.

### REPOSYNC

*--download-metadata*

This downloads not only the RPMS, but also extra metadata that may be useful, most important of which is an xml file that contains version information as relates to updates  This totally depends on the particular repository you’re syncing.

*--downloadcomps*

Also download the comps.xml file.  The comps.xml file is critical to deal with “grouping” of packages (example “yum groupinstall Development-tools” will not function unless the repository has that file).

*--newest-only*

Only download the latest versions of each RPM.    This may or may not be useful, depending on whether you only want the absolute newest of everything, or whether you want ALL versions of everything.

### CREATEREPO

*--groupfile*

If you have a comps.xml file for your repository, you need to tell createrepo exactly where it is.   

*--workers  N*

The number of worker processes to use.   This is super handy for repositories that have thousands and thousands of packages.     It speeds up the createrepo process significantly.

*--update*

Do an “update” versus a full new repo.   This drastically cuts down on the I/O needed to create the final resulting metadata.

## SUMMARY
The main point of this SysAdvent article was to help those using MREPO today to wrap their head around REPOSYNC, and (no thanks to RedHat), why you NEED to move away from MREPO to something else like REPOSYNC if you’re an RHN user.  My goal was to provide some simple examples and to provide understanding how it works.

If you do not actually have official RedHat servers (for example, you only have CentOS etc.), you may be able to keep using MREPO for quite some time, despite that the tool has not had any active development in years.   Clearly, a large part of MREPO’s functionality will break after 7/31/2017.  Regardless of whether you’re using RedHat or CentOS, REPOSYNC is in my opinion an excellent and really simple alternative to MREPO.  The only downside is you need multiple servers (one for each OS version), but virtualization helps keep that down to a minimal expense.
