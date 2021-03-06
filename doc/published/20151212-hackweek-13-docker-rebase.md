title: Docker Internals and Implementing Rebase
author: Aleksa Sarai
published: 2015-12-12 22:30:00
updated: 2015-12-12 22:30:00
description: >
  [SUSE's semi-annual Hackweek](https://hackweek.suse.com/) was last week and I
  decided to work on implementing `docker rebase`, mainly to learn about the
  internal image format of Docker and see whether it was possible to improve how
  the updating of Docker images works in practice (either rebuilding or
  [zypper-docker](https://github.com/SUSE/zypper-docker)).
tags:
  - docker
  - free software
  - hackweek
  - suse

The Docker image format isn't standardised (or really documented), so my project
was going to be quite a challenge. The basic idea I had in my head was implementing
a `git rebase`-like feature for Docker. Luckily, I'd already been looking at how
`git` strategies work, since `git` is actually only a collection of a few simple
graph theory tricks.

### The `git rebase` Method ###

The way `git rebase` works, in a rough sense is like this:

For the `git rebase A B` case, find the common parent `C`. This is equivalent to
the form `git rebase --onto A C B`. We would need to make an extension to the
`git rebase` model here, namely what happens when there's no common commit. This
will be dealt with by stupidly applying layers on top of `A` starting from `C`
and then going up to `B` (obviously this means you can't use the "common parent"
version).

For each layer starting from `C`, going to `B`:

* If the layer's diff is encountered in the history of `A`, we skip that layer as
  it has already been applied.
* Else, if the layer modifies files which don't exist (or delete files that do), just
  apply it blindly -- unless it's trying to remove a file that doesn't exist which
  is obviously an error.
* Else, if the layer's file modifications are such that the diff applies cleanly,
  then just apply it to the file. This is different from the above case for Docker,
  but not in the `git` case.
* Otherwise, bail and ask the user to solve the conflict for us.

Fortunately with a Docker image, we don't need to ask the user for any help. We
(at least in principle) should have all the information necessary to generate the
entire image history. There are some caveats for the special `#(nop)` layers which
are generated by `Dockerfile` builds, which must be considered separately to
regular layers.

### Docker Images ###

Docker images are stored as a **d**irected, **a**cyclic **g**raph (DAG), just
like `git`. Each node is an image configuration file, with a `tar`d filesystem
containing the set of changes made by the layer. The on-disk format is a `tar`
archive of the above with a directory of each layer, containing a JSON representation
of the image configuration and the layer's `tar` archive. You can follow along at
home by doing the following commands:

```language-bash
% docker save -o image.tar <image>
% mkdir image/
% tar xvfC image.tar image/
120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16/
120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16/VERSION
120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16/json
120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16/layer.tar
42eed7f1bf2ac3f1610c5e616d2ab1ee9c7290234240388d6297bc0f32c34229/
42eed7f1bf2ac3f1610c5e616d2ab1ee9c7290234240388d6297bc0f32c34229/VERSION
42eed7f1bf2ac3f1610c5e616d2ab1ee9c7290234240388d6297bc0f32c34229/json
42eed7f1bf2ac3f1610c5e616d2ab1ee9c7290234240388d6297bc0f32c34229/layer.tar
511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158/
511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158/VERSION
511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158/json
511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158/layer.tar
7318595639c20360ecb151e4d51e7d2c48dd26b517c0d55ceb6d1ece80787710/
7318595639c20360ecb151e4d51e7d2c48dd26b517c0d55ceb6d1ece80787710/VERSION
7318595639c20360ecb151e4d51e7d2c48dd26b517c0d55ceb6d1ece80787710/json
7318595639c20360ecb151e4d51e7d2c48dd26b517c0d55ceb6d1ece80787710/layer.tar
7f79605b64c3f9579f4fd9a78537879c066576677e6466b1ac9cffefe1b1898e/
7f79605b64c3f9579f4fd9a78537879c066576677e6466b1ac9cffefe1b1898e/VERSION
7f79605b64c3f9579f4fd9a78537879c066576677e6466b1ac9cffefe1b1898e/json
7f79605b64c3f9579f4fd9a78537879c066576677e6466b1ac9cffefe1b1898e/layer.tar
a9eb172552348a9a49180694790b33a1097f546456d041b6e82e4d7716ddb721/
a9eb172552348a9a49180694790b33a1097f546456d041b6e82e4d7716ddb721/VERSION
a9eb172552348a9a49180694790b33a1097f546456d041b6e82e4d7716ddb721/json
a9eb172552348a9a49180694790b33a1097f546456d041b6e82e4d7716ddb721/layer.tar
c3b53a67d369fb1a8a6455ac1d348d2cddd6d85b31bacdda602013e2087c8d98/
c3b53a67d369fb1a8a6455ac1d348d2cddd6d85b31bacdda602013e2087c8d98/VERSION
c3b53a67d369fb1a8a6455ac1d348d2cddd6d85b31bacdda602013e2087c8d98/json
c3b53a67d369fb1a8a6455ac1d348d2cddd6d85b31bacdda602013e2087c8d98/layer.tar
fadc9aad1a8c5d6d7b89d1e2f1a9229d354551dd00a6bff6085225d8a43c7608/
fadc9aad1a8c5d6d7b89d1e2f1a9229d354551dd00a6bff6085225d8a43c7608/VERSION
fadc9aad1a8c5d6d7b89d1e2f1a9229d354551dd00a6bff6085225d8a43c7608/json
fadc9aad1a8c5d6d7b89d1e2f1a9229d354551dd00a6bff6085225d8a43c7608/layer.tar
repositories
%
```

`repositories` just contains a JSON object describing what tags were exported.
If you export using an image's ID then it isn't present, and it's not important
for our work.

`VERSION` describes the version of the image format being used. For Docker 1.9,
this is always `1.0`, despite the fact that the `V1Image` structure is described
as being "legacy". It appears as though the image code is currently undergoing
a migration to an updated version. The content-addressable hashes were vital in
getting Notary and the Docker content trust to operate securely. However,
`docker save` doesn't output these "new and improved" images. I'm unsure if you
could actually subvert the security of Docker here (it checks that the layer IDs
are correct).

`layer.tar` and `json` are the above mentioned layer node information. Together,
they contain the same form of information as a `git` commit (with some additional
data about how they were created, which helps us when we have to deal with rebuilding
layers). One question that quickly arises is "but how does Docker use tar files
to deal with the removal of files?". This is a very good question, and the answer
lies in the history of Docker's storage drivers.

### AUFS ###

Despite it's flaws, AUFS (or the state of union filesystems at the time) was
quite important in the history of Docker's design. The layered design of Docker
images probably wouldn't have been as easy to implement if it wasn't for UFS and
AUFS. Sure they caused kernel crashes, but they showed potential.

As it turned out, AUFS had already solved the layered filesystem representation
problem before Docker. They invented several metadata prefixes that represented
multiple kinds of files. The main one we're interested in is `.wh.`, which
indicates a "whiteout file" -- a file that was `unlink()`d in the layer. There
are a few other AUFS metadata files and directories, which we don't really care
about.

In principle they are all properly replaced during the creating of the image `tar`
file. For reference, they are the `/.wh..wh.plnk/` directory which is used for
hardlinks to reference so you can reference files in other layers (hardlinks are
based on inodes, so you can't really reference an inode number you don't know
yet). The other is the `.wh..wh..opq` file which is used to indicate that a
directory is opaque (`readdir()` calls don't descend to layers below it).

### Docker Rebase ###

Applying on top of the `git rebase` model, the following is the `docker rebase`
design. The nomenclature is the same as the `git` model. `A` is the base `B` is
being applied on top of, starting from `C`.

* If an image is a `#(nop)` image, we have no choice but to blindly apply it. If
  it modifies a file that already existed or deletes a file that didn't exist
  we have to either bail and tell the user "sorry, you have to rebuild" or plough
  through with warnings.
* Else, if the image doesn't affect any files which exist, and doesn't delete
  any files we can apply it safely without rebuilding (but there might be
  [some caveats with this](#copies).
* Otherwise, we have to rebuild the image. There is a possible improvement here
  with layers that just modify metadata (but things like `chown -R` might prove
  an issue). Rebuilding the image currently would require importing the current
  state of our new image into Docker and then doing the whole `docker run` and
  `docker commit` shenanigans. This could be improved by faking the layers using
  `runc`, but probably would be a major pain.

This model could definitely use some improvement. However, it offers a solid
improvement over `docker build`. It doesn't have the same cache semantics as
`Dockerfile`, which is an improvement if you're into hacking Docker images. In
addition, it can be a massive time saver for building applications or custom
dependencies.

#### Copies ####

Unfortunately, the `git` analogy only goes so far. While Docker layers and `git`
commits might have very similar technical aspects, Docker layers describe an
entire system's filesystem. Redundant copies of code are not considered a good
practice, so `git` doesn't need to explicitly deal with multiple copies of the
same code. Unfortunately, it is quite common to have commands which generate files
from other files. It would be ideal if we could know what files were read by a
layer (but that's not practical without adding some dodgy debugging facilities
to Docker). Thus, we must be able to support the user explicitly setting a flag
saying they want to explicitly rebuild all layers we know how to rebuild.

#### `DiffID`, `ChainID`, `DiffID`, `ChainID`, `DiffID`, ...  ####

So, while all of that design is nice, we need to deal with the nitty gritty of
implementing this. There were a couple of issues faced during graph loading,
mainly involving the fact that Docker 1.9 doesn't actually output the modern
image format, instead outputting version `1.0` legacy images. However, that's
quickly solved by manually unmarshalling the JSON into the embedded `V1Image`.

However, another issue you'll run into as soon as you start working on applying
images on different layers is what the new layer ID should be. In that past, back
when I started contributing to Docker, the layer IDs were just random hexadecimal
strings. However, they were improved to be deterministic and later to be entirely
content-addressable. This results in a bunch of weirdness with layer IDs, and
that's where our issue comes in.

If you look closely, you'll notice that the ID of an image is based on a `ChainID`,
which represents a chain of `DiffID`s (that is, an image is a set of layers in
a particular order). However, generating a `DiffID` is not straightforward. It's
possible to get a `DiffID` from a corresponding `ChainID`, but that's the only
obvious way I could find. Naturally, there must be a way that Docker bootstraps
this. They've hardcoded the `DiffID` of an empty layer, and reading the tests
carefully appears to indicate that the `DiffID` (or `ChainID`, it isn't commented)
is generated from the `SHA256` hash of the `tar` stream of the layer. The format
of the layer is not clearly specified, as the `tar` stream of an empty layer is
(as you might guess) empty!

`layerStore.applyTar` appears to concur with this theory, but it also is not
clear what the format of the `tar` stream is. Looking into the AUFS and Overlay
graph drivers, there's not many comments explaining what to expect for the
incoming `tar` stream. It does appear to just be the `layer.tar` from above, but
a quick test shows this isn't the case:

```language-bash
% shasum -a 256 image/120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16/layer.tar
7f02483a97528d2c20fa0adb6b851743823616bc8e5bec765e04c8b061ae039b  extract/120e218dd395ec314e7b6249f39d2853911b3d6def6ea164ae05722649f34b16/layer.tar
```

The two hashes don't match. If you follow the graph driver all the way down the
rabbit hole you pass the `chrootarchive` package (which executes a subprocess to
run a single Go function, so it can `chroot` rather than dealing with paths in Go)
and end up in `archive.ApplyLayer`. This function appears to back up the assertion
that it's just the `layer.tar` from above. But we know it isn't. That's as far
as I got investigating how to forge a `DiffID`, although it's probably possible
to patch the Docker daemon to dump the image tar.

#### Generating New Layers ####

As I mentioned above, there's no real way of generating a new layer without
exporting the current state of the rebase to an image `tar` archive, importing
it into Docker, running `docker run` and `docker commit`, then exporting the new
image and applying that layer to the rebase. This is very inefficient (and would
be improved if Docker offered this functionality), and could definitely use some
improvement. One possibility would be to try to use `runc` to generate a new
layer. Unfortunately it would require quite a bit of trickery as we'd need to
diff the resultant filesystem (which could be a problem for large updates). Not
to mention the fact that `runc` configuration files are quite different from
Docker layer JSON configuration files.

### Show me the Code! ###

The code is available on my [GitHub][source]. It's licensed under the Apache 2.0
license. Unfortunately, the Docker image format lacks so much documentation that
I couldn't finish implementing the rebase model. All of the graph loading code
is done though (which is honestly the bulk of the work). I probably will work on
this project in my spare time, or in the next Hackweek.

[source]: /src/docker-rebase
