---
title: "Data Specifications & Trustworthy-by-Construction AI Agents"
date: 2024-12-18
layout: post
---

This is a bit delayed but I had a great time attending [Onward!](https://2024.splashcon.org/track/splash-2024-Onward-papers) and presenting work on 
data-specifications ([paper](https://dl.acm.org/doi/pdf/10.1145/3689492.3690054) and [video](https://www.youtube.com/live/-Br66SUjsdQ?si=fS5_O8CmxjDtmQph&t=20637)). 
This work was exciting to me for a variety of reasons. The most direct, and the focus of the paper, is the dire need for better data specification tools. 
I hear so many stories of bugs and major issues caused by seemingly simple data specification issues -- a misspelled property in a JSON format, missing 
validation/sanitization on a string that ends up in a SQL query, misinterpreting the units of a numeric field, and so on. These are simple issues but, without 
the right tools, they are easy make and inevitably one sneaks through. 

## Data Specifications
Th problem of data-specification and quality is what first motivated this work, and a simple example of the type of situation 
we wanted to address, is some code like the following JSON data coming from a RESTful style weather API:

```
const Forecast_Example = {
    "low": 28,
    "high": 34,
    "windSpeed": "5 to 10 mph",
    "windDirection": "N",
    "shortForecast": "Showers And Thunderstorms",
    "forecastPercent": 40,
    "units": "si"
};
```

This example illustrates many common challenges in this space:
- The `windSpeed` field is a string but there is actually a fixed format that (implicitly) this data is in.
- The numeric fields `low` and `high` are just numbers and their correct interpretation depends on knowing about, and using, a separate field (`units`).
- Simple typos, like `"n"` instead of `"N"` in the `windDirection` field, can cause downstream issues.
- Although not a bit issues here, many values of interest (say a integer larger than 2^53) are not representable in JSON.

To address these issues, we developed a tool called _Bosque Object Notation_ (BsqON) that allows developers to specify the structure of a data-type in an expressive & formal language, that is polyglot, and can be used to provide 
validation and versioning support. A BsqON spec for the `Forecast` data type might look like:

```
enum Units { us, si }
enum CompassDirection { North, South, East, West }

type Fahrenheit = Int;
type Mph = Nat;

entity TempRange { field low: Fahrenheit; field high: Fahrenheit; }
entity WindSpeedRange { field min: Mph; field max: Mph; }

type Percentage = Float & {
    validate 0.0<Percentage> <= $value;
    validate $value <= 100.0<Percentage>;
}

const ForecastOption: CRegex = /'Showers'|'Thunderstorms'|'Snow'|'Fog'/c;
type ForecastDetail = CString of /${ForecastType}(' and '${ForecastType})*/c;

entity Forecast {
    field temp: TempRange;
    field windSpeed: WindSpeedRange;
    field windDirection: CompassDirection;
    field shortForecast: CString;
    field forecastPercent: Percentage;
    field units: Units;
}
```

In this example we can see that we have replaced the sentinel strings with actual enums. In addition the numeric values are explicitly connected with their semantic unit information and the percentage value is set with the appropriate range constraints. Further, even string values with complex structures, like the `shortForecast` field, can be specified with regular expressions. Of course all of these properties and constraints can be mechanically validated at runtime and surfaced in IDE tooling!

## Literal Notation

specs

provenance

trustworthy-by-construction