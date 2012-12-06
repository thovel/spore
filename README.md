This is a set of tools to make it possible to ensure that users exist
and can log in to a system, using ssh keys and are in the correct
groups, with varying sets of authorizations in different
environments.

About Spores
============

The word Spore is inspired with the reproductive system of Fungus.
The inspiration is in-part caused by the fact that spores can survive
in harsh environments, but still be able to spring to life.  This
software product aims to seed a piece of software, e.g. an operating
system or http server with authentication information to authorized
parties.  In essence, using these tools, a system targeted by spore
will have the user definitions, SSH keys or similar that are
appropriate for that system, as defined by the administrator.

What is a spore?
----------------

In this software tool, a spore is one or more collections of user
definitions:


    person.ini:

    [name]
    full = John Doe

    [user]
    name = jdoe

That's a "minimal" spore which defines a user `jdoe` for the person
John Doe.  A spore can contain SSH keys (in the `ssh-keys` directory)
and other information too.

A spore seldom lives alone, and several person.ini files typically
live in sibling directories:

    jdoe/person.ini
    susan/person.ini

Collections of spores like this are referred to as a "directory of
spores," or just plural "spores," or just a "directory."

A spore may be applied to a system, which results in user accounts
being created, and SSH keys being added. However, it is also possible
to /disperse/ spores for more fine grained authorization.


Dispersing spores
-----------------

Spores on themselves can be used as-is to create the users on the
machines in question, but often, certain users are supposed to have
certaion groups on certain types of machines.  This is where
dispersing comes in.

A conf-file (yes, an .ini file format) defines how spores are
dispersed.  This file describes what roles a user has, and what
permissions are assigned to each role.

    disperse.conf:
    [users]
      jdoe = operator
      susan = user

    [permissions]
      operator = shell access to production
      operator = become root on production
      user = shell access to production

(Note: The syntax is very new, and is in a state of flux; don't count
on it not changing!  Particular, assigning groups directly might well
happen.)

In `[users]` the keys represent directory paths, so e.g. `jdoe`
refers to `/users/jdoe/person.ini`.

After "dispersing" to `production` you will get two spores defining
the users jdoe and susan, but with additional information about what
groups the users should get.  Specifically, jdoe, being an `operator`
gets to be in the `adm` group, which has sudo privileges.


About the tools
===============

There are three tools.  `spore`, `spore-disperse` and `spore-download-and-apply`.

`spore`
-------

The spore command accepts a command and a directory:

    spore <command> <directory>

example invocations:

    spore next /my/spores | sudo tee -a /var/log/spore-commands | sudo sh
    sudo spore apply /my/spores

Spore takes the definitions in the directory (see Declaration, below)
and

 * creates users and groups defined therein
 * adds users to groups
 * rolls out and revokes ssh authorized keys


`spore next`
------------
Looks through a declaration and sees if anything needs to be done. If
anything does need to be done, echos one or more commands to execute
to bring the system slightly more in compliance.

Running those scripts and then re-running `spore next` will
therefore, bit by bit, get your system into the described state.


`spore apply`
-------------
This is a simple tool which
- runs `spore next` over a declaration
- executes any commands needed to bring the system into compliance
- repeats until nothing is outstanding

Running this command will therefore bring your system into compliance
with the declaration.


`spore-disperse`
----------------

Spore-disperse rehashes the original spores it receives, based on the
definitions passed in on the command line.  The result is a new set
of spores which more closely fits a specific use case.

The point of dispersing is to allow a common and large directory of
user information, e.g. for a whole company or organization, while
at the same time providing spore information for different subsets
of users different access levels in different systems.

The "root" spore would be the maintained user directory, and for each
substantial subset of systems that users need access to, the
directory would be dispersed into smaller spores, with different and
perhaps overlapping subsets of the full user directory.  Users with
administrative rights in one spore might have little or no rights in
a different spore.


`spore-download-and-apply`
--------------------------

The command `spore-download-and-apply` is designed to run by cron,
and is a wrapper around `spore apply` which additionally:

 * downloads a tarball of spores from a configured URI
 * verifies that spores have been signed by a specific gpg key
 * performs consistency checks to ensure that the system hasn't
   changed in ways that spore doesn't like

### Conf file

The command requires a simple .conf file called
`/etc/spore-cronjob.conf` which specifies where and how to get spores
and what gpg key to verify.


    # The URI of a .tar.gz file containing spores
    spore_uri=http://some/ur/of/a/spore.tar.gz

    # The URI of the signature, defaults to "$spore_uri.asc"
    # spore_signature_uri=http://some/ur/of/a/spore.asc

    # If the spore or signature are protected by password, specify them here
    # http_user=someuser
    # http_passwd=somepasswd
    # http_proxy=someproxy:3128
    # https_proxy=someproxy:3128

    # GPG ID of the trusted signatures;
    spore_signee=123456ABCDEF

The conf-file is a bash script, and will be sourced, so it may
contain shell expressions.

### Operation

The operations always run in this order:

1. verify existing spores  (-v)
2. consistency check       (-c)
3. download new spores     (-d)
4. verify new spores       (-v)
5. apply new spores        (-a)

If no options are passed, all steps are run, which makes it possible
to install it as a symlink to e.g. `/etc/cron.hourly/`.  If it is
desirable to do this more often, then it's possible to write crontab
files that invoke the desired functionality:

    # Calculate the current state of spores every 5 minutes to update
    # /etc/motd.tail
    */5 * * * * /usr/sbin/spore-download-and-apply -c > /etc/motd.tail

### Verification step

Verification has to do with ensuring that the spores that are being
applied originate from a trusted source. The insecure nature of
downloading content over http or https requires an additional layer
of security.  This is done by verifying that the spore .tar.gz has
been cryptographically signed by a specific gpg key.

Each .tar.gz is paired with a .tar.gz.asc file (by default in the
same location, or in another location, if the .conf file specifies
one); and the .tar.gz /must/ have a valid signature for spore to
complete the verification step.

If the verification step cannot be completed, the command exits
immediately.

Verification is disabled if there is no local spore file.

If the conf file specifies a long key ID, the tarball needs to be
signed by that specific key, not just any key in root's keychain.

For any verification to occur, the signer's public key needs to be
added to root's keychain:

    # curl -s http://location/of/my-key.asc | gpg --import
    gpg: key ABCDEF12: "Spore Signer (spore-signer@mycompany.com)" imported

To get the long key ID:

    # gpg --with-colons --list-key ABCDEF12 | grep ^pub | cut -f5 -d:
    09876543ABCDEF12

The long key identifier (16 characters) uniquely identifies the key
on the keychain, and can be used to ensure that spores aren't
accepted unless they're signed by that specific key, regardless of
what public keys are trusted in root's keychain.

### Consistency check

The consistency check phase ensures that the current spore file
correctly describes the current state of the system.  If the system
differs from the spores in such a way that "spore next" says that
something needs to be done, the consistency check fails.

When the consistency check fails, the command exits immediately.

Consistency checks are disabled if there id no local spore file.

### Download new spores

Downloading new spore files is done conditionally. If the spore
files are not changed on the servers, no new files are downloaded,
and the command exits. The next phase only happens if new spores
arrive.

### Applying spores

The apply phase of the `spore-download-and-apply` command does just
this: it unpacks the tarball to a temporary directory and runs
`spore apply` on it, and cleans up after itself.


How to use it
=============

1. Maintain a central, well controlled repository of spores, with
   user information and public keys
2. Maintain a few "dispersion" configration files for each of the
   major systems involved
3. Run the central spores through the dispersions to create per-
   system spores
4. Package and sign these spores and deliver them to the machines
   in question
5. Finally run `spore apply` on the delivered spores after the
   signature has been verified.
6. Your users will have access to the system.


Declaration
===========

The declaration is a directory structure, with the following structure:

- At the root level, a directory called `users` where user definitions
  reside
- within `users` each user has its own directory
- within each user directory, a `person.ini` file must be present
- `person.ini` must contain a `[name]` section, with `full` key which
  specifies the user's full name e.g `John Doe`
- `person.ini` must contain a `[user]` section, with `name` key which
  specifies the user's unix name, e.g. `jdoe`
- within each user directory, a directory `ssh-keys` should be present
- within the `ssh-keys` directory, any number of public keys may
  be present, either in `.pub` files or in `.blacklisted` files.
- each ssh key file is a standard `id_rsa.pub` or `id_dsa.pub` file


Examples
========

The examples directory has a simple example; here I show what happens
when applying that declaration.  Have the `simple-declaration`
directory handy if you want to try this out on your own:

First, an overview of the declaration:

    $ find simple-declaration -type f
    simple-declaration/users/johndoe@example.com/person.ini
    simple-declaration/users/johndoe@example.com/dotfiles/hgrc
    simple-declaration/users/johndoe@example.com/dotfiles/gitconfig
    simple-declaration/users/johndoe@example.com/ssh-keys/home.pub
    simple-declaration/users/johndoe@example.com/ssh-keys/laptop-stolen-in-2011.blacklisted
    simple-declaration/users/johndoe@example.com/ssh-keys/work.pub
    simple-declaration/users/johndoe@example.com/ssh-keys/laptop.pub

To see if your system is complying with the policy, run

    $ spore next examples/simple-declaration
    useradd --create-home  --comment 'John Doe' 'jdoe'

It suggests that you should run `useradd` to create the user

    $ spore next examples/simple-declaration | sudo sh

It doesn't output anything...  Now see if we comply:

    $ spore next examples/simple-declaration
    mkdir --mode 700 '/home/jdoe/.ssh'
    cp 'examples/simple-declaration/users/johndoe@example.com/dotfiles/hgrc' '/home/jdoe/.hgrc'
    chown 'jdoe:' '/home/jdoe/.hgrc'
    chmod '600' '/home/jdoe/.hgrc'

Now, it suggests to create a .ssh directory in the user's home
directory and place a .hgrc file there.

    $ spore next examples/simple-declaration
    chown jdoe: '/home/jdoe/.ssh'
    cp 'examples/simple-declaration/users/johndoe@example.com/dotfiles/gitconfig' '/home/jdoe/.gitconfig'
    chown 'jdoe:' '/home/jdoe/.gitconfig'
    chmod '600' '/home/jdoe/.gitconfig'

Next it changes the owner of the directory, and adds another dotfile.

    $ spore next examples/simple-declaration
    echo 'ssh-dss AAAAB3NzaC== john@home' | tee >/dev/null -a
      '/home/jdoe/.ssh/authorized_keys'

... add an SSH key (the line has been split to make this readme more
readable).

This continues for another two keys until the `spore next`
command returns nothing (or just comments), and your system
"complies" with the declaration you specified.

The `spore apply` command merely does what you just did,
namely repeat calling `spore next` for as long as it takes,
faithfully executing each command to bring the system closer and
closer to being in compliance.

If you delete the jdoe user, and his home directory:

    $ sudo spore apply examples/simple-declaration
    Complete (8 commands run).

and your system is "compliant"


