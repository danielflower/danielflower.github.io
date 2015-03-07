---
layout: post_page
title: A Hamcrest Matcher for Lists of Lines
---

I have been writing a build plugin which writes a bunch of logging while it runs. I wanted to be able to take the output
from a test run (which is a `List<String>`) and make assertions about what the output does or does not contain.

You may be sceptical about whether or not log output should be tested. In this case, the log output is basically the UI of
the plugin and it is what people will use to troubleshoot and see what is going on. It's important, and logging the correct
information at the right time can be difficult, so yes! It should be unit tested.

I alway use Hamcrest for assertions due to the fact that the intention of the tests can be so clear when it is used well,
assertion error messages are clear, and the matchers are composable - more on compasability later. But I must admit I've
always hesitated to write my own custom matchers because they just seem a _little_ bit too bulky with too much boiler plate.
That is until I discovered the `TypeSafeDiagnosingMatcher<T>` base matcher that is built into Hamcrest. The `TypeSafe` part
of the name means it uses generics; the `Diagnosing` bit means it is easy to write error messages.

By extending this class you can not just quickly create a matcher, but crucially when matching fails the error message
can be created at exactly the point you want it to be, i.e. in the match method. The two methods to override are:

{% highlight java %}

    protected boolean matchesSafely(T actual, Description mismatchDescriptor);

    public void describeTo(Description description) {
{% endhighlight %}

As with normal Hamcrest matchers, you pass in expected values (or matchers) into the constructor, use `describeTo` to
explain what you are trying to match, and then compare the expected value to the `actual` value passed into the `matchesSafely`
method.

In my case, I wanted to be able to make assertions such as "from this list of Strings, only 1 contains the phrase
'Completed successfully'". From the test, it looks a bit like this:

{% highlight java %}

    List<String> output = doStuff();
    assertThat(output, hasOneLineThat(containsString("Completed Successfully")));
{% endhighlight %}

I also had tests checking that certain strings are never printed, or printed exactly twice, etc. So my matcher needed to
know how many times something should occur, and what the matching criteria is. In the example above, I just wanted a line
that contained a string - but I could also want an exact match, or a string starting with a match, or a regular expression
match, or any other Hamcrest String matcher. This is what I meant above when I said Hamcrest is composable: I can use any
`Matcher<String>` to compare lines. So the constructor of my matcher looks like this:

{% highlight java %}

    private ExactCountMatcher(Matcher<String> stringMatcher, int expectedCount) {
        this.stringMatcher = stringMatcher;
        this.expectedCount = expectedCount;
    }
{% endhighlight %}

The static `hasOneLineThat` method just takes a String matcher and passes that plus the value `1` to the constructor. In the `matchesSafely`
method I simply invoke the stringMatcher on each line of the output, and count how many times the matcher matches a line. If it
isn't the number of expected occurences, then the error is written to the `mismatchDiscriptor`. The full class should be easy enough to follow:

{% highlight java %}

    import org.hamcrest.Description;
    import org.hamcrest.Factory;
    import org.hamcrest.Matcher;
    import org.hamcrest.TypeSafeDiagnosingMatcher;

    import java.util.List;

    public class ExactCountMatcher extends TypeSafeDiagnosingMatcher<List<String>> {
        private final Matcher<String> stringMatcher;
        private int expectedCount;

        private ExactCountMatcher(Matcher<String> stringMatcher, int expectedCount) {
            this.stringMatcher = stringMatcher;
            this.expectedCount = expectedCount;
        }

        @Override
        protected boolean matchesSafely(List<String> items, Description mismatchDescriptor) {
            int count = 0;
            for (String item : items) {
                if (stringMatcher.matches(item)) {
                    count++;
                }
            }
            boolean okay = count == expectedCount;
            if (!okay) {
                mismatchDescriptor.appendText("was matched " + count + " times in the following list:");
                String separator = String.format("%n") + "          ";
                mismatchDescriptor.appendValueList(separator, separator, "", items);

            }
            return okay;
        }

        @Override
        public void describeTo(Description description) {
            description.appendDescriptionOf(stringMatcher).appendText(" " + expectedCount + " times");
        }

        @Factory
        public static Matcher<? super List<String>> hasNoLinesThat(Matcher<String> stringMatcher) {
            return new ExactCountMatcher(stringMatcher, 0);
        }

        @Factory
        public static Matcher<? super List<String>> hasOneLineThat(Matcher<String> stringMatcher) {
            return new ExactCountMatcher(stringMatcher, 1);
        }

    }
{% endhighlight %}

I complained about the bulkiness of matchers before - and this is 50 lines which does seem a lot - but it is all useful
code so I'm not so offended. Having the `mismatchDescriptor` is the key thing here as it has made writing the reasons
for failing very simple.