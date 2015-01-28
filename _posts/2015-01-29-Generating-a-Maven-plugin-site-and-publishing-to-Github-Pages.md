---
layout: post_page
title: Generating and a Maven plugin site and publishing to Github Pages
---

I have been working on a Maven plugin with a friend recently. This is the second plugin I've written for Maven. The first one I wrote
(the [maven gitlog plugin](https://github.com/danielflower/maven-gitlog-plugin)) simply had a `README.md` in the root
of the GitHub repo showing usage instructions, rather than generating a Maven site and publishing that.

I have always actively avoided Maven sites. I think it is because they are often full of automatically-generated useless
information, and by default they look horrible:

![Screenshot of default maven site](images/mvn-site/horrible-site.png)

But this time, I decided it would be worth actually learning how to create maven sites properly, because I am hoping
the plugin I'm working on will be used a bit, and every time I use a plugin and hit a maven generated site I go straight
to the usages and goals pages. For someone who uses maven often, this is going to make things quick, and also very importantly
the version of the plugin is always there, nice and clear.

I had also noticed there were some nicer looking Maven sites around these days:

![Screenshot of nicer maven site](images/mvn-site/better-site.png)

Now that is a design that doesn't offend my eyes too badly. But while there is plenty of documentation around the web on maven sites, I found it
wasn't clear on how to do exactly what I wanted to do, which was namely:

* How to use the newer, nicer looking Maven template? (hard to find when you don't know the name)
* How can I generate the "usage" and "goals" pages for a maven plugin, so that all the documentation is generated from Javadoc?
* How can I limit the amount of default reports that it spits out?
* How can I have a custom homepage?
* How can I have it convert markdown files to HTML pages?
* How can the current release version be automatically updated on each release?
* How can I automatically upload it to my `gh-pages` branch in my plugin's repo so that it gets republished automatically?

Turns out all the above is really simple once you know how.

Inputs and outputs
------------------

The Git repo that I wanted the documentation for is at 
[https://github.com/danielflower/multi-module-maven-release-plugin](https://github.com/danielflower/multi-module-maven-release-plugin). Given
a URL like that, by having static HTML pages in the `gh-pages` branch, GitHub will automatically publish your site to a URL like
[http://danielflower.github.io/multi-module-maven-release-plugin/](http://danielflower.github.io/multi-module-maven-release-plugin/) which
is a perfectly charming URL.

This repo is a Maven plugin and so has some mojos defined using annotations, with some javadoc, for example:

```java
    /**
     * If true then tests will not be run during a release.
     */
    @Parameter(defaultValue = "false", property = "skipTests")
    private boolean skipTests;
```

Given code like that, you get documentation generated looking like this:

![Optional paramaters for the plugin](images/mvn-site/mojo-options.png)

Generating the site
-------------------

The first thing you should do is add a reference to the Maven site plugin and add the plugin that allows conversion of markdown
to HTML during site generation:

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-site-plugin</artifactId>
                <version>3.4</version>
                <dependencies>
                    <dependency>
                        <groupId>org.apache.maven.doxia</groupId>
                        <artifactId>doxia-module-markdown</artifactId>
                        <version>1.6</version>
                    </dependency>
                </dependencies>
            </plugin>

At this stage, running `mvn site` will generate an ugly site and stick it your `target/site` folder, however it will not have
any documentation generated for the plugin goals and parameters. To do that, you need to add the `maven-plugin-plugin` to the
`<reporting>` section of your pom.

Working on a plugin, you will already have the `maven-plugin-plugin` defined elsewhere. You just need to define it again. If
using a goal prefix - which you should be - you will need to configure this in both declarations of the plugin-plugin otherwise
the documentation will be wrong. So using a property is a good idea for this.

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-plugin-plugin</artifactId>
                <version>3.4</version>
                <configuration>
                    <goalPrefix>${plugin.goal.prefix}</goalPrefix>
                </configuration>
            </plugin>

Running `mvn site` now will generate a bit too much stuff. All those default reports really are rather useless, and also an
`index.html` is generated. I wanted to have my own custom index page, and not have so many reports. To do this, you need to
actually define another report that is normally defined by default and specify just the reports you do want:

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-project-info-reports-plugin</artifactId>
                <version>2.8</version>
                <reportSets>
                    <reportSet>
                        <reports>
                            <report>project-team</report>
                            <report>cim</report>
                            <report>issue-tracking</report>
                            <report>license</report>
                            <report>scm</report>
                        </reports>
                    </reportSet>
                </reportSets>
            </plugin>

Adding custom pages
-------------------

At this stage, a [goals](http://danielflower.github.io/multi-module-maven-release-plugin/plugin-info.html) page is automatically generated, 
however [usage](http://danielflower.github.io/multi-module-maven-release-plugin/usage.html) is not. This is a page you will just have to
manually make, which is a good thing really. As the markdown dependency was registered with the site plugin above, this can be a markdown
page.

All custom resources for your Maven site go into `src/site/{filetype}/filename` - so my usage page for example - being markdown - is located
at [src/site/markdown/usage.md](https://github.com/danielflower/multi-module-maven-release-plugin/blob/master/src/site/markdown/usage.md.vm)
and this is just a simple markdown file.

The usage page includes instructions on how to use the plugin. The source looks like this:

    <build>
        <plugins>
            <plugin>
                <groupId>${project.groupId}</groupId>
                <artifactId>${project.artifactId}</artifactId>
                <version>${project.version}</version>
                <configuration>
                    <releaseGoals>
                        <releaseGoal>install</releaseGoal>
                    </releaseGoals>
                </configuration>
            </plugin>
        </plugins>
    </build>

Obviously I wanted do have the `${project.*}` variables automatically replaced during site generation. This didn't work the first time, but
after some googling I found you just need to append `.vm` to your filename and then you can use pom expressions. So this is cool - the version
specified here will always be up to date.

To create the homepage, I simply added a file called [index.md](https://github.com/danielflower/multi-module-maven-release-plugin/blob/master/src/site/markdown/index.md)
to the `markdown` folder.

Navigation
----------

Navigation is defined in [src/site/site.xml](https://github.com/danielflower/multi-module-maven-release-plugin/blob/master/src/site/site.xml):

    <body>
        <menu name="Overview">
            <item name="Introduction" href="index.html"/>
            <item name="Goals" href="plugin-info.html"/>
            <item name="Usage" href="usage.html"/>
            <item name="FAQ" href="FAQ.html"/>
        </menu>
        <menu ref="reports" inherit="top"/>
    </body>

In the 'Overview' section, only 'Goals' is a generated page, which is why it has the slightly strange name. Just set it to 'plugin-info.html'
and it will find it there.

The `reports` section will contain links to all auto-generated pages.

Using the nicer design
----------------------

To change the layout of a maven site, you specify a skin in the site.xml file. The following will get you the newer, more modern design:

    <skin>
        <groupId>org.apache.maven.skins</groupId>
        <artifactId>maven-fluido-skin</artifactId>
        <version>1.3.0</version>
    </skin>
    <custom>
        <fluidoSkin>
            <topBarEnabled>true</topBarEnabled>
            <sideBarEnabled>true</sideBarEnabled>
            <googleSearch>
                <sitesearch>${project.url}</sitesearch>
            </googleSearch>
            <gitHub>
                <projectId>danielflower/multi-module-maven-release-plugin</projectId>
                <ribbonOrientation>right</ribbonOrientation>
                <ribbonColor>gray</ribbonColor>
            </gitHub>
        </fluidoSkin>
    </custom>

You can also see a few other config options around the 'fork me on github' banner, and google search.

Generating the site during a release
------------------------------------

Just include `site` as one of the maven goals and it will stick it in `target/site`.

Automatically uploading the site to your GitHub pages area
----------------------------------------------------------

I was very happy to see that GitHub had already solved this problem, and it worked first-time without any troubles. All you need to do
is define your GitHub credentials in your `.m2/settings.xml` file, add a maven plugin, and run `mvn site`.

First off, put your GitHub username and password in your Maven settings associated with the server ID 'github':

    <settings>
        <servers>
            <server>
                <id>github</id>
                <username>danielflower</username>
                <password>mypassword</password>
            </server>
        </servers>
    </settings>

In your pom, you do need to tell the plugin which server ID you used:

    <properties>
        <github.global.server>github</github.global.server>
    </properties>

And now just declare the site-maven-plugin:

    <plugin>
        <groupId>com.github.github</groupId>
        <artifactId>site-maven-plugin</artifactId>
        <version>0.11</version>
        <configuration>
            <message>Creating site for ${project.version}</message>
        </configuration>
        <executions>
            <execution>
                <goals>
                    <goal>site</goal>
                </goals>
                <phase>site</phase>
            </execution>
        </executions>
    </plugin>

It's probably a good idea to associate that plugin with profile that you only enable during releases so that you don't upload your maven site
every time you run a build.

The documentation for this plugin is at [https://github.com/github/maven-plugins](https://github.com/github/maven-plugins)

Done
----

That's it. Sure, there is XML-a-plenty, and some of the naming is a bit weird, but in the end it is quite fast to have a plugin page
that looks reasonably professional, has useful information, and is automatically published.