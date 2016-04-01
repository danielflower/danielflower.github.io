---
layout: post_page
title: Using Swagger with embedded Jetty and without magic
---

There is a curious ailment that appears to afflict great swathes of Java developers: the wiring of an application must never use simple Java! Only
reflection-heavy, automagic, confusing configuration is allowed!

Take Jersey as an example, where the vast majority of tutorials showing how to expose your REST service in a Jetty server assumes it is okay to register
your class with the framework, and then have the framework instantiate it:

{% highlight java %}

    public class MyApplication extends ResourceConfig {
        public MyApplication() {
            register(MyResource.class);
            register(SomeOtherResource.class);
        }
    }
{% endhighlight %}

This means you can't pass dependencies into the constructor. So what should you do? Get dependencies from some
global, static, service provider.

For why?!?

It's sad that this anti-pattern of forcing instantiation of classes, and configuration, is so prevalent among
some of the biggest Java frameworks. It seems the `new` keyword is thought to be evil for some reason, and must be
avoided at all costs. Furthermore, classpath-scanning seems to be the holy grail of these
frameworks so that you don't have to configure anything. This also helps make it extremely difficult to understand
what is happening in what order.

It is possible to avoid all this nonsense by instantiating objects yourself and passing them to a resource config
object:

{% highlight java %}

    ResourceConfig rc = new ResourceConfig();
        rc.register(new MyResource());
        rc.register(new SomeOtherResource(someDependency));
{% endhighlight %}

Ahh, that's better.

Now to the point of this article: getting these manually instantiated REST services registered with Swagger.

Swagger
-------

I have come to love-hate [Swagger](http://swagger.io/). The documentation generated is great to use, and I don't
mind the copious amounts of annotation documentation added to the REST services, as the documentation lives with
the code in source control and there is no better way to specify it that I can think of. But I what I really dislike is how hard it is
to use if you use things like constructor-based dependency injection and want to have control over how it integrates
with your embedded web server.

To it's credit, the docs do show how you can register a package that has your REST resources in it without relying
on XML:

{% highlight java %}

    public static void configurSwagger() {
        BeanConfig beanConfig = new BeanConfig();
        beanConfig.setVersion("1.0");
        beanConfig.setTitle("My Rest Resource");
        beanConfig.setBasePath("/");
        beanConfig.setResourcePackage(MyResource.class.getPackage().getName());
        beanConfig.setScan(true);
    }
{% endhighlight %}

Isn't that just incredibly scary? You create an instance of an object, call some setters, and then let the object
be collected by the garbage collector. Furthermore, the order of the setters matters: `setScan` must be called
last because that "setter" is what causes the config to be applied.

I could not abide such an approach

Configuring Swagger explicitly
------------------------------

Here's what I came up with:

{% highlight java %}

    public static void registerSwaggerJsonResource(ResourceConfig rc) {
        new SwaggerContextService()
            .withSwaggerConfig(new SwaggerConfig() {
                public Swagger configure(Swagger swagger) {
                    Info info = new Info();
                    info.setTitle("My Rest Resource");
                    info.setVersion("1.0");
                    swagger.setInfo(info);
                    swagger.setBasePath("/");
                    return swagger;
                }
                public String getFilterClass() {
                    return null;
                }
            })
            .withScanner(new Scanner() {
                private boolean prettyPrint;
                public Set<Class<?>> classes() {
                    return rc.getInstances().stream()
                             .map(Object::getClass)
                             .collect(Collectors.toSet());
                }
                public boolean getPrettyPrint() {
                    return prettyPrint;
                }
                public void setPrettyPrint(boolean b) {
                    prettyPrint = b;
                }
            })
            .initConfig()
            .initScanner();

        rc.packages(io.swagger.jaxrs.listing.ApiListingResource.class
                      .getPackage().getName());
    }
{% endhighlight %}

What this does is register only the classes of the REST resources that you have already explicitly
registered with the ResourceConfig. There is no classpath scanning, no global variables, and no guessing
about what is being registered and what is not.

Sometimes, having more code that is easy to follow is preferable to having no code that requires
deep understanding of multiple frameworks to follow what is happening.

Speaking of having lots of code, here are the swagger dependencies:

{% highlight xml %}

    <dependency>
        <groupId>io.swagger</groupId>
        <artifactId>swagger-annotations</artifactId>
        <version>${swagger.version}</version>
    </dependency>
    <dependency>
        <groupId>io.swagger</groupId>
        <artifactId>swagger-core</artifactId>
        <version>${swagger.version}</version>
    </dependency>
    <dependency>
        <groupId>io.swagger</groupId>
        <artifactId>swagger-jaxrs</artifactId>
        <version>${swagger.version}</version>
    </dependency>
    <dependency>
        <groupId>io.swagger</groupId>
        <artifactId>swagger-models</artifactId>
        <version>${swagger.version}</version>
    </dependency>
{% endhighlight %}

In this case, `swagger.version` is `1.5.8` and with the above the REST schema should be available
as JSON at `/swagger.json`

Configuring swagger-ui
----------------------

To actually have the fancy, usable documentation hosted in your web server, you first need to get
the swagger-ui HTML into your project. One option is to use the HTML provided by webjars:

{% highlight xml %}

    <dependency>
        <groupId>org.webjars</groupId>
        <artifactId>swagger-ui</artifactId>
        <version>2.1.4</version>
        <scope>runtime</scope>
    </dependency>
{% endhighlight %}

Then you can construct a context handler which you can add to your handlers list:

{% highlight java %}

    public static ContextHandler buildSwaggerUI() throws Exception {
        ResourceHandler rh = new ResourceHandler();
        rh.setResourceBase(App.class.getClassLoader()
            .getResource("META-INF/resources/webjars/swagger-ui/2.1.4")
            .toURI().toString());
        ContextHandler context = new ContextHandler();
        context.setContextPath("/docs/");
        context.setHandler(rh);
        return context;
    }
{% endhighlight %}

That will expose the docs at `/docs` - pass the JSON URL to that to load your REST info: `/docs?url=%2Fswagger.json`

The code can be seen in context in the [App Runner project](https://github.com/danielflower/app-runner/blob/master/src/main/java/com/danielflower/apprunner/web/SwaggerDocs.java).