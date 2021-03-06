# Hello World

Based on Orleans [Hello World](https://dotnet.github.io/orleans/Samples-Overview/Hello-World) sample.

## Overview

![SequenceDiagram](https://raw.githubusercontent.com/OrleansContrib/Orleans.Activities/master/docs/HelloWorld/SequenceDiagram-Overview.png)

## Interface

IHello is the same.

```c#
public interface IHello : IGrainWithGuidKey
{
  Task<string> SayHello(string greeting);
}
```

## Grain

Before the grain, you have to define 3 things: Grain State, Workflow Interface and Workflow Callback Interface

### Grain State

Workflows always have a state. Even if they never persist it. You can use the `WorkflowState` base class or implement the `IWorkflowState` interface.

```c#
public class HelloGrainState : WorkflowState
{ }
```

### Workflow Interface

These are the operations that the grain calls on the workflow, these operations should __NOT__ be the same as the public grain interface methods (see `IHello`)!

There are 2 restrictions on the methods:

* must have 1 parameter, with type `Func<Task<anything>>` or `Func<Task>` (executed when the workflow accepts the request)
* the return type must be `Task` or `Task<anything>`

```c#
public interface IHelloWorkflowInterface
{
  Task<string> GreetClient(Func<Task<string>> clientSaid);
}
```

### Workflow Callback Interface

These are the operations that the workflow calls back on the grain.

There are 2 restrictions on the methods:

* can have max. 1 parameter with any type
* the return type must be `Task<Func<Task<anything>>>` or `Task<Func<Task>>` (executed when the workflow accepts the response)

```c#
public interface IHelloWorkflowCallbackInterface
{
  Task<Func<Task<string>>> WhatShouldISay(string clientSaid);
}
```

### Grain

The class definition, where we define the TGrainState, TWorkflowInterface and TWorkflowCallbackInterface type parameters.

__NOTE:__ The grain must implement (if possible explicitly) the TWorkflowCallbackInterface interface (see `IHelloWorkflowCallbackInterface`).

```c#
public sealed class HelloGrain : WorkflowGrain<HelloGrainState, IHelloWorkflowInterface, IHelloWorkflowCallbackInterface>,
  IHello, IHelloWorkflowCallbackInterface
```

Constructor, in this example without Dependency Injection, just define the activity factory and leave the workflow definition identity null.

```c#
public HelloGrain()
  : base((wi) => new HelloActivity(), null)
{ }
```

Optionally see what happens during the workflow execution with tracking, this can be left out, there is a default empty implementation in the base class.

```c#
protected override IEnumerable<object> CreateExtensions()
{
  yield return new GrainTrackingParticipant(GetLogger());
}
```

A mandatory (boilerplate) implementation of the unhandled exception handler. Because workflows can run in the backround after an incoming call returns the result, we can't propagate back exceptions after this point. Workflow will by default abort in case of unhandled exception, depending on the `Parameters` property.

```c#
protected override Task OnUnhandledExceptionAsync(Exception exception, Activity source)
{
  GetLogger().Error(0, $"OnUnhandledExceptionAsync: the workflow is going to {Parameters.UnhandledExceptionAction}", exception);
  return Task.CompletedTask;
}
```

The public grain interface method, that does nothing just calls the workflow's only affector operation. A normal grain can store data from the incoming message in the state, call other grains, closure the necessary data into the parameter delegate. After the await it can build a complex response message based on the value the workflow returned and the grain state, or any other information.

The parameter delegate executed when the workflow accepts the incoming call.

It also shows how to implement idempotent responses for the incoming calls. In the repeated case, the parameter delegate won't be executed!

```c#
public async Task<string> SayHello(string greeting)
{
  try
  {
    return await WorkflowInterface.GreetClient(() =>
      Task.FromResult(greeting));
  }
  catch (RepeatedOperationException<string> e)
  {
    return e.PreviousResponseParameter;
  }
}
```

This is the explicit implementation of the effector interface's only method, that does nearly nothing. A normal grain can modify the grain's State, call other grain's operations or do nearly anything a normal grain method can.  

The return value delegate executed when the workflow accepts the outgoing call's response.

```c#
Task<Func<Task<string>>> IHelloWorkflowCallbackInterface.WhatShouldISay(string clientSaid) =>
  Task.FromResult<Func<Task<string>>>(() =>
    Task.FromResult(string.IsNullOrEmpty(clientSaid) ? "Who are you?" : "Hello!"));
```

## Workflow / Activity

And see the Workflow. It calls back the grain, and returns the response to the grain at the end.

![HelloActivity.xaml](https://raw.githubusercontent.com/OrleansContrib/Orleans.Activities/master/docs/HelloWorld/HelloActivity.png)

That's all. Ctrl+F5, and it works.

## Details

If you want to dig deep into the source and understand the detailed events in the background, this sequence diagram can help (this is not a completely valid diagram, but displaying every asnyc details, even the AsyncAutoResetEvent idle-queue, this would be 2 times bigger).

![SequenceDiagram](https://raw.githubusercontent.com/OrleansContrib/Orleans.Activities/master/docs/HelloWorld/SequenceDiagram-Details.png)
