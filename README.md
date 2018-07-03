Erlang Problems
===============

This project is a simple Erlang exercise with a couple of modules waiting to be filled in with code. There are
two, high-level tests showing some expected output for each module. You should probably write some more tests along
the way to completing the exercises. There are also shell scripts included to quickly run the tests and to clean
the project after testing. The only requirement is an Erlang installation.

JSON Rendering
--------------

One module wants you to implement rendering of an Erlang term into JSON. The general rules are that an Erlang atom 
or binary string should become a JSON string, an integer should become a JSON number, and a list of terms should
become a JSON array. Because Erlang has no concept mirroring a JSON object (until maps were added, but we'll
assume those don't exist yet), we'll say that a single-element tuple containing a proplist represents a JSON object
(i.e. `{[{field, value}, ...]}`. Beware Erlang's "a string is a list".

Weather Forecasts
-----------------

One module mimics retrieving data from an external weather API. To avoid non-portability, the weather
data has been hardcoded into a separate module instead of using an HTTP client and a real API. The fake data module
has a sleep built in to simulate the delay in talking to an API. The problem expressed by the test is to retrieve
forecast data for several locations at the same time without incurring an excessive wait.

jsone
=====

[![hex.pm version](https://img.shields.io/hexpm/v/jsone.svg)](https://hex.pm/packages/jsone)
[![Build Status](https://travis-ci.org/sile/jsone.svg?branch=master)](https://travis-ci.org/sile/jsone)
[![Code Coverage](https://codecov.io/gh/sile/jsone/branch/master/graph/badge.svg)](https://codecov.io/gh/sile/jsone/branch/master)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

An Erlang library for encoding, decoding [JSON](http://json.org/index.html) data.


Features
--------
- Provides simple encode/decode function only
- [RFC7159](http://www.ietf.org/rfc/rfc7159.txt)-compliant
- Supports UTF-8 encoded binary
- Pure Erlang
- Highly Efficient
  - Maybe one of the fastest JSON library (except those which are implemented in NIF)
      - See [Benchmark](#benchmark)
  - Decode function is written in continuation-passing style(CPS)
      - CPS facilitates application of 'creation of sub binary delayed' optimization
      - See also [Erlang Efficiency Guide](http://www.erlang.org/doc/efficiency_guide/binaryhandling.html)


QuickStart
----------

```sh
# clone
$ git clone git://github.com/sile/jsone.git
$ cd jsone

# compile
$ make compile

# run tests
$ make eunit

# dialyze
$ make dialyze

# Erlang shell
$ make start
1> jsone:decode(<<"[1,2,3]">>).
[1,2,3]
```

Enable HiPE
-----------

If you want to use HiPE compiled version, please add following code to your rebar.config.

```erlang
{overrides,
  [
    {override, jsone, [{erl_opts, [{d, 'ENABLE_HIPE'}, inline]}]}
  ]}.
```

or use `native` profile. The `make` command supports profile as well. For example:

```sh
$ make start profile=native
```

Usage Example
-------------

```erlang
%% Decode
> jsone:decode(<<"[1,2,3]">>).
[1,2,3]

> jsone:decode(<<"{\"1\":2}">>).
#{<<"1">> => 2}

> jsone:decode(<<"{\"1\":2}">>, [{object_format, tuple}]). % tuple format
{[{<<"1">>, 2}]}

> jsone:decode(<<"{\"1\":2}">>, [{object_format, proplist}]). % proplist format
[{<<"1">>, 2}]

> jsone:try_decode(<<"[1,2,3] \"next value\"">>). % try_decode/1 returns remaining (unconsumed binary)
{ok,[1,2,3],<<" \"next value\"">>}

% error: raises exception
> jsone:decode(<<"1.x">>).
** exception error: bad argument
     in function  jsone_decode:number_fraction_part_rest/6
        called as jsone_decode:number_fraction_part_rest(<<"x">>,1,1,0,[],<<>>)
     in call from jsone:decode/1 (src/jsone.erl, line 71)

% error: returns {error, Reason}
> jsone:try_decode(<<"1.x">>).
{error,{badarg,[{jsone_decode,number_fraction_part_rest,
                              [<<"x">>,1,1,0,[],<<>>],
                              [{line,228}]}]}}


%% Encode
> jsone:encode([1,2,3]).
<<"[1,2,3]">>

> jsone:encode(#{<<"key">> => <<"value">>}).  % map format
> jsone:encode({[{<<"key">>, <<"value">>}]}). % tuple format
> jsone:encode([{<<"key">>, <<"value">>}]).  % proplist format
<<"{\"key\":\"value\"}">>

> jsone:encode(#{key => <<"value">>}). % atom key is allowed
<<"{\"key\":\"value\"}">>

% error: raises exception
> jsone:encode(#{123 => <<"value">>}). % non binary|atom key is not allowed
** exception error: bad argument
     in function  jsone_encode:object_members/3
        called as jsone_encode:object_members([{123,<<"value">>}],[],<<"{">>)
     in call from jsone:encode/1 (src/jsone.erl, line 97)

% error: returns {error, Reason}
> jsone:try_encode({[{123, <<"value">>}]}).
{error,{badarg,[{jsone_encode,object_members,
                              [[{123,<<"value">>}],[],<<"{">>],
                              [{line,138}]}]}}

% 'object_key_type' option allows non-string object key
> jsone:encode({[{123, <<"value">>}]}, [{object_key_type, scalar}]).
<<"{\"123\":\"value\"}">>

% 'undefined_as_null' option allows encoding atom undefined as null
> jsone:encode(undefined,[undefined_as_null]).
<<"null">>

%% Pretty Print
> Data = [true, #{<<"1">> => 2, <<"array">> => [[[[1]]], #{<<"ab">> => <<"cd">>}, false]}, null].
> io:format("~s\n", [jsone:encode(Data, [{indent, 1}, {space, 2}])]).
[
  true,
  {
    "1": 2,
    "array": [
      [
        [
          [
            1
          ]
        ]
      ],
      {
        "ab": "cd"
      },
      false
    ]
  },
  null
]
ok

%% Number Format
> jsone:encode(1). % integer
<<"1">>

> jsone:encode(1.23). % float
<<"1.22999999999999998224e+00">> % default: scientific notation

> jsone:encode(1.23, [{float_format, [{decimals, 4}]}]). % decimal notation
<<"1.2300">>

> jsone:encode(1.23, [{float_format, [{decimals, 4}, compact]}]). % compact decimal notation
<<"1.23">>
```


Data Mapping (Erlang <=> JSON)
-------------------------------

```
Erlang                  JSON             Erlang
=================================================================================================

null                   -> null                       -> null
undefined              -> null                       -> undefined                  % undefined_as_null
true                   -> true                       -> true
false                  -> false                      -> false
<<"abc">>              -> "abc"                      -> <<"abc">>
abc                    -> "abc"                      -> <<"abc">> % non-special atom is regarded as a binary
{{2010,1,1},{0,0,0}}   -> "2010-01-01T00:00:00Z"     -> <<"2010-01-01T00:00:00Z">>     % datetime*
{{2010,1,1},{0,0,0.0}} -> "2010-01-01T00:00:00.000Z" -> <<"2010-01-01T00:00:00.000Z">> % datetime*
123                    -> 123                        -> 123
123.4                  -> 123.4                      -> 123.4
[1,2,3]                -> [1,2,3]                    -> [1,2,3]
{[]}                   -> {}                         -> {[]}                       % object_format=tuple
{[{key, <<"val">>}]}   -> {"key":"val"}              -> {[{<<"key">>, <<"val">>}]} % object_format=tuple
[{}]                   -> {}                         -> [{}]                       % object_format=proplist
[{<<"key">>, val}]     -> {"key":"val"}              -> [{<<"key">>, <<"val">>}]   % object_format=proplist
#{}                    -> {}                         -> #{}                        % object_format=map
#{key => val}          -> {"key":"val"}              -> #{<<"key">> => <<"val">>}  % object_format=map
{{json, IOList}}       -> Value                      -> ~~~                        % UTF-8 encoded term**
{{json_utf8, Chars}}   -> Value                      -> ~~~                        % Unicode code points**
```

\* see [jsone:datetime_encode_format()](doc/jsone.md#type-datetime_encode_format)

\** `{json, IOList}` and `{json_utf8, Chars}` allows inline already encoded JSON
values. For example, you obtain JSON encoded data from database so you don't
have to decode it first and encode again. See [jsone:json_term()](doc/jsone.md#type-json_term).

API
---

See [EDoc Document](doc/jsone.md)


Benchmark
---------

The results of [poison](https://github.com/devinus/poison) benchmarking.

See the [BENCHMARK.md](BENCHMARK.md) file for more information.

### EncoderBench Result

__Non HiPE__:

|                  | jiffy        | jsone             | poison        | jazz          | jsx           |
|:-----------------|-------------:|------------------:|--------------:|--------------:|--------------:|
| maps             |   7.23 μs/op |   10.64 μs/op (2) |   13.58 μs/op |   19.30 μs/op |   29.28 μs/op |
| lists            | 210.40 μs/op |  157.39 μs/op (3) |  109.30 μs/op |  201.82 μs/op |  357.25 μs/op |
| strings*         |  98.80 μs/op |  595.63 μs/op (5) |  416.78 μs/op |  399.89 μs/op |  262.18 μs/op |
| string escaping* | 144.01 μs/op |  732.44 μs/op (2) | 1318.82 μs/op | 1197.06 μs/op | 1324.04 μs/op |
| large value**    | 408.03 μs/op | 1556.85 μs/op (3) | 1447.71 μs/op | 1824.05 μs/op | 2184.59 μs/op |
| pretty print**   | 420.94 μs/op | 1686.55 μs/op (3) | 1534.74 μs/op | 2041.22 μs/op | 5533.04 μs/op |


__HiPE__:

|                  | jiffy        | jsone             | poison        | jazz          | jsx           |
|:-----------------|-------------:|------------------:|--------------:|--------------:|--------------:|
| maps             |   7.69 μs/op |    6.12 μs/op (1) |   12.32 μs/op |   22.90 μs/op |   27.03 μs/op |
| lists            | 207.75 μs/op |   69.93 μs/op (1) |   79.04 μs/op |  229.95 μs/op |  278.01 μs/op |
| strings*         |  96.67 μs/op |  321.69 μs/op (5) |  142.43 μs/op |  310.10 μs/op |  179.96 μs/op |
| string escaping* | 146.85 μs/op |  317.10 μs/op (2) | 1277.54 μs/op | 1311.85 μs/op |  767.67 μs/op |
| large value**    | 409.73 μs/op |  664.34 μs/op (2) |  806.24 μs/op | 1630.21 μs/op | 1777.62 μs/op |
| pretty print**   | 419.55 μs/op |  724.28 μs/op (2) |  844.76 μs/op | 1888.71 μs/op | 4872.34 μs/op |

\* binary representation of [UTF-8-demo.txt](https://github.com/devinus/poison/blob/2.1.0/bench/data/UTF-8-demo.txt)  <br />
\** [generated.json](https://github.com/devinus/poison/blob/2.1.0/bench/data/generated.json)

### ParserBench Result

__Non HiPE__:

|                    | jiffy        | jsone             | poison        | jsx           |
|:-------------------|-------------:|------------------:|--------------:|--------------:|
| json value*        | 544.84 μs/op | 1364.38 μs/op (2) | 1401.35 μs/op | 1844.55 μs/op |
| UTF-8 unescaping** |  63.01 μs/op |  399.38 μs/op (4) |  249.70 μs/op |  281.84 μs/op |


__HiPE__:

|                    | jiffy        | jsone             | poison        | jsx           |
|:-------------------|-------------:|------------------:|--------------:|--------------:|
| json value*        | 542.77 μs/op |  561.15 μs/op (2) |  751.36 μs/op | 1435.10 μs/op |
| UTF-8 unescaping** |  62.42 μs/op |   92.63 μs/op (2) |  118.97 μs/op |  172.07 μs/op |

\* [generated.json](https://github.com/devinus/poison/blob/2.1.0/bench/data/generated.json) <br />
\** [UTF-8-demo.txt](https://github.com/devinus/poison/blob/2.1.0/bench/data/UTF-8-demo.txt)

# forecast

Erlang API for the [forecast.io](https://developer.forecast.io/) weather service.

## Use
`FORECAST_API_KEY` comes from upfront free registration at forecast.io and must be set in the environment

```
$ export FORECAST_API_KEY=...
```

### Using From the Shell
```
$ make
$ make run
...
 
% Optionally load record def
1> rr("include/forecast.hrl").

2> forecast:start().

3> forecast:get({{lat, 60.173324}, {long, 24.941025}}).                                   
```

### Using From Your Own Project
* Add `forecast` to `applications` in your `.app.src` file
* Add URL for the repo in the `deps` in your rebar.config. For example, `{forecast, ".*", {git, "https://github.com/derek121/forecast.git"}}`
* Ensure `FORECAST_API_KEY` is set in your environment
* Run:
  * Building with relx and then running will start `forecast`, after which calls may be made
  * Otherwise, calling `forecast:start/0` will start `forecast`, after which calls may be made
* Optionally load record def. Example:
  * `rr("lib/forecast-0.0.1/include/forecast.hrl").`

## Example
```
forecast $ make run
...
1> forecast:start().
{ok,[inets,asn1,public_key,ssl,jiffy,forecast]}
2> rr("include/forecast.hrl").
[alert,data_block,data_point,flags,forecast]
3> forecast:get({{lat, 60.173324}, {long, 24.941025}}).
#forecast{latitude = 60.173324,longitude = 24.941025,
          timezone = <<"Europe/Helsinki">>,offset = 2,
          currently = #data_point{time = 1421939091,
                                  summary = <<"Overcast">>,icon = <<"cloudy">>,
                                  sunrise_time = undefined,sunset_time = undefined,
                                  moon_phase = undefined,nearest_storm_distance = undefined,
                                  nearest_storm_bearing = undefined,precip_intensity = 0,
                                  precip_intensity_max = undefined,
                                  precip_intensity_max_time = undefined,
                                  precip_probability = 0,precip_type = undefined,
                                  precip_accumulation = undefined,temperature = 20.62,
                                  temperature_min = undefined,
                                  temperature_min_time = undefined,
                                  temperature_max = undefined,
                                  temperature_max_time = undefined,
                                  apparent_temperature = 8.41,
                                  apparent_temperature_min = undefined,
                                  apparent_temperature_min_time = undefined,...},
          minutely = #data_block{summary = undefined,icon = undefined,
                                 data = undefined},
          hourly = #data_block{summary = <<"Light snow (under 1 in.) starting later this evening.">>,
                               icon = <<"snow">>,
                               data = [#data_point{time = 1421938800,
                                                   summary = <<"Overcast">>,icon = <<"cloudy">>,
                                                   sunrise_time = undefined,sunset_time = undefined,
                                                   moon_phase = undefined,nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0,
                                                   precip_intensity_max = undefined,
                                                   precip_intensity_max_time = undefined,
                                                   precip_probability = 0,precip_type = undefined,
                                                   precip_accumulation = undefined,temperature = 20.29,...},
                                       #data_point{time = 1421942400,summary = <<"Overcast">>,
                                                   icon = <<"cloudy">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0,
                                                   precip_intensity_max = undefined,
                                                   precip_intensity_max_time = undefined,
                                                   precip_probability = 0,precip_type = undefined,
                                                   precip_accumulation = undefined,...},
                                       #data_point{time = 1421946000,summary = <<"Overcast">>,
                                                   icon = <<"cloudy">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0,
                                                   precip_intensity_max = undefined,
                                                   precip_intensity_max_time = undefined,
                                                   precip_probability = 0,precip_type = undefined,...},
                                       #data_point{time = 1421949600,summary = <<"Overcast">>,
                                                   icon = <<"cloudy">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0,
                                                   precip_intensity_max = undefined,
                                                   precip_intensity_max_time = undefined,
                                                   precip_probability = 0,...},
                                       #data_point{time = 1421953200,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0.0037,
                                                   precip_intensity_max = undefined,
                                                   precip_intensity_max_time = undefined,...},
                                       #data_point{time = 1421956800,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0.0042,
                                                   precip_intensity_max = undefined,...},
                                       #data_point{time = 1421960400,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,precip_intensity = 0.0046,...},
                                       #data_point{time = 1421964000,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,
                                                   nearest_storm_bearing = undefined,...},
                                       #data_point{time = 1421967600,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,
                                                   nearest_storm_distance = undefined,...},
                                       #data_point{time = 1421971200,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,moon_phase = undefined,...},
                                       #data_point{time = 1421974800,summary = <<"Light Snow">>,
                                                   icon = <<"snow">>,sunrise_time = undefined,
                                                   sunset_time = undefined,...},
                                       #data_point{time = 1421978400,summary = <<"Light Sn"...>>,
                                                   icon = <<"snow">>,sunrise_time = undefined,...},
                                       #data_point{time = 1421982000,summary = <<"Ligh"...>>,
                                                   icon = <<...>>,...},
                                       #data_point{time = 1421985600,summary = <<...>>,...},
                                       #data_point{time = 1421989200,...},
                                       #data_point{...},
                                       {...}|...]},
          daily = #data_block{summary = <<76,105,103,104,116,32,115,
                                          110,111,119,32,40,50,226,
                                          128,147,53,32,...>>,
                              icon = <<"snow">>,
                              data = [#data_point{time = 1421877600,
                                                  summary = <<"Light snow (under 1 in.) until afternoon, starti"...>>,
                                                  icon = <<"snow">>,sunrise_time = 1421909941,
                                                  sunset_time = 1421935597,moon_phase = 0.07,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,precip_intensity = 0.0028,
                                                  precip_intensity_max = 0.0051,
                                                  precip_intensity_max_time = 1421920800,
                                                  precip_probability = 0.53,precip_type = <<"snow">>,
                                                  precip_accumulation = 0.971,...},
                                      #data_point{time = 1421964000,
                                                  summary = <<"Light snow (under 1 in.) in the morning and "...>>,
                                                  icon = <<"snow">>,sunrise_time = 1421996224,
                                                  sunset_time = 1422022147,moon_phase = 0.11,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,precip_intensity = 0.0041,
                                                  precip_intensity_max = 0.008,
                                                  precip_intensity_max_time = 1422032400,
                                                  precip_probability = 0.5,precip_type = <<...>>,...},
                                      #data_point{time = 1422050400,
                                                  summary = <<"Mostly cloudy throughout the day.">>,
                                                  icon = <<"partly-cloudy-day">>,sunrise_time = 1422082504,
                                                  sunset_time = 1422108699,moon_phase = 0.14,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,precip_intensity = 0.0008,
                                                  precip_intensity_max = 0.0039,
                                                  precip_intensity_max_time = 1422050400,
                                                  precip_probability = 0.07,...},
                                      #data_point{time = 1422136800,
                                                  summary = <<"Mostly cloudy until evening.">>,
                                                  icon = <<"partly-cloudy-day">>,sunrise_time = 1422168781,
                                                  sunset_time = 1422195252,moon_phase = 0.18,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,precip_intensity = 0,
                                                  precip_intensity_max = 0,
                                                  precip_intensity_max_time = undefined,...},
                                      #data_point{time = 1422223200,
                                                  summary = <<"Breezy throughout the day and fl"...>>,
                                                  icon = <<"snow">>,sunrise_time = 1422255055,
                                                  sunset_time = 1422281806,moon_phase = 0.22,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,precip_intensity = 0.0008,
                                                  precip_intensity_max = 0.0019,...},
                                      #data_point{time = 1422309600,
                                                  summary = <<"Light snow (under 1 in.) and"...>>,
                                                  icon = <<"snow">>,sunrise_time = 1422341327,
                                                  sunset_time = 1422368361,moon_phase = 0.25,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,precip_intensity = 0.0031,...},
                                      #data_point{time = 1422396000,
                                                  summary = <<"Light snow (under 1 in.)"...>>,
                                                  icon = <<"snow">>,sunrise_time = 1422427597,
                                                  sunset_time = 1422454917,moon_phase = 0.29,
                                                  nearest_storm_distance = undefined,
                                                  nearest_storm_bearing = undefined,...},
                                      #data_point{time = 1422482400,
                                                  summary = <<"Light snow (under 1 "...>>,icon = <<"snow">>,
                                                  sunrise_time = 1422513864,sunset_time = 1422541474,
                                                  moon_phase = 0.33,nearest_storm_distance = undefined,...}]},
          alerts = [],
          flags = #flags{darksky_unavailable = undefined,
                         darksky_stations = undefined,datapoint_stations = undefined,
                         isd_stations = [<<"027950-99999">>,<<"029750-99999">>,
                                         <<"029780-99999">>,<<"029860-99999">>,<<"029880-99999">>],
                         lamp_stations = undefined,metar_stations = undefined,
                         metno_license = <<"Based on data from the Norwegian Meteoro"...>>,
                         sources = [<<"isd">>,<<"madis">>,<<"metno_ce">>,
                                    <<"metno_ne">>,<<"fnmoc">>,<<"cmc">>,<<"gfs">>],
                         units = <<"us">>}}

```
License
-------

This library is released under the MIT License.

See the [COPYING](COPYING) file for full license information.
