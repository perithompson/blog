---
title: "Creating Multiarch Containers"
date: 2021-02-19T16:31:48Z
slug: "creating-multiarch-containers"
description: "Post about creating multi architecture images for use with kubernetes"
keywords: ["windows","multiarch","docker","buildx","kubernetes","k8s"]
draft: false
tags: ["windows","multiarch","docker","buildx","kubernetes","k8s"]
math: false
toc: true
---

## Windows Pods

Ever since starting working with Kubernetes on windows, I've been struggling to get my head around working with daemon sets on hybrid clusters.  For those that don't know, this common Kubernetes functionality is used frequently for supporting kubernetes infrastructure such as network agents or even running e2e tests.  

The issue with Windows is two-fold:

1. Windows containers are locked to the OS version of the Host[^1]
2. Many community manifests or chart do not currently employ node selectors meaning linux containers often try to run on Windows nodes[^2]

This makes the experience for Windows Administrators or Developers pretty painful, especially when many Go binaries (that is common with Kubernetes services) can be cross compiled by using GOOS e.g. `GOOS=windows`. Recently, however, I was introduced to the mighty *Pause* Image.  This unassuming container is always there for you, acting as a temporary container while kubernetes prepares your application in a Pod!s Guess what it contains all the OS arch images you could ever ask for which, as of v3.4.1, will run on near enough any computer with a container runtime!  Don't believe me... run this on Windows and linux `docker run k8s.gcr.io/pause:3.4.1`.  How does this help us you ask?

Take a look at this docker manifest for the pause image

```
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:c20822a0b1fb541515ebf0f432be0ed0158998cb33d42eb718089ee1bc6d7873",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:2bb63f004ed7a0f1549e1c293d9d87ff6089628f79ed4ce5710a339bf492e504",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:f2b9c1d3b6b9c96938827c1fdc745f91c9543da3a27e19c995797abc0c07a409",
         "platform": {
            "architecture": "arm64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:25bac98f63f75a3b999cfa6ca522eecf640dea57147402145243f76616af2e63",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:4917ee28ee62b645de3eb837948400403ef21a52b45ce30e216f29957aaf6c77",
         "platform": {
            "architecture": "s390x",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:66452f92a567a06da93d81944ee02db30929e32087644d5ff282ace1218f7946",
         "platform": {
            "architecture": "amd64",
            "os": "windows",
            "os.version": "10.0.17763.1757"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:da6fa3d6d66b5cf504ffd1a1d16f8d367bd6bd50f14cb69734ec7f16c5cb2eb1",
         "platform": {
            "architecture": "amd64",
            "os": "windows",
            "os.version": "10.0.18362.1256"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:87b29346b4f3abfabb07335d64c5043bc1dcf1f9a202793a7b8d3c5b0cf36314",
         "platform": {
            "architecture": "amd64",
            "os": "windows",
            "os.version": "10.0.18363.1379"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:528c5483f974f36d2f1a0f0716a7b6ec47f7dac24f18c9d4385273625c43d318",
         "platform": {
            "architecture": "amd64",
            "os": "windows",
            "os.version": "10.0.19041.804"
         }
      }
   ]
}
```

Even if you aren't familiar with a docker manifest, you can see that this is actually a manifest *list*, meaning that the docker image is actually a collection of seperate manifests, each built for a different architecture or in the case of windows different OS versions...

Now, you may expect that this means in our build pipeline we have a collection of computers to make this single image but thankfully, with the magic of (qemu)[https://www.qemu.org/] and some docker trickery we actually do this on one machine, which I will go through now.

First we need 2 Dockerfiles one for Windows and one for linux, a program that can be cross compiled (in my example I am going to use Sonobuoy, a simple testing utility from VMware Tanzu) and a workstation setup to use docker.  The Windows dockerfile cannot run any actual windows commans so `RUN` is out of the question, but provided you have the components ready these can be injected using `ADD` or `COPY`.

## Enter buildx!

The build process for this image is actually done by a *(link here)!!* Makefile but as that can be a little overwhelming to consume straight, I will break this down into single steps that will help you understand and replicate this for you application. Let's start with setting up our qemu image-builder and cloning our sonobuoy repo.

``` bash
# Enable experimental docker CLI
export DOCKER_CLI_EXPERIMENTAL=enabled

# Create Qemu image builder
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
docker buildx create --name img-builder --use
docker buildx inspect --bootstrap

# Clone specific sonobuoy tag
mkdir -p $GOPATH/src/github.com/vmware-tanzu
cd $GOPATH/src/github.com/vmware-tanzu
git clone --branch v0.20.0 https://github.com/vmware-tanzu/sonobuoy.git
cd sonobuoy

# build the components
make clean
make build/windows/amd64/sonobuoy.exe
make build/linux/amd64/sonobuoy
make build/linux/arm64/sonobuoy

chmod +x build/linux/arm64/sonobuoy
chmod +x build/linux/amd64/sonobuoy
```

With this in place we can now start by editing our Dockerfiles ready. For linux I am going to create images for amd64 and arm64, for windows I am going to create an image for every version from 1809 to 20H2.  The Dockerfiles here are pretty simplistic examples but they serve the purpose for understanding the process and the only changes I am making are adding some build arguments to help with my loops. 


Dockerfile
```
ARG BASEIMAGE
FROM ${BASEIMAGE}

ARG ARCH
ADD /build/linux/${ARCH}/sonobuoy /sonobuoy

WORKDIR /
CMD ["/sonobuoy", "aggregator", "--no-exit", "-v", "3", "--logtostderr"]
```

Dockerfile_windows
```
ARG BASEIMAGE
FROM ${BASEIMAGE}

ADD build/windows/amd64/sonobuoy.exe /sonobuoy.exe
WORKDIR /
CMD /sonobuoy.exe aggregator --no-exit -v 3 --logtostderr
```

Next we will start with the getting my linux images built and pushed to my docker registry.  We also want to start to prepare a space sperated string containing a list of all of the iamges that we want to include in our manifest list.

```
# Use our image builder to build an image for linux/amd64
docker buildx build --pull --output=type=registry --platform linux/amd64 \
	-t my-repo/my-sonobuoy:v0.20-linux-amd64 \
   --build-arg ARCH="amd64" --build-arg BASEIMAGE="gcr.io/distroless/static-debian10:latest" \
   -f Dockerfile .

# Add to our manifest list
MANIFESTLIST+="my-repo/sonobuoy:v0.20-linux-amd64 "

# Use our image builder to build an image for linux/arm64v8
docker buildx build --pull --output=type=registry --platform linux/arm64 \
	-t my-repo/sonobuoy:v0.20-linux-arm64v8 \
   --build-arg ARCH="arm64" --build-arg BASEIMAGE="arm64v8/ubuntu:16.04" \
   -f Dockerfile .

# Add to our manifest list
MANIFESTLIST+="my-repo/sonobuoy:v0.20-linux-arm64v8 "
```

So far so good, but now comes the tricky part, creating windows images on a non-windows OS!

```
# Set our image tags that we want to build and the servercore base image
OSVERSIONS=("1809" "1903" "1909" "2004" "20H2")
BASEIIMAGE="mcr.microsoft.com/windows/servercore:"
```

Next we will build and push these images, much the same as the linux images above but for brevity, I'll create a loop...
```
for VERSION in ${OSVERSIONS[*]}
do 
	export BASEIIMAGE="mcr.microsoft.com/windows/nanoserver:$VERSION"
  	docker buildx build --pull --output=type=registry --platform windows/amd64 \
		-t my-repo/sonobuoy:v0.20-windows-amd64-${VERSION} --build-arg BASEIMAGE=$BASEIIMAGE -f DockerfileWindows .
   
   # Add our images to the list
	MANIFESTLIST+="my-repo/sonobuoy:v0.20-windows-amd64-${VERSION} "
done
```

At this point my-repo should have a images for all of my Windows versions and linux Architectures but they are just a collection of tags... lets put them all together!  

The docker manifest create command takes 2 arguments first is the manifest list image name and our space seperated list of all the images to include which looks like this...

`my-repo/sonobuoy:v0.20-windows-amd64-1809 my-repo/sonobuoy:v0.20-windows-amd64-1903 my-repo/sonobuoy:v0.20-windows-amd64-1909 my-repo/sonobuoy:v0.20-windows-amd64-2004 my-repo/sonobuoy:v0.20-windows-amd64-20H2 my-repo/sonobuoy:v0.20-linux-arm64v8 my-repo/sonobuoy:v0.20-linux-amd64`

We create our new manifest list
```
# We use --amend in case the manifest already exists
docker manifest create --amend my-repo/sonobuoy:v0.20 $MANIFESTLIST
```

Next we can annotate our linux images to let our CRI know which image to use later on.  These annotations can container any of the following
```
Options:
      --arch string           Set architecture
      --os string             Set operating system
      --os-features strings   Set operating system feature
      --os-version string     Set operating system version
      --variant string        Set architecture variant
```

So in our case we will do the following, (notice we can also set an arm64 variant)
```
docker manifest annotate --os linux --arch amd64 "my-repo/sonobuoy:v0.20" "my-repo/sonobuoy:v0.20-linux-amd64"
docker manifest annotate --os linux --arch arm64 --variant v8 "my-repo/sonobuoy:v0.20" "my-repo/sonobuoy:v0.20-linux-arm64v8"
```

Next we do the same for our windows images, making sure to add the windows "full" version, which we will extract from our base image
```
for VERSION in ${OSVERSIONS[*]}
do 
  full_version=`docker manifest inspect mcr.microsoft.com/windows/servercore:${VERSION} | grep "os.version" | head -n 1 | awk '{print $$2}' | sed 's@.*:@@' | sed 's/"//g'`  || true; 
  docker manifest annotate --os-version ${full_version} --os windows --arch amd64 "my-repo/sonobuoy:v0.20" "my-repo/sonobuoy:v0.20-windows-amd64-${VERSION}"
done
```

And that's it! Now, no matter which OS or Windows version we run the image on, we will no longer have to worry about the dreaded `no matching manifest for windows/amd64 {version} in the manifest list entries.`












&nbsp;
&nbsp;

[^1]: This is only true when you are running Process Isolated containers, hyper-v isolation can get around this
[^2]: To prevent this you can Taint your node or use nodeSelectors/RuntimeClasses


