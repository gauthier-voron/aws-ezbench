AWS Small Toolset
=================

A set of scripts to benchmark broker system on AWS **ezpz**.


Spawn
-----

Spawn spot instances or normal instances and print their name on standard
output.

    $ ./spawn --image='The Benchmarkable' --key='gauthier' --size=13 \
              --type=c5.2xlarge --region=eu-central-1 >> ids.conf

This spawns a new fleet of 13 instances and print the fleet id on stdout.
The instances are spot instances which are cheaper but may be reclaimed by
Amazon if ran for too long.
To reserve regular instances, use `--reserved` flag.


Terminate
---------

Terminate one or many instances from standard input.

    $ ./terminate < ids.conf

This terminates every spot or regular instances listed in `ids.conf`.


Run
---

Run ssh or scp commands to instances listend on standard input.

    $ ./run -- ls < ids.conf
    [ 3.73.55.64      ] . .. .bashrc
    [ 18.198.23.80    ] . .. .bashrc
    [ 18.195.50.191   ] . .. .bashrc

This runs `ls` via ssh on every instances listed in 'ids.conf'.

    $ ./run --scp -- local-file :remote-file < ids.conf

This copies 'local-file' to eah instance as 'remote-file' (remote files start
with a ':' character).

You can also do parameter substitution to send different orders to different
instances.

    $ ./run -- echo %{ip} < ids.conf
    [ 3.73.55.64      ] 3.73.55.64
    [ 18.198.23.80    ] 18.198.23.80
    [ 18.195.50.191   ] 18.195.50.191

The parameter substitution happens for characters enclosed in an `%{}` canvas.
The group of characters is called the "name" and is replaced by a "value".
If there is no value associated with a name then no substitution happens.
Every instances have default values for the names:

  - ip : the public ip of the instance
  - id : a unique identifier
  - region : the AWS region where the instance runs.

You can add values for new names, or override existing names with a parameter
file.

    $ ./run -- echo %{region} %{id} < ids.conf
    [ 3.73.55.64      ] eu-central-1 sfr-1b3ba1a4-66d8-4260-88c7-a91cc08dfde[0]
    [ 18.198.23.80    ] eu-central-1 sfr-1b3ba1a4-66d8-4260-88c7-a91cc08dfde[2]
    [ 18.195.50.191   ] us-west-1 i-0f8aa75ddf1d9c0fd
    [ 3.72.46.253     ] eu-central-1 sfr-1b3ba1a4-66d8-4260-88c7-a91cc08dfde[1]
    [ 54.93.223.113   ] us-west-1 i-0a837b2cfcc021b7c
    $
    $ cat > params.conf <<EOF
    [name-0]                                         # assign values to 'my-name-0'
    i-0a837b2cfcc021b7c = foo                          # assign to regular instance
    sfr-1b3ba1a4-66d8-4260-88c7-a91cc08dfde[2] = bar   # or to a spot instance
    
    # Assignements can also be done for all or some spot instances of the same
    # fleet.
    #
    [name-1]
    sfr-1b3ba1a4-66d8-4260-88c7-a91cc08dfde = quux       # assign to all fleet
    sfr-1b3ba1a4-66d8-4260-88c7-a91cc08dfde[0..1] = foo  # override for [0] and [1]
    
    # It can also be done for an entire region or even for everyone.
    #
    [name-2]
    * = default
    us-west-1 = us
    EOF
    $
    $ ./run --dispatch=params.conf -- echo %{name-0} %{name-1} %{name-2} < ids.    conf
    [ 3.73.55.64      ] %{name-0} foo default
    [ 18.198.23.80    ] bar quux default
    [ 18.195.50.191   ] %{name-0} %{name-1} us
    [ 3.72.46.253     ] %{name-0} foo default
    [ 54.93.223.113   ] foo %{name-1} us

Parameter substitution is especially convenient when fetching files from many
instances.

    $ ./run --scp -- :remote-file local-file.%{ip}

Parameters can also be used to select a subset of instances to run a command.

    $ ./run --filter="region=us-west-1" -- uname -a
    $ ./run --filter="region!=ap-northeast-1" -- uname -a
    $ ./run --filter="numeric-value>17" -- uname -a
    $ ./run --filter="some-value=~^perl-style.*regexp\s\d+" -- uname -a
    $ ./run --filter="must-be-defined" -- uname -a

The operators are `==` and `!=` for string comparison, `<`, `<=`, `>`, `>=` for
numeric comparison, `=~` and `!~` for regular expression matching.
If only the name is specified, the filter selects only the instances for which
this name is defined.
If the name is prefixed by a `!`, it selects only the instances for which the
name is undefined.


Benchmark
---------

Do a full benchmark of the broker system.

    $ ./benchmark ids.conf params.conf hotstuff 400 8 300

This uses the instances listed in `ids.conf` with the roles and identifiers
listed in `params.conf` to test the broker system with `hotstuff` as total
order broadcast with a consensus batching of `400` batch per consensus, `8`
workers per hardware broker and each broker worker sending `300` batches.

There are a few requireements for this command to work:

  - All instances must have been launched with the image `The Benchmarkable`.

  - All instanves must have at least `16 GiB` or memory.

  - There must be 1 instance with the parameter `role` set to `rendezvous`.

  - There must be at least 4 instances with the parameter `role` set to
    `server`.

  - There must be at least 1 instance with the parameter `role` set to
    `broker`.

  - All `server` instances must have a parameter `id` being an integer between
    `0` and the number or server minus 1.

  - All `broker` instances must have a parameter `id` being an integer between
    `0` and the number or server minus 1.

Here is an example of a valid `params.conf` file:

    [role]
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[0] = rendezvous
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[1..4] = server
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[5..8] = broker
    
    [id]
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[1] = 0
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[2] = 1
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[3] = 2
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[4] = 3
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[5] = 0
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[6] = 1
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[7] = 2
    sfr-a7c73dd2-f8f1-4bef-9e17-6061dc3b38b2[8] = 3

The available total order broadcast are `loopback`, `hotstuff` and `bftsmart`.
