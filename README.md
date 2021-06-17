# Red Hat CodeReady Workspaces - Advanced Configuration

## Custom Devfile Registry

Based on the [official documentation](https://access.redhat.com/documentation/en-us/red_hat_codeready_workspaces/2.8/html/administration_guide/customizing-the-registries_crw), but simplified!

The best way to layout a standardized workspace for a project is with a `devfile`.  Although it's perfectly valid to reference a `devfile.yaml` file that's part of a public git repository, that might not always be the best option for you and your team.

You may only have private git repositories, or you might simply want a nice integrated experience to get devs up and running. If so, a custom Devfile Registry might be right for you.

## Creating A Custom Devfile Registry

First, clone the [CodeReady Workspaces git respoitory](https://github.com/redhat-developer/codeready-workspaces) and switch to the branch that matches your version of CodeReady Workspaces. For example, if you are using version `2.8`:

```
$ git clone https://github.com/redhat-developer/codeready-workspaces.git
$ cd codready-workspaces
$ git checkout crw-2.8-rhel-8
```

Next, change to the "devfiles" directory:

```
$ cd dependencies/che-devfile-registry/devfiles
```

Create a new directory for your devfile, then change to it:

```
$ mkdir my-devfile
$ cd my-devfile
```

Copy your `devfile.yaml` into this directory, then create a new file called `meta.yaml`:

```
$ cp <some dir>/devfile.yaml .
$ touch meta.yaml
```

Edit `meta.yaml` to includ infomration and tags relevanat to your devfile.  This will be what appears in the CodeReady Workspaces UI:

```
---
displayName: "My Custom Workspace"
description: Custom workspace including, including supporting tools such as PostgreSQL and PgAdmin4..
tags: [".NET", "C#", ".NET SDK", ".NET Runtime", "Netcoredbg", "Omnisharp", "UBI8"]
icon: /images/type-dotnet.svg
globalMemoryLimit: 2198Mi
```

With this done, you're ready to build your new container image.

The example below uses Docker on your your desktop, but you can just as easily use Podman or Buildah.

To build the new Devfile Registry image:

1. Login to your image registry (OpenShift image registry, quay.io, etc...):

```
$ docker login <registry url>
```

2. Back out 2 directories.  You should be in the in the `che-devfile-registry` directory:

```
$ cd ../..
$ pwd
<some path>/codeready-workspaces/dependencies/che-devfile-registry
```

3. Run the following command to build the image:

```
$ docker build \
    -t <registry url>/crw-devfile:2.8.0 \
    -f ./build/dockerfiles/Dockerfile \
    --target registry \
    --build-arg "PATCHED_IMAGES_TAG=$(head -n 1 VERSION)" \
    --build-arg "USE_DIGESTS=false" .
```

Don't forget the `.` at the end!

An example, using my own registry location:

```
$ docker build \
    -t quay.io/pittar/crw-devfile:2.8.0 \
    -f ./build/dockerfiles/Dockerfile \
    --target registry \
    --build-arg "PATCHED_IMAGES_TAG=$(head -n 1 VERSION)" \
    --build-arg "USE_DIGESTS=false" .
```

4. Push the image to your registry:

```
$ docker push <registry url>/crw-devfile:2.8.0
```

5. Update CodeReady Workspaces CheCluster CR

Finally, with the image pushed, you need to update the CodeReady Workspaces `checluster` custom resource to use this new image.

You can see a full example of the custom resource in the root of this directory.  If you want to simply update your existing instance, then edit your existing `checluster` and add the following lines to the `server` stanza, substituting for the location of your image.

```
  server:
    devfileRegistryPullPolicy: Always
    devfileRegistryImage: 'quay.io/pittar/crw-devfile:2.8.2'
```

Unless you plan on updating your checluster definition with a new tag each time you make a change, you will want to make sure you set the `devfileRegistryPullPolicy` to `Always`.  That way, all you need to do is restart the `devfile-registry` deployment in order to get the latest changes.

That's it!  Once you save this configuration the CodeReady Workspaces operator will restart the devfile-registry with your custom image.  The next time you login to CodeReady Workspaces you should see your custom devfile listed among the "Getting Started" tiles!


