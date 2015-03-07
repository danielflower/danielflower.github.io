---
layout: post_page
title: Introducing the Multi-module Maven Release Plugin for Git
---

It's 2015 and I'm writing a Maven plugin. Most Java developers that I respect have a rather strong dislike of
Maven, so what am I doing here? Well I don't think Maven is fundamentally too bad as it is pretty stable and
does most of what I want in a build tool. I basically want:

* A simple way to define my dependencies
* A test runner
* A JAR file (or tgz) containing my classes and dependencies created with version numbers
* A simple release process that uploads versioned packages to something like Nexus

Maven mostly fails cosmetically on the first 3 - the XML is in the most verbose format possible, and the XML is, well,
XML. But aside from that it is quite straightforward. Alternatives like Gradle are probably better (I much prefer
the ability to script over coding in XML), but it's
still not as mature and it just hasn't been quite strong enough to break through my intertial co-efficent
of laziness for properly learning a new build system.

Aside from the copious XML, another place where Maven really doesn't do well is in releases - particularly when you have
multiple modules within
a single repository. While it's possible to release single modules within a Maven project with the default
`maven-release-plugin`, this will not build and release dependent modules properly. It is common that for each
release, every module is released, and they all share the same version. They don't have to share the same version,
but if they don't then the tag on your repo will be of the aggregator name and version, so it is hard to locate
specific versions of a module in your VCS history. Finally, I have found that when the release process makes
commits to your repo (as the maven-release-plugin does to update the versions in the pom) then it results in a
messy VCS history, lots of little merges, and it makes continuous deployment a bit more difficult, as if you
really release to some environment on a commit, then the release will go into an infinite release loop. Also
the fact that tests run twice by default is a waste of time.

After years of working on many different projects of various sizes and complexity, myself and some others identified
what we really wanted from a release process:

* Each module in a multi-module project should be versioned independently.
* A module should not be re-released if it hasn't changed since the last release
* Semantic versioning is important, but we don't need the full release version in source control. Given a
"development version" such as "1.0-SNAPSHOT", the first release version should be "1.0.0", the second
"1.0.1", etc. Humans control the the first part of the version; computers automatically control the last part.
* There should be a tag for each module released
* Errors should be detected early and reported clearly

By supporting only Git, these requirements become relatively simple to implement because using jgit, questions such
as "has this module changed since the last release" are relatively simple to answer.

The way the plugin works is rather simple:

1. For each module in the repo, figure out an appropriate release version
2. Rewrite each pom to its release version, and update dependencies between them
3. Run `mvn deploy` (or whatever is requested by the user)
4. Tag the repo and revert the pom changes

A simple example
----------------

Given a simple project where there is a parent, a library, and an app that uses the library, you may start with
the following versions during development:

    parent 1.0-SNAPSHOT
       |--lib 2.0-SNAPSHOT
       |--app 1.5-SNAPSHOT (depends on lib 2.0-SNAPSHOT)

Then running `mvn releaser:release` will result in the following being deployed into Nexus:

    parent 1.0.0
       |--lib 2.0.0
       |--app 1.5.0 (depends on lib 2.0.0)

The git repo will have the following tags added: `parent-1.0.0`, `lib-2.0.0`, and `app-1.5.0`. Note that the tags point
to the SNAPSHOT versions, as the concrete versions are never actually committed to the repo. So development will continue
with the app having verison `1.5-SNAPSHOT`, for example.

Now imagine a change is made only to `app` and the project is released again. The fact that only the app has changed will be
detected by the plugin, and so the following will be released:

    app 1.5.1 (depends on lib 2.0.0)

The only new tag will be `app-1.5.1`. Note how the version of `lib` used by app is the version made in a previous release.

If there is now a change made to `lib`, then the following will be released:

    lib 2.0.1
    app 1.5.2 (depends on lib 2.0.1)

And the tags will be `lib-2.0.1` and `app-1.5.2`. In this case, although `app` hasn't been changed directly, it depends on
`lib`, and `lib` has changed, and so `app` is released with the new version of `lib`.

All this time, `parent` has not been re-released because it hasn't changed.

Note that it also works with projects that have just a single module, projects where the parent is separate from the
aggregator, nested modules within modules, and more.

Migrating from the maven-release-plugin
---------------------------------------

It is generally enough to just delete any `maven-release-plugin` references you have (and you can delete any scm plugins too),
and then change your release build to just call `releaser:release`.

Installing
----------

For information on the current version and other configuration, see the
[official documentation at the plugin page](https://github.com/danielflower/multi-module-maven-release-plugin).

I hope you will give it a try and let me know if you have any feature requests, bugs, or other feedback on
the [issue tracker at GitHub](https://github.com/danielflower/multi-module-maven-release-plugin/issues).
