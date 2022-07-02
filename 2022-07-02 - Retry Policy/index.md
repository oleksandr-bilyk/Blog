# F# Retry policy


Golden standard for retry logic according to "Service Reliability Engineering" (Google SRE) book has following properties:
1. Exponential timeout. Fibonacci series is good for testing.
2. Random shift to each timeout to avoid throttling.
3. Infinite cout of retried limited by end time.

Unfortunately [Polly](https://github.com/App-vNext/Polly) doens't support combination of infinite and limited by time. 

Let's develop fibonacci/exponential infinit sequence generator.
```fsharp
/// Infinite number generator that starts as fibonacci generator but returns values not bigger than max value.
let buildFibonacciNumbersWithMaxGenerator (max: int) =
    let mutable a, b = 0, 1

    let mutable strategy =
        fun () ->
            let newValue = a + b
            a <- b
            b <- newValue
            newValue

    fun () ->
        let result = strategy ()

        if result <= max then
            result
        else
            strategy <- fun () -> max
            strategy ()
```
Let's write unit test for fibonacci generator
```fsharp
[<Fact>]
let testBuildFibonacciNumbersGenerator() =
    let generator = buildFibonacciNumbersWithMaxGenerator 40
    let actual = [ 1 .. 10 ] |> List.map (fun _ -> generator())
    let expected = [ 1; 2; 3; 5; 8; 13; 21; 34; 40; 40 ]
    Assert.Equal<int list>(expected, actual)
```

Now lets develop retry policy as pure function.

```fsharp
type RetrySideEffects =
    { GetTimeNow: unit -> DateTime
      GetDelay: unit -> TimeSpan
      Sleep: TimeSpan -> Async<unit> }

let retry<'TOk, 'TError>
    (sideEffects: RetrySideEffects)
    (action: unit -> Async<Result<'TOk, 'TError>>)
    (endTime: DateTime)
    =
    let rec iteration index =
        async {
            match! action () with
            | Ok r -> return Ok r
            | Error e ->
                let now = sideEffects.GetTimeNow()
                if now <= endTime then
                    let delay = sideEffects.GetDelay()
                    let next = now + delay

                    if next <= endTime then
                        do! sideEffects.Sleep delay
                        return! iteration (index + 1)
                    else
                        return Error e
                else
                    return Error e
        }

    iteration 0

```
And here is how to test retry policy
```fsharp
type RetryState = 
    | CallAction of Result<string, string>
    | GetTimeNow of DateTime
    | GetDelay of TimeSpan

let buildRetryTest endTime (callLogSeq: RetryState seq) expectedResult =
    let callLog = callLogSeq.GetEnumerator()
    let getTransaction () =
        if callLog.MoveNext() then callLog.Current
        else raise (InvalidOperationException("CallAction expected."))

    let getTimeNow () =
        match getTransaction() with
        | GetTimeNow t -> t
        | _ -> 
            raise (InvalidOperationException("GetTimeNow expected."))
    let getDelay () = 
        match getTransaction() with
        | GetDelay timeSpan ->
            timeSpan
        | _ ->
            raise (InvalidOperationException("GetTimeNow expected."))
    let action oc: Async<Result<string, string>> = 
        match getTransaction() with
        | CallAction r -> async { return r }
        | _ -> 
            raise (InvalidOperationException("CallAction expected."))
    let resultActual = 
        retry 
            { GetTimeNow = getTimeNow
              GetDelay = getDelay
              Sleep = (fun _ -> async { return () }) }
            action 
            endTime |> Async.RunSynchronously
    Assert.True((expectedResult = resultActual))
    Assert.True((callLog.MoveNext() = false), "End of transaction log.")

[<Fact>]
let ``Immediate result`` () =
    buildRetryTest 
        (DateTime(100L))
        (seq {
            Ok "Hello" |> CallAction
            //GetTimeNow DateTime.Now
        })
        (Ok ("Hello"))

[<Fact>]
let ``Result from second attempt`` () =
    buildRetryTest 
        (DateTime(100L))
        (seq {
            CallAction (Error "Error1")
            GetTimeNow (DateTime(10L))
            GetDelay (TimeSpan(0L))
            CallAction (Ok "ResultFromSecondAttempt")
        })
        (Ok ("ResultFromSecondAttempt" ))

[<Fact>]
let ``Result from third attempt`` () =
    let d1 = TimeSpan.FromTicks(12L)
    buildRetryTest 
        (DateTime(100L))
        (seq {
            CallAction (Error "Error1")
            GetTimeNow (DateTime(10L))
            GetDelay(TimeSpan(0L))
            CallAction (Error "Error2")
            GetTimeNow (DateTime(20L))
            GetDelay(TimeSpan(0L))
            CallAction (Ok "Hello")
        })
        (Ok ("Hello"))

[<Fact>]
let ``Timeout after second attempt`` () =
    buildRetryTest 
        (DateTime(100L))
        (seq {
            CallAction (Error "Error1")
            GetTimeNow (DateTime(10L))
            GetDelay(TimeSpan(0L))
            CallAction (Error "Error2")
            GetTimeNow (DateTime(99L))
            GetDelay(TimeSpan(10L))
        })
        (Error "Error2")
```
