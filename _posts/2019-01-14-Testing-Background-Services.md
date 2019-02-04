---
layout: post
published: draft
title: >
    Testing Background Services and Loops with AutoResetEvent 
category: Programming
excerpt:
   A common use case for for .NET Core 2.1's `BackgroundService` (or its IHostedService interface) is to run a loop that waits for some work to do, and then sleeps for a little while. Testing them can be a slight challenge, however.
---

A common use case for for .NET Core 2.1's `BackgroundService` (or its IHostedService interface) is to run a loop that waits for some work to do, and then sleeps for a little while. It might look something like this, using a contrived example of incrementing numbers:

{% highlight csharp %}
public class MyBackgroundService : BackgroundService {
    public MyBackgroundService(List<int> workToDo) {
        _workToDo = workToDo;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
        while(!stoppingToken.IsCancellationRequested) {
            _workToDo.Add(_workToDo.LastOrDefault() + 1);

            await Task.Delay(_checkWaitTime);
        }
    }
}
{% endhighlight %}

While complex work should usually be kept somewhere else outside of the loop, it's possible that things might get hairy enough that some tests might help us out.

This presents a problem, as there's no way to control when the loop continues. It might be possible to control the wait duration, but that could lead to unpredictable tests. What we really want is a way to control when the loop continues from the outside.

## Enter AutoResetEvent

The .NET Framework has a synchronisation primitive called `AutoResetEvent` that can act as a handle on a loop. Instead of calling `await Task.Delay(TimeSpan)`, we can call `_autoResetEvent.WaitOne()`, which then allows us to decide when to continue the loop by calling `_autoResetEvent.Set()`:


{% highlight csharp %}
_autoResetEvent = new AutoResetEvent(initialState: false);
protected override async Task ExecuteAsync(CancellationToken stoppingToken) {
    while(!stoppingToken.IsCancellationRequested) {
        _workToDo.Add(_workToDo.LastOrDefault() + 1);

        _autoResetEvent.WaitOne();
    }
}
...

// some other thread
_autoResetEvent.Set();
{% endhighlight %}

This could be useful for our tests, but first lets put it behind an interface so we can switch the behaviour and make it a bit nicer.

{% highlight csharp %}
public interface IResetAwaiter
{
    Task Wait(TimeSpan duration, CancellationToken token = default(CancellationToken));
}
{% endhighlight %}

Next lets implement a timed version that will run for reals with `Task.Delay`:

{% highlight csharp %}
public class TimedResetAwaiter : IResetAwaiter
{
    public Task Wait(TimeSpan duration, CancellationToken token)
    {
        return Task.Delay(duration, token);
    }
}
{% endhighlight %}

 And then a version that we'll use for testing that allows us to `await` until

1. the test calls `Progress()` to allow the loop to continue, and
2. the service continues execution until it hits the wait task again

{% highlight csharp %}
public class ManualResetAwaiter : IResetAwaiter
{
    private readonly AutoResetEvent _manualResetEvent;
    private TaskCompletionSource<bool> _waitTask;

    public ManualResetAwaiter()
    {
        _manualResetEvent = new AutoResetEvent(false);
    }

    public Task Wait(TimeSpan duration, CancellationToken token)
    {
        _manualResetEvent.WaitOne(duration);
        _waitTask?.SetResult(true);
        return Task.FromResult(true);
    }

    public Task<bool> Progress()
    {
        _waitTask = new TaskCompletionSource<bool>();
        _manualResetEvent.Set();
        return _waitTask.Task.TimeoutAfter(TimeSpan.FromSeconds(5));
    }
}
{% endhighlight %}

`TimeoutAfter` is a a helpful little extension method that times out after waiting for the loop to complete instead of running forever:
{% highlight csharp %}
public static class TaskExtensions
{
    public static async Task<bool> TimeoutAfter(this Task task, TimeSpan duration)
    {
        var timeout = Task.Delay(duration);
        var winningTask = await Task.WhenAny(timeout, task);
        return winningTask == task;
    }
}
{% endhighlight %}

We can now update our worker implementation to use this new abstraction:

{% highlight csharp %}
public MyBackgroundService(List<int> workToDo, IResetAwaiter resetAwaiter) { 
    // set locals 
}

protected async Task ExecuteAsync(CancellationToken stoppingToken) 
{
    while(!stoppingToken.IsCancellationRequested) {
        _workToDo.Add(_workToDo.LastOrDefault() + 1);

        await _resetAwaiter.Wait(TimeSpan.FromMinutes(1), stoppingToken);
    }
}
{% endhighlight %}

At runtime, a TimedResetAwaiter can be passed in as a constructor argument via DI or `new`, and for testing we can pass in a ManualResetAwaiter that gives us control over the loop operation.

## The Test

And now we can write a test that looks like this:

{% highlight csharp %}
[Fact]
public async Task MyBackgroundService_WhenLooping_ShouldBeUnderMyControl()
{
    var awaiter = new ManualResetAwaiter();
    var work = new List<int>();
    var backgroundService = new MyBackgroundService(work, awaiter);

    await backgroundService.StartAsync();

    Assert.Equal(new List<int>() { 1 }, work);
    
    Assert.True(await awaiter.Progress());
    Assert.Equal(new List<int>() { 1, 2 }, work);

    Assert.True(await awaiter.Progress());
    Assert.Equal(new List<int>() { 1, 2, 3}, work);
}
{% endhighlight %}


## Conclusion

This technique can be useful for controlling any loop that depends on waiting between periods of work. 

Pulling out the wait to its own abstraction means that you could also substitute more advanced wait strategies without affecting the accompanying tests.