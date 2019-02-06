---
layout: post
published: draft
title: >
    Testing Background Services and Loops with AutoResetEvent 
category: Programming
excerpt:
   A common use case for for .NET Core 2.1's `BackgroundService` (or its IHostedService interface) is to run a loop that waits for some work to do, and then sleeps for a little while. Testing them can be a slight challenge, however.
---

<small>
#### Updates:
- 6 Feb 2019 - Fixed a bug with ManualResetAwaiter in which it wouldn't accurately wait for loops to wrap around. I've reworked it slightly to reduce the number of cases it needs to handle.
</small>

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

 And then a version that we'll use for testing that allows us to `await` until the service continues execution and reaches the `Wait` call again. The `Progress(TimeSpan)` method will return false if the work doesn't complete before the timeout, which we can assert on. 

{% highlight csharp %}
public class ManualResetAwaiter : IResetAwaiter {
	AutoResetEvent _resetEvent;
	SemaphoreSlim _waitEvent;
	TaskCompletionSource<bool> _firstHit;
	
	public ManualResetAwaiter()
	{
		_resetEvent = new AutoResetEvent(false);
		_waitEvent = new SemaphoreSlim(1);
		_firstHit = new TaskCompletionSource<bool>();
	}

	public async Task<bool> Progress(TimeSpan timeout)
	{
		_resetEvent.Set();
		return await _waitEvent.WaitAsync(timeout);
	}

	public Task Wait(TimeSpan duration, CancellationToken token)
	{
		if(_firstHit.Task.IsCompleted) _waitEvent.Release();
		if(!_firstHit.Task.IsCompleted) _firstHit.SetResult(true);
		_resetEvent.WaitOne();
		return Task.CompletedTask;
	}

	public async Task WaitFirstTime()
	{
		await _waitEvent.WaitAsync();
		await _firstHit.Task;
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

    // Wait until the service starts up and hits the Wait call the first time
    await awaiter.WaitFirstTime();

    // Continue execution and wait until Wait is hit again 
    Assert.True(await awaiter.Progress());
    Assert.Equal(new List<int>() { 1 }, work);
    
    // And again...
    Assert.True(await awaiter.Progress());
    Assert.Equal(new List<int>() { 1, 2 }, work);

    // And again...
    Assert.True(await awaiter.Progress());
    Assert.Equal(new List<int>() { 1, 2, 3}, work);
}
{% endhighlight %}


## Conclusion

This technique can be useful for controlling any loop that depends on waiting between periods of work. 

Pulling out the wait to its own abstraction means that you could also substitute more advanced wait strategies without affecting the accompanying tests.

