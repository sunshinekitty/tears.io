---
layout: post
comments: true
title: Kubernetes + Jenkins = 💔
preview: I've spent the past month migrating many projects to Jenkins.  I would like to highlight a few of the issues I found with Jenkins on Kubernetes.
---

I've spent the past month migrating many projects to Jenkins.  At the beginning of my journey I felt it was a good idea to use Jenkins inside of existing Kubernetes infrastructure.  The idea of dynamic build slaves spinning up and down upon demand seemed great on paper.  I would like to highlight a few of the issues I found with running Jenkins and it's slaves in Kubernetes.  You may find that your business requirements differ and that none of these problems affect you.


### Jenkins Master is Ephemeral

This can be avoided by mounting an ebs or efs volume, but adds to the hassle of getting up and running.


### Debugging failed builds

When your CI service is running smoothly you couldn't be happier.  However in the early stages of setting things up you will probably run into a few issues here and there.  The problem is that while running inside Kubernetes your slaves are destroyed after each build has finished.  There is no way to keep the environment up and running without putting a very long sleep inside your build step before where you think the failure is.


### Plugin support

There are a couple of Jenkins plugins for using Kubernetes build slaves.  The first I tried was [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) which seemed alright after the initial troubles of getting it setup, though as I began moving projects I found it very limiting in what I could do with the slaves.  In order to build Docker containers in the slaves a custom pod template was required which this plugin did not support.  I then stumbled upon [Kubernetes CI Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+CI+Plugin) which had just the functionality I needed.

I did end up having problems with the latter plugin, which I submitted a bug to it's developer at [JENKINS-41071](https://issues.jenkins-ci.org/browse/JENKINS-41071) ~~though no telling when it will actually be fixed~~.  This issue has since been fixed.


### Build containers are..janky

Late into the process I found I needed to have a MySQL instance running to execute certain tests for a project.  This should be simple to do with Kubernetes, simply add another container to the pod template as seen above.  But wait, the plugin we're using passes additional arguments to the slaves via Kubernetes args, something the jnlp slave requires.  It's great this is done automatically for those who don't need to run more than one container, but there's no way to prevent this from happening.  When trying to run another container in the same pod I found that I had to modify the Docker CMD in Kubernetes by using the args parameter to something that wouldn't throw up when these extra arguments were tacked on.

After all this trouble I still found that if a job takes longer than 15 minutes to run the jnlp slave will somehow lose connection with the master and the build will sit idle forever.

The last issue I had with build slaves was that as the requirements of Jenkins grew so did my containers.  This affected the startup time of the jobs and did not provide a good user experience.


### Take Aways

There were some niceties to using Jenkins in Kubernetes.

You're able to easily inject environment variables and secrets into slaves using Kubernetes with config maps.

Build slaves are very easy to manage and keep consistent by using containers.

If you are able to keep slave container sizes down spin-up time of containers is
tolerable.

For simple projects this should work very well for you, but plugins seem to be a large limiting factor in having this scale to more than a few simple projects.

I would like to revisit this in the future, and see if I can contribute to the lack of functionality in plugins.
