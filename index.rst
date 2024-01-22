#####################################
Organization of Phalanx into projects
#####################################

Abstract
========

   Phalanx is currently a very flat structure, with no hierarchy or security controls beyond being able to login to ArgoCD.  Here we will propose a structure to the Phalanx repository to allow for different projects and access control to those projects.

Problem Statement
=================

Over time, Phalanx has gotten more crowded and full over time, mostly due to its own success.
Phalanx has grown on a few different axes:

#. The number of people and groups utilizing Phalanx for configuring their environments
   and deploying their software.
#. The number of environments controlled by the Phalanx configuration
#. The number of applications / services that are deployed and configured by
   the Phalanx environment.

Some growing pains have come from these things happen, especially on the growth
of multiple axes.

#. As more people gain access to ArgoCD, this essentially gives them control over
   important parts of the cluster that they won't normally need access to.  With
   more people having access, it is more likely that accidents will happen.
#. As the number of environments increase, different environments have different
   sets of applications that need to be run on them.  For example, what is being
   run on the summit or the base is going to be different from the clusters
   running in Google.
#. As the number of applications increase, this can complicate the applications
   folder, and make it hard to understand which applications are related, and
   which are not.  For example, looking at the applications directory, it will
   be displayed in lexical sort order, which makes it hard to understand the
   relations for the different applications.  While renaming all these is
   a possibility, this name would be carried into the UI making it kind of
   ugly.  All this adds a bit of cognitive load when looking at the repository
   and many of these same problems also carry over into the ArgoCD UI.  The
   ArgoCD UI seems to struggle a bit with large numbers of applications.

Restructuring of the Phalanx Repository
=======================================

ArgoCD Project Definition
=========================

One thing to point out is that the layout of the Phalanx git repository
is completely unrelated to how ArgoCD lays out the applications and
projects.  The view that ArgoCD makes depends on the yaml files that
define the application, not the location of those files in the repository.

For example the line that sets the project in the ArgoCD application is
found at: https://github.com/lsst-sqre/phalanx/blob/7bf274d80199bfc3bd5aca49c99e03bcb97fd01c/environments/templates/alert-stream-broker-application.yaml#L25

For example the line that sets the path that points to the application
chart can be found at: https://github.com/lsst-sqre/phalanx/blob/7bf274d80199bfc3bd5aca49c99e03bcb97fd01c/environments/templates/alert-stream-broker-application.yaml#L27

So this means that we can reorganize these files as we want without
or with changing the project name.

Potential Project Lists
=======================

First we need to see how we want to break down the monolith into
separate projects.  These projects may be related to their usage,
their importance, what team is working on them, where they are
deployed, or anything else we deem necessary.

#. Infrastructure
   - argocd
   - cert-manager
   - gafaelfawr
   - ingress-nginx
   - squareone
   - vault-secrets-operator
#. RSP
   - butler
   - datalinker
   - hips
   - livetap
   - giftless
   - nublado
   - mobu
   - portal
   - postgres
   - tap
   - siav2
   - ssotap
   - vo-cutouts
#. EFD
   - telegraf
   - telegraf-ds
   - sasquatch
   - strimzi
   - exposurelog
   - narrativelog
#. Telescope
   - all the telescope services

1: Breaking up the applications directory
=========================================

In the applications directory, create one directory for each project in
the section above, and use gitmv to move each directory in applications
into its project.

Once that is done, go to the environments directory and edit the paths
to add the project name into the path.  At this point, things should
still work without any other changes.

2: Break up the environments directory
======================================

Now in the environments/templates directory, you will find one yaml
per application in the applications directory.  Make directories with
the project names and move the yaml file for each application into
the appropriate project directory.

Note that doing this does not change the project yet.  Open up each
yaml file in a directory and set the ``spec: project:`` key to the
name of the project and directory.  Now that application will be
considered part of that project.

At this point, things should still work for us all by itself and
not doing further steps.

3: Add project level to environment values.yaml
===============================================

Now this has absolutely nothing to do with argocd and really offers
no technical benefit, but it does maybe offer a little bit less
cognitive load when working with the environments yaml files.

Right now, the values files look like this:

applications:
  argocd: true
  butler: false
  exposurelog: false
  ...

Sometimes containing all the applications.

So now that we are breaking things up on projects, this allows
for one additional level of hierarchy and some syntatic sugar.

applications:
  infrastructure:
    enabled: true
  rsp:
    portal: true
    nublado: true
    tap: true
  telescope:
    enabled: false

I think each project should pick that it is either a "salad bar"
type project, where you pick what you want, or an "all-or-nothing"
project like infrastructure.  This prevents complications where
you have both enabled and applications, and trying to handle
special rules would make things more confusing.

Also many of the projects contain applications that rely on
each other.  For example, ArgoCD relies on ingress-nginx, and
vault-secrets-operator is required by many of the applications.

4. Project Organization
=======================

At this point, after doing the reorganizations in the above sections,
the current people who can access ArgoCD can try out some of the UI
features in ArgoCD for filtering by project.  This should provide
some help and will give us ideas that we are on the right track
with how the UI works.

5. Adding RBAC To Each Project
==============================

Now we will talk about how to assign projects to groups of users.
We can either assign specific users to the access rules, or try
to assign groups to the access rules.  For now, we hardcode users
and we are connected to Google for SSO.  Right now I'm not sure
if there's an ability to use a group.

We currently allow everyone in SQuare to look at every project
and every application with admin privledges.  So it's more likely
that different groups will want to manage access to their own
projects and that is it.  Sometimes we'll want to share read-only
access to other projects, if that helps for investigations.

What we'll start with is each project will have a set of policies
to let someone work on that project.  We will then assign that
list of policies to a list of users that will work on it.

Here's an example of a policy for a generic project named infrastructure::

        p, role:infrastructure, applications, action/*, infrastructure/*, allow
        p, role:infrastructure, applicationsets, action/*, infrastructure/*, allow
        p, role:infrastructure, exec, action/*, infrastructure/*, allow
        p, role:infrastructure, logs, get, infrastructure/*, allow

And give them read-only access to other projects::

        g, role:infrastructure, role:readonly
