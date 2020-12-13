# Lazy Load with Expiration - C# vs F#

.NET has [Lazy class](https://docs.microsoft.com/en-us/dotnet/api/system.lazy-1?view=net-5.0) .
F# has [Lazy Expression](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/lazy-expressions).
Had tried to create Lazy load of some object with expiration (e.g. configuration with refresh timeout).
Implementation had to have following features:
1. Be fully testable. Only pure functions may be tested properly.
2. Be thread-safe.
Initially I was thinking about C# class. Note that C# code in this article may contains only class signature with method's logic ommited.

Warning: I will do full SOLID decomposition and we will have many OOP classes and interfaces.
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
public sealed class DateTimeProvider: IDateTimeProvider {
  public DateTime GetUtcNow() => DateTime.UtcNow;
}
/// Will be created inside of the factory .
public class LazyExpirable<TValue> {
  public LazyExpirable(Func<TValue> getValue, TimeSpan timeSpan, IDateTimeProvider dateTimeProvider);
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
But what if lifecycle will be not just time span between DateTome.UtcNow snapshots? It would be great to have more generic way to define lifecycle. Let's define lifecycle as some object that defines if value is steal alive. Value will be created with lifecycle.
```C#
public interface IDateTimeProvider {
  DateTime GetUtcNow();
}
public interface ILifecycle {
  bool IsAlive();
}
public sealed class LifecycleByTime: ILifecycle {
  // Dependency Injection
  private readonly IDateTimeProvider dateTimeProvider;
  // Initial state.
  private readonly DateTime createdAt;
  // Lifetyme patameter.
  private readonly TimeSpan timeSpan;
  public LifecycleByTime(IDateTimeProvider dateTimeProvider, TimeSpan timeSpan){
    this.dateTimeProvider = dateTimeProvider;
    this.timeSpan = timeSpan;
    this.createdAt = dateTimeProvider.GetUtcNow();
  }
  // Is alive when not expired.
  public bool IsAlive() => this.createdAt + this.timeSpan > this.dateTimeProvider.GetUtcNow();
}
// We need factory for Dependency Injection.
public sealed class LifecycleByTimeFactory {
  // Dependency injection
  private readonly IDateTimeProvider dateTimeProvider;
  public LifecycleByTimeFactory(IDateTimeProvider dateTimeProvider){
    this.dateTimeProvider = dateTimeProvider;
  }
  public ILifecycle NewLifecycle(TimeSpan timeSpan) => new LifecycleByTime(this.dateTimeProvider, timeSpan); 
}
/// Aggrigates values and its lifecycle.
public class ValueWithLifecycle<TValue>{
  public ValueWithLifecycle(TValue value, ILifecycle lifecycle);
  public TValue Value { get; } 
  public ILifecycle Lifecycle { get; }
}
// Now Lazy may be activated with generic lifecycle.
public class LazyWithLifecycle<TValue> {
  public LazyExpirable(Func<ValueWithLifecycle<TValue>> getValueWithLifecycle);
  /// Gets value using lazy loaded value and lifecycle.
  /// Implementation is steal ommited.
  public TValue GetValue();
}
/// Factory class may be used with Dependency Injection Container.
public class LazyExpirableFactory {
  private readonly LifecycleByTimeFactory lifecycleByTimeFactory;
  public LazyExpirableFactory(LifecycleByTimeFactory lifecycleByTimeFactory){
    this.lifecycleByTimeFactory = lifecycleByTimeFactory;
  }
  private ValueWithLifecycle<TValue> NewValueWithLifecycle<TValue>(Func<TValue> getValue, TimeSpan timeSpan) => 
    new ValueWithLifecycle(getValue, lifecycleByTimeFactory.NewLifecycle(timeSpan));
  
  public LazyExpirable<TValue> NewLazyExpirable<TValue>(Func<TValue> getValue, DateTime expireOn) =>
    new LazyWithLifecycle(NewValueWithLifecycle);
}
```

Please note that C# OOP above container just type signatures and didn't contain logic itseto filllf. 
Now let's look at functional F# signature with mazimally verbose style to help developers who are not familiar with F# well.
```F#
/// Function that returns is 
type Lifecycle = unit -> bool
type GetValueWithLifecycle = 
let lazyWithLifecycle getValueWithLifecycle =
    <implementation>
```
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
