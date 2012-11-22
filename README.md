This is a tool to ensure that users exist and can log in to a system,
using ssh keys and are in the correct groups, etc.

About the tools
===============

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


spore next
----------
Looks through a declaration and sees if anything needs to be done. If
anything does need to be done, echos one or more commands to execute
to bring the system slightly more in compliance.

Running those scripts and then re-running `spore next` will
therefore, bit by bit, get your system into the described state.


spore apply
-----------
This is a simple tool which
- runs `spore next` over a declaration
- executes any commands needed to bring the system into compliance
- repeats until nothing is outstanding

Running this command will therefore bring your system into compliance
with the declaration.


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


