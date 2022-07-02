# F# Retry policy


Golden standard for retry logic according to "Service Reliability Engineering" (Google SRE) book has following properties:
1. Exponential timeout. Fibonacci series is good for testing.
2. Random shift to each timeout to avoid throttling.
3. Infinite cout of retried limited by end time.

Unfortunately [Polly](https://github.com/App-vNext/Polly) doens't support combination of infinite and limited by time. 

```fsharp
module RetryPolicy

open System

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
