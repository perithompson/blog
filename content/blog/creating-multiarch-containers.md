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

This makes the experience for Windows Administrators or Developers pretty painful, especially when many Go binaries (that is common with Kubernetes services) can be cross compiled by using GOOS e.g. `GOOS=windows`. Recently, however, I was introduced to the mighty *Pause* Image.  This unassuming container is always there for you, supplying you with all of the kubernetes goodness and guess what it contains all the OS images you could ever ask for!  Don't believe me.... run this on Windows and linux `docker run k8s.gcr.io/pause:3.4.1`.  How does this help us you ask?

## Enter buildx!

Take a look at this docker manifest for the pause image

```
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:5fe7626c74b248c86a0c6edb05dbbd0b517866347ba7f2409ec0a3a182e96fb4",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:58a0881ed234969919fbcd39fd5b3b2ab7e02a5791d05e978a6013025fa4fc71",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:911677fee0acf1dbba7e62779e02bea89bc5c28d0ba1ebc442610bf1c39c9a56",
         "platform": {
            "architecture": "arm64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:18ba00f386182f4f9de31dc52af30f32f9aa84433df706e8b97b620fbfc0791c",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 526,
         "digest": "sha256:795a1d16653524f948c02c796ae1651bb43d12dd2983a1b05ba3aa23032238be",
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
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:66452f92a567a06da93d81944ee02db30929e32087644d5ff282ace1218f7946",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:da6fa3d6d66b5cf504ffd1a1d16f8d367bd6bd50f14cb69734ec7f16c5cb2eb1",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:da6fa3d6d66b5cf504ffd1a1d16f8d367bd6bd50f14cb69734ec7f16c5cb2eb1",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:87b29346b4f3abfabb07335d64c5043bc1dcf1f9a202793a7b8d3c5b0cf36314",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:87b29346b4f3abfabb07335d64c5043bc1dcf1f9a202793a7b8d3c5b0cf36314",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:528c5483f974f36d2f1a0f0716a7b6ec47f7dac24f18c9d4385273625c43d318",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 950,
         "digest": "sha256:528c5483f974f36d2f1a0f0716a7b6ec47f7dac24f18c9d4385273625c43d318",
         "platform": {
            "architecture": "amd64",
            "os": "windows"
         }
      }
   ]
}
```

As you can see there is a lot of images there
&nbsp;
&nbsp;

[^1]: This is only true when you are running Process Isolated containers, hyper-v isolation can get around this
[^2]: To prevent this you can Taint your node or use nodeSelectors/RuntimeClasses



