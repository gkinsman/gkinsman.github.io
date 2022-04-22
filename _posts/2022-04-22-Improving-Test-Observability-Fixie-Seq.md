---
layout: post
published: draft
title: >
  Improving Test Observability with Fixie+Seq
category: Programming
excerpt: For years now it's been possible to forward logs from Serilog to xUnit's `ITestOutputHelper`, but in doing so we lose the structure that makes structured logs great. How can we get the best of both worlds - the advantages of `ITestOutputHelper` and the analysis capabilities of a local structured log viewer like [Seq](https://datalust.co/)?
---

For years now it's been possible to forward logs from Serilog to xUnit's `ITestOutputHelper`, but in doing so we lose the structure that makes structured logs great. How can we get the best of both worlds - the advantages of `ITestOutputHelper` and the analysis capabilities of a local structured log viewer like [Seq](https://datalust.co/)?

### Where are we now?

Using [Serilog.Sinks.Xunit](https://github.com/trbenning/serilog-sinks-xunit), you can view application logs emitted during a test right in your IDE's test viewer:

[code](https://github.com/gkinsman/SeqFixieSample/blob/master/test/SeqFixieSample.XunitTests/ApiTestContext.cs#L26)
![](/images/xunit-outputhelper-logs.png)

This is super useful for debugging problems in tests, as we can view just the logs for the failing test without any other noise getting in the way. However, we miss out on one of the prime benefits of structured logging: the structure!

Since this is Serilog, we can plumb the test logs through to a local structured log viewer, like Seq:

[code](https://github.com/gkinsman/SeqFixieSample/blob/master/test/SeqFixieSample.XunitTests/ApiTestContext.cs#L25)
![](/images/seq-logs-xunit.png)

Fantastic! We can now see and query the structure of the test logs. But we've lost something important here - can you spot it?

We can no longer isolate the application logs for a particular test, so if there's one test that's hanging for example, we won't be able to view just the logs leading up to that hang. We're writing the log output to xUnit's `ITestOutputHelper`, but that's only useful when viewing logs in a test runner - Seq has no idea which test these logs came from.

Additionally, xUnit unfortunately doesn't currently have a way to run code _around_ a test, requiring you to add code within tests to add this metadata, which I think we can agree isn't a great state of affairs.

## Enter Fixie

[Fixie](http://fixie.github.io/) is an alternative test runner for .NET with an 'emphasis on low ceremony defaults and flexibile customization'.

It inverts the way our tests are run, so that instead of xUnit calling our `[Fact]`s, we tell fixie how to find and/or execute our tests.

Fixie's extensibility is in customising _discovery_ and _execution_. Given that I'm already pretty happy with how Fixie _discovers_ tests (public instance methods on classes ending with `Tests`), we're just going to change how it executes them (fixie will auto-discover implementations of this interface):

{% highlight csharp %}
public class AppTestProject : ITestProject
{
    public void Configure(TestConfiguration configuration, TestEnvironment environment)
    {
        configuration.Conventions.Add<DefaultDiscovery, SingleConstructionExecution>();
    }
}
{% endhighlight %}

### Execution

So let's now walk through the code of the custom execution bit by bit - the whole app can be found [here](https://github.com/gkinsman/SeqFixieSample), and the code for this section is [here](https://github.com/gkinsman/SeqFixieSample/blob/master/test/SeqFixieSample.FixieTests/FixieTestProject.cs#L18).

1. Implement `IExecution` which gives us the discovered `TestSuite`, and we give Fixie a Task back to execute.

{% highlight csharp %}
class SingleConstructionExecution : IExecution
{
    public async Task Run(TestSuite testSuite)
    {
{% endhighlight %}

{:start="2"}
2. I use a shared context, but you could create this for each test if you wanted!

{% highlight csharp %}
var context = new ApiTestContext();
{% endhighlight %}

{:start="3"}
3. We're a little too far away from the app execution context here so we can't use `LogContext`. Instead, we'll add a [simple enricher](https://github.com/gkinsman/SeqFixieSample/blob/master/test/SeqFixieSample.FixieTests/MapEnricher.cs) that lets us `Set` props on the global logger, which will enrich every log event. We also override `Log.Logger` here with our own so that we can customise it just for tests, and to add this enricher.

{% highlight csharp %}
var mapEnricher = new MapEnricher();

Log.Logger = new LoggerConfiguration()
    ...
    .Enrich.With(mapEnricher)
    ...
{% endhighlight %}

{:start="4"}
4. Create a test run guid and add it to the enricher so that we can filter by `TestRunId` later on.
5. Log an event to bookend the test run

{% highlight csharp %}
var testRunId = Guid.NewGuid().ToString();
mapEnricher.Set("TestRunId", testRunId);

Log.Information("Beginning test run {TestRunId}", testRunId);
{% endhighlight %}

{:start="6"}    
6. I'm allowing test classes to accept an optional context (the `ApiTestContext` from before). Fixie provides a helpful abstraction over `Activator.CreateInstance` in `Construct`, so we use that.

{% highlight csharp %}
try {
    foreach(var testClass in testSuite.TestClasses) {
        object instance = testClass.Type.GetConstructors()
            .Any(ctor => ctor.GetParameters().Any())
            ? testClass.Construct(context)
            : testClass.Construct();

{% endhighlight %}

{:start="7"}
7. So now let's add `TestName` and `TestClass` to the enricher, and wrap the test execution in a `SerilogTimings` operation so that we can keep track of our test durations over time.

{% highlight csharp %}
foreach(var test in testClass.Tests) {
    mapEnricher.Set("TestName", test.Name);
    mapEnricher.Set("TestClass", testClass.Type.Name);
    using (Operation.Time("Executing test {TestName}", test.Name)) {
        await test.Run(instance);
    }
}
{% endhighlight %}

{:start="8"}
8. Cleanup! Bookend the test run, dispose the context, flush the logger.

{% highlight csharp %}
}
finally {
    Log.Information("Cleaning up test context");
    context.Dispose();
    Log.CloseAndFlush();
}
{% endhighlight %}

## What do you get?

The ability to slice and dice the entire log output of your app by TestName/TestRunId/TestClass, and no `SynchronizationContext` shenanigans!:

![](/images/seq-logs-fixie-testrun.png)

![](/images/seq-fixie-logs-dash.png)
