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

There are two tools.  `spore` and `spore-disperse`.

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

Now, it suggests to create a .ssh directory in the user's home
directory.  The next thing...

    $ spore next examples/simple-declaration
    chown jdoe: '/home/jdoe/.ssh'

...is to change the owner of the directory, and then ...

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


