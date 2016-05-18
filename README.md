# GoProx

[![Build Status](https://travis-ci.org/arschles/goprox.svg?branch=master)](https://travis-ci.org/arschles/goprox)
[![Go Report Card](http://goreportcard.com/badge/arschles/goprox)](http://goreportcard.com/report/arschles/goprox)
[![Docker Repository on Quay](https://quay.io/repository/arschles/goprox/status "Docker Repository on Quay")](https://quay.io/repository/arschles/goprox)

A proxy cache server for Go packages, for you to run in your own cloud.

# Alpha
This project is alpha status. It's in early development and not all features are available, but it's ready for testing. See below for more information

# About

GoProx is an open source Git server that proxies requests to Go packages and caches package code outside of hosted source code repositories. Since it speaks the Git protocol, popular tools like [glide](https://github.com/Masterminds/glide), [Godep](https://github.com/tools/godep), and even `go get` can talk to GoProx instead of GitHub (or others).

The server stores and streams its git repositories on [Amazon S3](https://aws.amazon.com/s3/), and achieves slightly faster download times than [GitHub](https://github.com) in some cases. Below is an example benchmark run on the same machine with the same network connection. Note that the Kubernetes repository is approximately 316 MB.

Directly from GitHub:

```console
ENG000656:Desktop aaronschlesinger$ time git clone https://github.com/kubernetes/kubernetes.git
Cloning into 'kubernetes'...
remote: Counting objects: 267274, done.
remote: Compressing objects: 100% (19/19), done.
remote: Total 267274 (delta 6), reused 2 (delta 2), pack-reused 267253
Receiving objects: 100% (267274/267274), 191.03 MiB | 891.00 KiB/s, done.
Resolving deltas: 100% (175363/175363), done.
Checking connectivity... done.

real	5m13.588s
user	0m13.046s
sys	0m7.299s
```

From a GoProx server running on [Google Container Engine](https://cloud.google.com/container-engine/) (which is in [Google Cloud](https://cloud.google.com/)):

```console
ENG000656:k8s1 aaronschlesinger$ time git clone http://git.goprox.io/github.com/kubernetes/kubernetes
Cloning into 'kubernetes'...
remote: Counting objects: 260017, done.
remote: Compressing objects: 100% (82979/82979), done.
remote: Total 260017 (delta 170072), reused 260017 (delta 170072)
Receiving objects: 100% (260017/260017), 187.82 MiB | 1.41 MiB/s, done.
Resolving deltas: 100% (170072/170072), done.
Checking connectivity... done.

real	4m5.802s
user	0m10.398s
sys	0m6.450s
```

# Why It Helps

GoProx lets you control the source of your dependeny _server and code_, so regardless of what happens with the original code, you'll always have a copy in your S3 account, and you'll always be able to access it as long as you have a GoProx server running.

Since it's not a shared server, database or code repository, you don't need to rely on a third party to serve or store your dependencies for you. Instead, you maintain the server and store dependencies in S3.

# Problems It Solves

Not relying on third parties to hold your dependency code for you is important, given how Go dependency management works.

All the tools we have today for tracking, managing and/or fetching Go dependencies download raw Go code or metadata directly from GitHub and other popular source code repos.

This method is simple and very effective, but it comes with some pitfalls:

1. If the hosted repo is unavailable, your build won't have access to required dependencies and it will fail
2. If the author of a library you depend on modifies a tag or a commit, the code you depended on may change or disappear

The above two pitfalls aren't just theoretical either.

Downtime happens for all repositories, but Google Code has shut down and it had a [big impact](https://www.reddit.com/r/golang/comments/42r1j7/codegooglecom_is_down_all_packages_hosted_there/) on the Go community at large. To a project that depended on a `code.google.com` package, it looked like the dependency was deleted. The Google Code shutdown was a big inspiration for me to build GoProx.

(note that even in other languages these problems exist. The node.js community had a big [issue](http://blog.npmjs.org/post/141577284765/kik-left-pad-and-npm) when someone deleted a small package that was very "deep" in many large dependency trees)

## Example Scenario

Let's imagine a scenario that shows how relying on third party repositories can be harmful to your project, and then show how GoProx can help.

Say your project depends on a (fictional) package of mine called `github.com/arschles/gofictional`. You use [glide](https://github.com/Masterminds/glide), so you get repeatable, deterministic builds because you've pinned your version of that package to a known Git SHA.

One day, I update my package, squash it into the commit SHA that you depend on and push the update to my `master` branch. This means that next time you pull dependencies (for example, in your CI system), your project will build with my new update.

At best, your code won't build, but at worst your tests won't catch the change and you'll ship fundamentally different software without knowing it.

If you change the dependency's source to use GoProx, you'll be effectively pulling the code from your S3 bucket, not from GitHub. Even though I squashed commits into the original SHA, the code GoProx will serve won't have it. If you decide to use the updated code, you can tell GoProx to update its cache.
