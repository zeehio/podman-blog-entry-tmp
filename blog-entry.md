# Additional groups in your containers: Easier with Podman 4.7

> This article explains how to map an additional group to your containers,
> a feature much easier to achieve with Podman 4.7 and above. [Sergio Oller](https://github.com/zeehio)
> was invited to write it after his pull request [#18713](https://github.com/containers/podman/pull/18713)
> got merged.

## Summary

Rootless containers are becoming easier and easier to use. Starting with Podman
4.7, it is easier to map additional groups to your containers so you can conveniently
access shared files. This is in short how you can do it:

1. Subordinate the group ID you would like to mount to your user:

        # Find out your user name:
        $ whoami
        alice
        # Find the group ID our user belongs to:
        $ getent group researchers
        researchers:x:2000:alice
        # It's "2000", and alice belongs to it.
        # Subordinate it to your user:
        $ sudo usermod --add-subgids 2000-2000 alice

2. Update your rootless namespace:

        $ podman system migrate

3. Run your container:

    With Podman 4.7 and above:

        $ podman run \
            --rm \
            --group-add keep-groups \
            --gidmap="+g102000:@2000" \
            --volume "/mnt/data:/data" \
            alpine ls -lisa /data

    With earlier Podman versions:

        # Figure out the corresponding intermediate mapping with:
        $ podman unshare cat /proc/self/gid_map
        # Interpret the output... (keep reading!)
        # Assuming the intermediate mapping is GID 1, then:
        $ podman run \
            --rm \
            --group-add keep-groups \
            --uidmap "0:0:65535"  \
            --gidmap "0:0:1" \
            --gidmap "1:2:65535" \
            --gidmap "102000:1:1" \
            --volume "/mnt/data:/data" \
            alpine ls -lisa /data


Files group-owned by the `researchers` group will appear inside the container as
belonging to group ID `102000`.

The Step 3 above is simpler on Podman version 4.7 and above. Earlier it required
that the user understood mappings in rootless Podman. Feel free to keep reading
if you want an explanation for all of that.

Hopefully you find the new syntax easier to use and one more reason for updating Podman!

## Motivation: Problems with rootful containers

Container technologies have grown a lot during the last decade. Cloud
providers, application developers, software engineers, data scientists and
many others use them to distribute and run all kinds of software.

While doing research in academia, I had the opportunity to work with many brilliant
and motivated scientists. Undergraduate, master students, PhD researchers, postdocs...
We all had to write and run our own code, and sometimes download and run some niche
software implementing some method related to a relevant paper. Fortunately for us, we had access to
High Performance Computing (HPC) servers for our calculations, and we had access to
fantastic tools such as [JupyterHub](https://jupyter.org/hub), [RStudio Server](https://posit.co/products/open-source/rstudio-server/)
and the [Jupyter Docker stack](https://jupyter-docker-stacks.readthedocs.io/en/latest/) and [rocker](https://rocker-project.org/) images, running under Docker to isolate our dependencies.

Being one of the administrators of the HPC was really fun, but it also was a
significant overhead over our own research interests. Users needed the flexibility
that containers provide, so we trusted them with access to our Docker. This meant
that they were given access to the `docker` group and got equivalent 
**root permissions**. We never had an issue because of that,
but it always caused some **concern** to all of us, and it was a responsibility they had
to bear even if they did not want to.

Their usage of Docker also came with permission issues. While mounting directories
in their containers, they had **file ownership issues**. A forgotten [`--user`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#user-u-user-group) flag
somewhere and they ended up with files owned by root that they did not know how
to get back under their control.

These were two major pain points of using rootful Docker.

Wouldn't it be great if we didn't have to give root-equivalent permissions to our users
and they did not suffer those file permissions? Well, **rootless containers** solve this!

## Rootless containers

When running rootless containers, the container engine runs as the current user, without
any additional permissions. No root powers are given on the host, although root-like
permissions are available inside the container. I first heard about rootless containers
through the [rootless Podman tutorial](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md), however Docker supports running in [rootless mode](https://docs.docker.com/engine/security/rootless/) as well.

With rootless Podman, each user has its own list of images and list of containers
running. All their images, container layers, etc, is stored under their home
directory (instead of under `/var/lib/`). When a container is started in rootless mode, 
it feels like being root. `apt`, `dnf`, or any package manager works, so it is possible
to install system libraries. And it all happens without getting any additional permissions
in the host. Files created as root inside the container belong to the user who started
the container in the host. It feels like magic.

As a scientist myself I wonder how things work, and rootless Podman is no exception.
One question I wondered was related to "emulating multiple users": When I run a rootless
container, I can create files in the container as root, that are mine in the host. I
could create files as other users in the container as well, so somehow they had to belong
to some other user in the host. I am just one user in the host and rootless Podman
has "no extra permissions". How is that possible?

More importantly, one requirement for rootless Podman to work for us was the ability
to mount the shared data directory that contained our shared projects. All researchers belong
to the `researchers` group, and we needed rootless containers to access our `/mnt/data`
directory in the host, with all our shared data. Even with `--volume /mnt/data:/data` and
the `--group-add keep-groups` option, we were not able to access the files, since
they appeared to be owned by `nobody` inside the container. We learnt the user namespace
had to be set up in some way. [We](https://github.com/rocker-org/rocker-versioned2/pull/636) [were](https://github.com/rocker-org/rocker-versioned2/issues/251) [not](https://github.com/rocker-org/rocker-versioned2/issues/346) [alone](https://github.com/NERSC/podman-hpc/issues/68).

## How does Rootless Podman lets the user act as multiple users?

Podman does that through a Linux kernel feature called "subordinate ids". When
we create a user we assign to it a user ID (UID). On modern Linux distributions
prepared for rootless containers, besides the UID we also assign 65536 additional
subordinate UIDs (see [man 8 useradd](https://man7.org/linux/man-pages/man8/useradd.8.html)). For instance,
my UID is 1000, and I also have subordinated the range of UIDs 100000-165535.

Find your subordinated user ids with:

    getsubids $USER

Or, if your shadow version is older than 4.10 and you don't have the `getsubids` command, you
can use:

    grep $USER /etc/subuid

You will get your user name, the initial subordinated UID and the length of the subordinated
id range. If you don't have additional subuids, you will have to enable user namespaces.
Follow the [rootless tutorial](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)
in the Podman documentation if you have issues.

The Linux kernel lets users map their own UID and their subordinated IDs in "user namespaces". The
user can impersonate those subordinated IDs as if they were their own. 
*With the right commands, I can run things as if I was UID 100000*. Needless
to say, the default assigned subordinated UIDs are not UIDs from other users,
and subordinated UID ranges do not overlap between users by default, so it is still
possible to trace who does what in the system.

A user namespace is a mapping of user-controlled host UIDs to arbitrary UIDs in the namespace.
By mapping UID 1000 (me) to number 0, files created as root (UID 0) in the namespace will appear
as files owned by me in the host. Mapping UID 100000 (one of my subordinated UIDs) to
the UID 1001, when a file is owned by 1001 in the container it appears as owned by 100000 at
the host. It was impossible for me to map UIDs I didn't control. For instance, I could not
map host UID 1001 (someone else) to any value in the container, since it is not me and
it has not been subordinated to me.

When a process runs "inside a user namespace", it sees the mapped UIDs, instead of the
real ones. And any file owned by an unmapped user will appear as owned by the
"overflow user ID" (usually `65534`). Use `cat /proc/sys/kernel/overflowuid` to
figure the overflow UID in your system if you like. You can learn more about user
namespaces at `man 7 user_namespaces`.

The kernel offers the possibility to nest user namespaces, so mappings can
be chained if desired. For instance, a first mapping maps host UID 100000 to an
intermediate mapping value of 1, and a second mapping maps 1 to 1000. A process
running at the second namespace will see UID 1000, that will correspond to
UID 100000 at the host.

That's how the magic can work.

## How does rootless Podman map users?

Reading the [--uidmap documentation](https://docs.podman.io/en/latest/markdown/podman-run.1.html#uidmap-flags-container-uid-from-uid-amount) helps to understand how Podman uses the namespaces:

Podman creates two nested user namespaces, and the user can only customize the second one with the `--uidmap` and `--gidmap` arguments.

Since each user has a different set of subordinated IDs, the first mapping (called "intermediate") just
standardizes all the available IDs from 0 to n, so further mapping commands are user-independent. The
UID of the host user becomes 0 at the intermediate mapping, and subordinated IDs (such as 100000) get mapped to
1:n.

The second mapping is up to the user to customize.

## And what about rootless Podman mapping groups?

The same ideas that we learnt for users apply to groups as well. There are subordinated group ids,
that you can list with `getsubids -g $USER` or `grep $USER /etc/subgid`.

When a user is created on a modern distribution, subordinated UIDs **and subordinated GIDs** are assigned to it.
Since by default the number of UIDs and GIDs subordinated is the same, it is very common that
they actually have the same values. Podman acknowledges that, and if a `--uidmap` is given
without the corresponding `--gidmap` (or viceversa), Podman will assume the same mapping applies to both.

The main difference groups have with respect to users, is that a user not only belongs
to a main GID but also to a list of additional GIDs. The Linux kernel gives you full
control over your main GID, so you can freely map it. However it does not give you
control over all your additional GIDs, so by default those additional GIDs can't
be mapped.

## So how can we map an additional group with Podman 4.7 and above?

You will need to subordinate your additional group to your user. Here is
an example, subordinating the group `researchers` to `alice`:

    # Find the group ID our user belongs to:
    $ getent group researchers
    researchers:x:2000:alice
    # It's "2000". Subordinate it to alice:
    $ sudo usermod --add-subgids 2000-2000 alice

Since the subordinated ids have changed, tell Podman to recreate the intermediate
user namespace:

    $ podman system migrate

Then you can create your container:

    $ podman run \
        --rm \
        --group-add keep-groups \
        --gidmap="+g102000:@2000" \
        --volume "/mnt/data:/data" \
        alpine ls -lisa /data

The process running in the container should keep your current groups `--group-add keep-groups`.

You can provide a group mapping with `--gidmap`, so the `researchers` host group (`2000`) is mapped to a known GID (`102000` in the example above) in the container. See how the `--gidmap` argument contains some `flags` (`+g`) and an operator `@`, new of this Podman release. Before these operators were available, specifying such mappings was more complicated.

## How can we map an additional group with previous Podman versions?

The process is a bit more convoluted, but still possible. I wrote these instructions
for the [rocker rootless tutorial](https://rocker-project.org/use/rootless.html), at
the rocker project, but I guess it is worth that they reach a wider audience here.

First subordinate the group to your user and update the intermediate user namespace as
described above.

Then, we have to write the user mappings. Without the `@` operator, we can't refer to the host GID directly, so we must find the GID in the intermediate user namespace corresponding to host GID `2000` with:

```{.sh}
podman unshare cat /proc/self/gid_map
#          0       1000          1
#          1       2000          1
#          2     100000      65536
```

The table shows that GID 2000 in the host (middle column) is mapped to
intermediate GID 1 (left column).

Then, without the `+` flag, we must provide the whole GID map, and not just the
extra piece we need:

| Type  | Container ID | Intermediate ID | Reason                                                                               |
| ----- | ------------ | --------------- | ------------------------------------------------------------------------------------ |
| Group |      0       |       0         | Identity mapping (our main host GID, was mapped to intermediate 0)                   |
| Group |   102000     |       1         | We map container GID 102000 to intermediate GID 1, that we saw matches host GID 2000 |
| Group |  1 - 65534   |   2 - 65535     | The rest of intermediate GIDs (excluding 0 and 1) need to be mapped 1-n              |

This will become `--gidmap "0:0:1" --gidmap "1:2:65535" --gidmap "102000:1:1"`.

But there is more! By default, if the `--gidmap` is given without a `--uidmap`, Podman assumes that the user wants to use the same IDs for `--uidmap` than for `--gidmap`, so we must specify a user ID mapping as well to avoid the copy. We will map:

| Type  | Container ID | Intermediate ID | Reason                                                                               |
| ----- | ------------ | --------------- | ------------------------------------------------------------------------------------ |
| User  |  0 - 65534   |   0 - 65534     | Identity mapping (no change needed in user mapping besides the default one)          |

This will become `--uidmap "0:0:65535"`.

On the instructions above, the `"+g102000:@2000"` GID mapping was copied to the UID mapping as well, but the `g` flag
we used signaled the UID mapping that the mapping didn't apply to UIDs.

Putting all together, the command becomes:

    $ podman run \
        --rm \
        --group-add keep-groups \
        --uidmap "0:0:65535"  \
        --gidmap "0:0:1" \
        --gidmap "1:2:65535" \
        --gidmap "102000:1:1" \
        --volume "/mnt/data:/data" \
        alpine ls -lisa /data

## Final words

The extended syntax for mappings found on Podman 4.7 allows you to provide a
more expressive and simple mappings, hopefully easier to use.

A big thank you to [@giuseppe](https://github.com/giuseppe), [@Luap99](https://github.com/Luap99),
[@TomSweeneyRedHat](https://github.com/TomSweeneyRedHat) and [@rhatdan](https://github.com/rhatdan), for 
their support, feedback and time reviewing this work. And to RedHat for funding them
to do it.

I recently switched from academia to industry (and I am quite happy so far!), so
if you are at academia and feel this change useful, consider this to be my
"goodbye gift" to you and your team. Looking forward to your discoveries!

