# Async methods

Async methods pattern is reactive programming replacement for `Tasks`, `Futures` or any other .NET async patterns.

## Guide

### General

1. Async methods should return `IObservable<TResult>`.
2. If no return value is produced, use `IObservable<Unit>` as return type and `Unit.Default` as return value.
3. `IObservable<TResult>` returned by async method should always trigger either **one** `OnNext`+`OnCompleted` (in case of success) or `OnError` (in case of failure) for every subscription.
4. Subscriptions should not produce side effects.

### Implementation

1. To make sure that returned `IObservable<Result>` behaves exactly like described in previous section, use `AsyncSubject<TResult>`.
2. Return `AsyncSubject<TResult>` with operator `.AsObservable()` to make sure that subject is not available for modifications outside the method body.
3. Using `AsyncSubject<TResult>` is not necessary for async methods that just return modified result of another async method.

## Examples

### Interface

``` C#
public interface IUserService
{
    // async method with return type
    IObservable<User> GetUser(string name);
    
    // async method that only modifies result of another one
    IObservable<int> GetUserId(string name);
    
    // async method without return type
    IObservable<Unit> DeleteUser(string name);
    
    // async method with return type
    IObservable<User> CreateUser(string name);
    
    // async method that combines execution of other async methods
    IObservable<User> GetOrCreateUser(string name);
}
```

### Implementation

``` C#
public class SimpleUserService : IUserService
{
    public IObservable<User> GetUser(string name)
    {
        AsyncSubject<User> subject = new AsyncSubject<User>();
        
        // (start of async code)
        // success:
        subject.OnNext(user);
        subject.OnCompleted();
        // error:
        subject.OnError(exception);
        // (end of async code)
        
        return subject.AsObservable();
    }
    
    public IObservable<int> GetUserId(string name)
    {
        return GetUser(name).Select(user => user.Id);
    }
    
    public IObservable<Unit> DeleteUser(string name)
    {
        AsyncSubject<Unit> subject = new AsyncSubject<Unit>();
        
        // (start of async code)
        // success:
        subject.OnNext(Unit.Default);
        subject.OnCompleted();
        // error:
        subject.OnError(exception);
        // (end of async code)
        
        return subject.AsObservable();
    }
    
    public IObservable<User> CreateUser(string name)
    {
        AsyncSubject<User> subject = new AsyncSubject<User>();
        
        // (start of async code)
        // success:
        subject.OnNext(newUser);
        subject.OnCompleted();
        // error:
        subject.OnError(exception);
        // (end of async code)
        
        return subject.AsObservable();
    }
    
    public IObservable<User> GetOrCreateUser(string name)
    {
        AsyncSubject<User> subject = new AsyncSubject<User>();
        
        GetUser(name).Subscribe(user =>
        {
            subject.OnNext(user);
            subject.OnCompleted(user);
        }, exception =>
        {
            if(exception is UserNotFoundException)
            {
                CreateUser(name).Subscribe(user =>
                {
                    subject.OnNext(user);
                    subject.OnCompleted(user);
                }, subject.OnError);
            }
            else
            {
                subject.OnError(exception);
            }
        }
        
        return subject.AsObservable();
    }
}
```

## TODO

1. Support for cancellation.
2. Is `async method` a proper name for this pattern?
