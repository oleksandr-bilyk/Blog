# F# Lazy Load with Expiration

.NET has [Lazy class](https://docs.microsoft.com/en-us/dotnet/api/system.lazy-1?view=net-5.0) .
F# has [Lazy Expression](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/lazy-expressions).
Had tried to create Lazy load of some object with expiration (e.g. configuration with refresh timeout).
Implementation had to have following features:
1. Be fully testable. Only pure functions may be tested properly.
2. Be thread-safe.
Initially I was thinking about C# class. Note that C# code in this article may contains only class signature with method's logic ommited.
```C#
public class LazyExpirable<TValue> {
  public LazyExpirable(Func<TValue> getValue, DateTime expireOn);
  public TValue GetValue();
}
```
However such C# class will not be testable because somewhere inside of `GetValue()` method we will need to call `System.DateTime.UtcNow` that makes our logic not pure and not testable.
To solve this in OOP paradigm we need to use [Dependency Injection](https://www.goodreads.com/book/show/9407722-dependency-injection-in-net).
We need to create interface that will let to get current time. Also we will have to update our LazyExpirable class signature.
Also we need to create a factory class may be used with [Dependency Injection Container](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection).
```C#
/// This interface may be injected everywhere where UTC time may be requested.
public interface IDateTimeProvider {
  DateTime GetUtcNow();
}
public sealed class DateTimeProvider {
  public DateTime GetUtcNow() => DateTime.UtcNow;
}
/// Will be created inside of the factory .
public class LazyExpirable<TValue> {
  public LazyExpirable(Func<TValue> getValue, DateTime expireOn, IDateTimeProvider dateTimeProvider);
  public TValue GetValue();
}
/// Factory class may be used with Dependency Injection Container.
public class LazyExpirableFactory {
  private readonly IDateTimeProvider dateTimeProvider;
  public LazyExpirableFactory(IDateTimeProvider dateTimeProvider){
    this.dateTimeProvider = dateTimeProvider;
  }
  public LazyExpirable<TValue> NewLazyExpirable<TValue>(Func<TValue> getValue, DateTime expireOn) =>
    new LazyExpirable(getValue, expireOn, this.dateTimeProvider);
}
```

Now let's look at functional F# signature and implementation.
```F#
let lazyWithLifecycle getValueWithLifecycle =
    let newStateLazy () = lazy(getValueWithLifecycle())
    let mutable state = newStateLazy()
    fun () ->
        let value, isLive = state.Force()
        if isLive() then value
        else
            let newState = newStateLazy()
            state <- newState
            let (valueNew: 'a, _) = newState.Force()
            valueNew
```
