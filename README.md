This is a tool to ensure that users exist and can log in to a system.

About the tools
===============


outstanding-policies
--------------------
Looks through a declaration and sees if anything needs to be done. If
anything does need to be done, echos one or more commands to execute
to bring the system slightly more in compliance.

Running those scripts and then re-running `outstanding-policies` will
therefore, bit by bit, get your system into the described state.


iterate-over-policies
---------------------
This is a simple tool which
- runs outstanding-policies over a declaration
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


