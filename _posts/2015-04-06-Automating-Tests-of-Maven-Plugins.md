---
layout: post_page
title: Automating tests of Maven plugins
draft: true
---

When I wrote my first Maven plugin, I didn't even try to write end to end tests for it. There were unit tests, sure, but
nothing to actually check what happens when a Maven project using my plugin executed the plugin. The only way to test was
to manually add the plugin to some project and see what happens. As this is a boring manual step, it often gets neglected
during a updates.

When writing [the multi-module maven release plugin](/2015/03/08/The-Multi-Module-Maven-Release-Plugin-for-Git.html) I realised
that my cavalier attitude to automated testing was untenable, due to the fact that the plugin needs to work with all kinds
of Maven project layouts (single modules, multiple modules, parents acting as aggregators, sibling parents, nested modules, etc)
and also it needed to interact with Git repositories and push tags to remote locations. I knew that if I didn't have a test
suite against that releases would inevitably lead to regression errors. So I started to look at how to do integration tests
for plugins.

### How the official plugins test

I downloaded some of the official maven plugins and looked at how they tested. I didn't stumble across any full end-to-end
tests, but did see them creating mock objects and injecting them into the Maven dependency injection framework.

As an aside, in my opinion dependency injection is one of the most important practices in object-oriented programming, but I'm yet to
come across a dependency injection framework that can beat plain and simple Java code for wiring up applications.

I admit I don't understand the intricacies of the Maven dependency injection framework, so I wasn't confident that mocking out
large parts of maven in my tests would give me the confidence I was looking for in my tests.

In the small amount of time I looked, I didn't find any good guides on doing what I wanted. The rest of this post explains what
I ended up doing.

### What I wanted to test

My plugin behaves differently depending on the project's module structure, the git history of the project, configuration of
the plugin and possibly the version of maven being used.

So I needed to have a bunch of sample projects that I could edit and create a git history for, then execute Maven as if a person
(or build agent) was executing the plugin against he project. I wanted to assert that the output of the build had useful messages
(and more importantly, useful error messages when things went wrong) and that maven deployed the things I wanted to.

### Using sample projects in tests

I created a number of projects in a sub-folder called [test-projects](#TODO). I then created a simple utility method that would
copy the project into a unique sandbox folder so that I could mess around with it without affecting other tests. The first thing
I did was create a "photocopier" class that copied the project into a unique folder:

{% highlight java %}

    public class Photocopier {
        public static File copyTestProjectToTemporaryLocation(String projectName) throws IOException {
			// Step 1: find the test-projects folder. This may be different depending on if it the tests
			// are run from Maven or from an IDE, so two locations are checked.
            File source = new File("test-projects", projectName);
            if (!source.isDirectory()) {
                source = new File(FilenameUtils.separatorsToSystem("../test-projects/" + projectName));
            }
            if (!source.isDirectory()) {
                throw new RuntimeException("Could not find project " + projectName);
            }

			// Step 2: Create a unique folder and copy everything there
            File target = folderForSampleProject(projectName);
            FileUtils.copyDirectory(source, target);
            return target;
        }
    
        public static File folderForSampleProject(String moduleName) {
            return new File(FilenameUtils.separatorsToSystem("target/samples/" + moduleName + "/" + UUID.randomUUID()));
        }
    }
{% endhighlight %}

I had an extra requirement in that I needed my projects to be in Git repos, cloned from a "remote" repo so that changes could be
pushed there. To do that, I used the Photocopier above, then used `jgit` to run `git init` on the directory, then created another
empty directly, and ran `git clone` using a relative file path to the `remote`. So I ended up having two separate folders: a `local`
and a `remote`. I created a [separate class](#TODO) for creating these.

Finally, I created a bunch of static methods so that I could create each project in one line, and I created a simple class called
`TestProject` that had a few useful fields, like a reference to the directory that the project was in.

{% highlight java %}

    public static TestProject singleModuleProject() {
        return project("single-module");
    }
    public static TestProject nestedProject() {
        return project("nested-project");
    }
    public static TestProject moduleWithScmTag() {
        return project("module-with-scm-tag");
    }
    public static TestProject moduleWithProfilesProject() {
        return project("module-with-profiles");
    }
    public static TestProject inheritedVersionsFromParent() {
        return project("inherited-versions-from-parent");
    }
    public static TestProject independentVersionsProject() {
        return project("independent-versions");
    }
    public static TestProject parentAsSibilngProject() {
        return project("parent-as-sibling");
    }
    public static TestProject deepDependenciesProject() {
        return project("deep-dependencies");
    }
    public static TestProject moduleWithTestFailure() {
        return project("module-with-test-failure");
    }
    public static TestProject moduleWithSnapshotDependencies() {
        return project("snapshot-dependencies");
    }
{% endhighlight %}

In my tests, I could create sandboxed project folders effortlessly: `TestProject testProject = TestProject.singleModuleProject();`

Taking the time to create sandboxed folders and classes like this pays off extremely quickly.

### Referencing the plugin under test

Each pom of each test project had a plugin declaration section, including the plugin that was being written. Here is an example:

	<plugin>
		<groupId>com.github.danielflower.mavenplugins</groupId>
		<artifactId>multi-module-maven-release-plugin</artifactId>
		<version>${current.plugin.version}</version>
		<configuration>
			<releaseGoals>
				<releaseGoal>install</releaseGoal>
			</releaseGoals>
		</configuration>
	</plugin>


			
			