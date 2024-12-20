---
title: "Data Specifications & Trustworthy-by-Construction AI Agents"
date: 2024-12-20
layout: post
---

This is a bit delayed but I had a great time attending [Onward!](https://2024.splashcon.org/track/splash-2024-Onward-papers) and presenting work on data-specifications ([paper](https://dl.acm.org/doi/pdf/10.1145/3689492.3690054) and [video](https://www.youtube.com/live/-Br66SUjsdQ?si=fS5_O8CmxjDtmQph&t=20637)). This work was exciting to me for a variety of reasons. The most direct, and the focus of the paper, is the dire need for better data specification tools. I hear so many stories of bugs and major issues caused by seemingly simple data specification issues -- a misspelled property in a JSON format, missing validation/sanitization on a string that ends up in a SQL query, misinterpreting the units of a numeric field, and so on. These are simple issues but, without the right tools, they are easy make and inevitably one sneaks through. 

## Data Specifications
The problem of data-specification and quality is what first motivated this work, and a simple example of the type of situation we wanted to address, is some code like the following JSON data coming from a RESTful style weather API:

```
const Forecast_Example = {
    "low": 28,
    "high": 34,
    "windSpeed": "5 to 10 mph",
    "windDirection": "N",
    "shortForecast": "Showers And Thunderstorms",
    "forecastPercent": 40,
    "units": "si"
};
```

This example illustrates many common challenges in this space:
- The `windSpeed` field is a string but there is actually a fixed format that (implicitly) this data is in.
- The numeric fields `low` and `high` are just numbers and their correct interpretation depends on knowing about, and using, a separate field (`units`).
- Simple typos, like `"n"` instead of `"N"` in the `windDirection` field, can cause downstream issues.
- Although not a bit issues here, many values of interest (say a integer larger than 2^53) are not representable in JSON.

To address these issues, we developed a tool called _Bosque Object Notation_ (BsqON) that allows developers to specify the structure of a data-type in an expressive & formal language, that is polyglot, and can be used to provide validation and versioning support. A BsqON spec for the `Forecast` data type might look like:

```
enum Units { us, si }
enum CompassDirection { North, South, East, West }

type Fahrenheit = Int;
type Mph = Nat;

entity TempRange { field low: Fahrenheit; field high: Fahrenheit; }
entity WindSpeedRange { field min: Mph; field max: Mph; }

type Percentage = Float & {
    validate 0.0<Percentage> <= $value;
    validate $value <= 100.0<Percentage>;
}

const ForecastOption: CRegex = /'Showers'|'Thunderstorms'|'Snow'|'Fog'/c;
type ForecastDetail = CString of /${ForecastType}(' and '${ForecastType})*/c;

entity Forecast {
    field temp: TempRange;
    field windSpeed: WindSpeedRange;
    field windDirection: CompassDirection;
    field shortForecast: ForecastDetail;
    field forecastPercent: Percentage;
    field units: Units = Units#us;
}
```

In this example we can see that we have replaced the sentinel strings with actual enums. In addition the numeric values are explicitly connected with their semantic unit information and the percentage value is set with the appropriate range constraints. Further, even string values with complex structures, like the `shortForecast` field, can be specified with regular expressions. Of course all of these properties and constraints can be mechanically validated at runtime and surfaced in IDE tooling!

## Literal Notation
The ability to precisely describe data schemas and constraints is a big step and enables us to support a range of high-value scenarios -- including automatic validation of inputs to services, sanitizing data as it is ingested into a system, generating friendly API SDKs, fuzzing inputs for testing, and so on. We can also automatically generate parsers/serializes for various data formats, like JSON or OpenAPI, as well. 

However, there is are two major pain-point in these formats:
- There are many "primitive" types that cannot be natively encoded in JSON (e.g., 64-bit integers, decimals, UUIDs, etc.) and thus they are transformed to/from strings. This is not only inefficient but also error-prone.
- Representing values that are unions (or subtypes) is also not natively supported. Thus, these values must be tagged using some convention, like tag fields or envelopes, and users need to agree on and remember to use them!

With this in mind the BsqON system also introduces a new literal format that allows for fully fidelity encoding or any BsqON value and has a clean syntax for subtypes. This format also provides a easy human readable/authorable syntax with shorthands to simplify common situations e.g. eliminating redundant field/type names.

An example of a forecast value in the BsqON literal format (with the default level of inline meta-data) would look like:
```
Forecast{
    TempRange{ 33<Fahrenheit>, 52<Fahrenheit> },
    WindSpeedRange{ max=2<Mph>, min=0<Mph> },
    CompassDirection.North,
    'Snow'<ForecastDetail>,
    10.0<Percentage>
}
```

As this example shows this format provides a clean encoding for the data, with some nice support for reordering field declarations with explicit names or just using positional arguments and allowing default values. The ability to explicitly call out the type of a value, like `10.0<Percentage>`, helps catch many types of errors and makes the format (mostly) self-describing.

In some cases this explicitness can be a bit verbose and most of this information is actually redundant, given the type information in the BsqON schema, so we allow compact forms that with a more JSON-like shorthand. For example, the same forecast value in a compact form be written as:

```
Forecast {
    {33, 52},
    {max=2, min=0},
    CompassDirection.North,
    'Snow',
    10.0f
}
```
In this representation almost all of the explicit meta-data overhead has been removed! 

Finally, as a critical feature, this format is also very efficient to parse! In fact it has been explicitly designed to be parsable in a single (streaming) LL(1) pass. This ensures that we can efficiently parse large messages and provide good error messages in the case of a parse failure.

## High-Value Scenarios
Now that we have a semantically rich data specification format it is possible to create tools that simplify major pain-points of service based software development. 

### Fuzz Test Generation
One of the most common issues in software development is the lack of good test coverage. This is particularly true for end-to-end (or integration) testing of APIs. Unfortunately, writing high coverage tests is often time consuming and tedious while simple automated test generation tools often produce low-value tests that have limited coverage.

Using the BsqON format we can generate high-value tests that have high coverage. We can do this generating partition coverage tests from the specification -- where we sample satisfying values for each primitive and combine them based on the constraints of the specification. Given the structure of the specifications, and the high-failure exposure rate of [_all-pairs_ testing](https://ieeexplore.ieee.org/document/1321063), we can expect these to produce a high-value set of tests. Further, as the specs are reducible to SMT formulas, we can know that we can generate even very specific values (like [section 6.3](https://dl.acm.org/doi/pdf/10.1145/3689492.3690054) in the paper where order ids must be unique)! 

### Mock Creation
A similar problem is mock creation when testing a service that depends on other services. For example we might be writing a sailboat rental service that calls our weather API to see if the weather is safe for sailing. In order to test the service we need to either:
- Test it against the live weather API -- in which case we end up with flakey and un-reproducible tests.
- Standup a separate isolated/deterministic weather API service -- in which case we need to maintain a separate service and ensure it is always up-to-date with the real service.
- We need to write mocks for the weather API calls -- which is a time-consuming, error prone, and tedious process.

In this case we can use the BsqON specification to _automatically_ generate a mock service that, whenever a request is made, generate a response that is valid according to the specification. Again since the specification is reducible to SMT formulas we can reliably generate valid data, even when the specification is complex. We can also vary the response to test edge cases, use LLMs to sample over normal values, and record results for later deterministic testing.

For example on the weather API we might get a airport code like `ABQ` and we can generate a random response like:
```
Forecast {
    {0, 1},
    {0, 1},
    CompassDirection.North,
    'Snow',
    10.0f
}
```

Or to test an edge case of, very stormy, on a otherwise natural input we might generate a response like:
```
Forecast {
    {60, 80},
    {max=0, min=0},
    CompassDirection.North,
    'Showers and Thunderstorms',
    100.0f
}
```

### Data Provenance and Exposure Control
The final application that this type of data-specification enables that we have been exploring is managing data provenance and exposure control. In many cases we need to know where data came from, how it was transformed, and who has access to it. This is particularly true in regulated industries, like healthcare, finance, and government, where data privacy and security are paramount.

By generating the APIs for services from the BsqON specification we can automatically track the provenance of data as it flows through the system. This allows us to know where data came from, how it was transformed, and who has access to it. We can ensure that policies such as encryption at rest, logging of sensitive data, and that data is only use for authorized purposes (e.g. a 2FA phone number is not used for marketing) are enforced. This can be used to generate audit logs, ensure compliance with data regulations, and provide transparency to users about how their data is being used.

## Trustworthy-by-Construction AI Agents
This all brings us to the final, and maybe most important, application of this work -- trustworthy-by-construction AI agents. As AI agents become more and more prevalent in our lives, and as they are used in more and more critical applications, it is critical that we can trust them. For example consider the following API:

```
%** Allow AI agent to make payments on your behalf **%​
api transfer(amt: USD, from: Account, to: Account): Bool;
```

Would you trust a LLM agent to generate actions that use this API? Even if, based on testing and training, it was able to correct API 99.9% of the time? 

The answer is probably no. The reason is that the small percentage of the time that it fails could be catastrophic. It could be that the agent is hacked or that the agent is just bad luck. But either way the result is the same -- and hopefully not too painful to recover from.

Instead of simply saying failure is inevitable and, simply that our AI agents are probably correct most of the time, we can use the BsqON specification to ensure that the AI agent is _trustworthy-by-construction_ and we can semantically guarantee good behaviors. Consider the revised API:

```
const paymentLimit: USD = ...;

api transfer(amt: USD, from: Account, to: Account): Bool
    requires from.balance >= amt;
    requires amt <= paymentLimit;
``` 

In this API we are able to specify that the agent must have enough money in the account to make the payment and that the payment must be below a certain limit. This is a simple example but we can imagine much more complex requirements -- say that the `amt` and payee match from a list of approved expenses.

Even better BsqON also support soft, best practice constraints, like prefering that meetings are scheduled during working hours, that can be used to guide the AI agent to make better decisions. For example:

```
function inWorkingHours(time: TZDateTime, user: UserID) Bool {…}

%** Allow AI agent to schedule meetings **%
api scheduleMeeting(time: TZDateTime, attendees: UserID[]): Bool 
    requires softchk users.allOf(fn(usr) => inWorkingHours(usr, time));
;
```

These features provide multi-faceted value. The fact that these constraints are syntactically explicit in the APIs means that the agent can see and take them into consideration during generation. We can use them to improve inference-time compute, and, of course, we can then dynamically (or statically) ensure that the agents plans satisfy these constraints!

Thus this work provides a key component for the future of AI agents by enabling us to make them trustworthy-by-construction with semantically sound behavioral guarantees _beyond_ just providing statistical assurances that an output is probably correct!
