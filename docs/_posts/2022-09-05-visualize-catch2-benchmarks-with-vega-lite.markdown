---
layout: post
title:  "Visualize Catch2 benchmarks with Vega-Lite"
date:   2022-09-05 20:20:00 +0100
categories: data
tags: c++ catch2 vega vega-lite benchmark visualization chart
author: JohT
discussions-id: 30
---

Whereas there are many tools that visualize unit test results, fully automated visualizations of benchmark results are hard to find. This article shows step by step, how to create a bar chart with [Vega-Lite][VegaLite] from scratch and how to use it to visualize [Catch2][Catch2] benchmark results for C++.

### Table of Contents
{:.no_toc}
1. A markdown unordered list which will be replaced with the table of contents.
{:toc}

{% raw %}

## Introduction

C++ is known to be used for projects where performance is crucial. However, just using C++ doesn't guarantee for high performance. During implementation of performance critical code units, [Microbenchmarks][Microbenchmark] are used early on and continuously to measure their performance. This is similar to unit testing, but with a different goal. 

Whereas there are many tools that visualize unit test results, fully automated visualization of benchmark results are hard to find. This article shows step by step, how to create a bar chart with [Vega-Lite][VegaLite] from scratch and how to use it to visualize [Catch2][Catch2] benchmark results for C++.

## Create a simple bar chart with [Vega-Lite][VegaLite]

### What is [Vega-Lite][VegaLite]?

[Vega-Lite][VegaLite] is...
> a high-level grammar for interactive graphics. It provides a concise JSON syntax for supporting rapid generation of interactive multi-view visualizations to support analysis.

It...
> compiles a Vega-Lite specification into a lower-level, more detailed [Vega][Vega] specification and renders it using Vega’s compiler.

[Vega][Vega] is...
> a visualization grammar, a declarative language for creating, saving, and sharing interactive visualization designs. With Vega, you can describe the visual appearance and interactive behavior of a visualization in a JSON format, and generate web-based views using Canvas or SVG.

### [Vega-Lite Online Editor][VegaLiteEditor] Example

Lets get right into it and create a simple bar chart. Open the [Vega-Lite Online Editor [11]][VegaLiteEditor], click on `examples` and select `Simple Bar Chart`. Try different settings and values to get familiar with it.

### Change to horizontal bar chart

By simply swapping `x` and `y` in the `encoding` object, you get a horizontal bar chart.

## Create a benchmark chart with [Vega-Lite][VegaLite]

We'll change the data source to an external URL containing already existing benchmark data. How this data is obtained will be shown later in [Get Catch2 benchmarks results](#get-catch2-benchmarks-results).

```json
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
        "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  }
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "A simple bar chart with embedded data.",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
        "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "mark": "bar",
  "encoding": {
    "y": {"field": "a", "type": "nominal", "axis": {"labelAngle": 0}},
    "x": {"field": "b", "type": "quantitative"}
  }
}
```
</details>
{::options parse_block_html="false" /}

The chart disappears and some warnings are shown. This is because the structure and fields of the data are now different and need to be adapted in the `encoding` object. We can fix this problem with the following  settings:

```json
{
  "encoding": {
    "x": {
      "field": "mean[0].$.value",
      "title": "execution time in ns",
      "type": "quantitative"
    },
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal"
    }
  }
}
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "mean[0].$.value",
      "title": "execution time in ns",
      "type": "quantitative"
    },
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal"
    }
  }
}
```
</details>
{::options parse_block_html="false" /}

### Filtering data

The data contains benchmark results for two different scenarios (kernel size). Showing both of them in one chart makes it hard to read. We can filter the data to only show the results for one scenario using a `transform` `filter`:

```json
  "transform": [{"filter": "indexof(datum.$.name, '((kernel size 16))') > 0"}]
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {"filter": "indexof(datum.$.name, '((kernel size 16))') > 0"}
  ],
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "mean[0].$.value",
      "title": "execution time in ns",
      "type": "quantitative"
    },
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal"
    }
  }
}
```
</details>
{::options parse_block_html="false" /}

### Execution time in microseconds

The mean execution time is given in nanoseconds. This leads in this case to large numbers, which are hard to compare. So lets convert the mean execution time to microseconds and round it to the nearest integer value. These transformations will also help us later when it comes to sorting.

```json
  "transform": [
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)", 
      "as": "MeanExecutionTime"
    }
  ]
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {
      "filter": "indexof(datum.$.name, '((kernel size 16))') > 0"
    },
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)", 
      "as": "MeanExecutionTime"
    }
  ],
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "MeanExecutionTime",
      "title": "execution time in us",
      "type": "quantitative"
    },
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal"
    }
  }
}
```
</details>
{::options parse_block_html="false" /}

### Sorting algorithms by execution time

Adding `"sort": "x"` to the y axis `encoding` will result in a list of algorithms sorted by their execution time beginning with the fastest one:

```json
{
  "encoding": {
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal", 
      "sort" : "x"
    }
  }
}
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {"filter": "indexof(datum.$.name, '((kernel size 16))') > 0"},
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)",
      "as": "MeanExecutionTime"
    }
  ],
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "MeanExecutionTime",
      "title": "execution time in us",
      "type": "quantitative"
    },
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal", 
      "sort" : "x"
    }
  }
}
```
</details>
{::options parse_block_html="false" /}

### Faceted Chart (Advanced)

The previous chart shows the execution time for the different algorithms, but only for the filtered scenario. The benchmarks are parametrized by their kernel size, meaning that every benchmark is done for every kernel size. It would be nice to show that in a multi chart, so that every kernel size gets its own chart. 

The first step is to remove the filter and extract "kernel size ..." of the benchmark description into a dedicated variable. Since the parameter is surrounded with two round brackets, a split using regex can easily be done like this:

```json
{
  "transform": [
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)",
      "as": "MeanExecutionTime"
    },
    {
      "calculate": "split(split(datum.$.name, '((')[1],'))')[0]",
      "as": "ParametrizedBenchmarkDescription"
    }
  ]
}
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)",
      "as": "MeanExecutionTime"
    },
    {
      "calculate": "split(split(datum.$.name, '((')[1],'))')[0]",
      "as": "ParametrizedBenchmarkDescription"
    }
  ],
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "MeanExecutionTime",
      "title": "execution time in us",
      "type": "quantitative"
    },
    "y": {
      "field": "$.name", 
      "title": "algorithm", 
      "type": "nominal", 
      "sort" : "x"
    }
  }
}
```
</details>
{::options parse_block_html="false" /}


As described in [Faceting a Plot into a Trellis Plot [12]][VegaLiteFacet], faceted charts can be created by adding `facet` settings, where the discriminating field is defined, and surrounding the remaining chart description by `spec`. 

```json
{
  "facet": {
    "field": "ParametrizedBenchmarkDescription",
    "type": "nominal"
  },
  "spec": {

  }
}
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)",
      "as": "MeanExecutionTime"
    },
    {
      "calculate": "split(split(datum.$.name, '((')[1],'))')[0]",
      "as": "ParametrizedBenchmarkDescription"
    }
  ],
  "facet": {
    "field": "ParametrizedBenchmarkDescription", 
    "type": "nominal"
  },
  "spec": {
    "mark": "bar",
    "encoding": {
      "x": {
        "field": "MeanExecutionTime",
        "title": "execution time in us",
        "type": "quantitative"
      },
      "y": {
        "field": "$.name",
        "title": "algorithm",
        "type": "nominal",
        "sort": "x"
      }
    }
  }
}
```
</details>
{::options parse_block_html="false" /}

The result is a large chart that is divided into two columns with the same scale. This might be useful for comparison charts. In our case, however, we rather want two independent charts with different execution time scales and different algorithm orders. This can be achieved using `resolve` as follows:

```json
{
  "https://joht.github.io/johtizen/": {
    "scale": {
      "x": "independent",
      "y": "independent"
    }
  }
}
```

{::options parse_block_html="true" /}
<details><summary markdown="span">Whole Vega-Lite JSON</summary>

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)",
      "as": "MeanExecutionTime"
    },
    {
      "calculate": "split(split(datum.$.name, '((')[1],'))')[0]",
      "as": "ParametrizedBenchmarkDescription"
    }
  ],
  "facet": {
    "field": "ParametrizedBenchmarkDescription", 
    "type": "nominal"
  },
  "resolve": {
    "scale": {
      "x": "independent",
      "y": "independent"
    }
  },
  "spec": {
    "mark": "bar",
    "encoding": {
      "x": {
        "field": "MeanExecutionTime",
        "title": "execution time in us",
        "type": "quantitative"
      },
      "y": {
        "field": "$.name",
        "title": "algorithm",
        "type": "nominal",
        "sort": "x"
      }
    }
  }
}
```
</details>
{::options parse_block_html="false" /}

To fix the x axis that is now shown on top of the chart, the name of the algorithm is defined as a distinct variable replacing `$.name`. This also enables us to remove the kernel size in the description, that is already shown on top of the chart. Here is the additional entry for the `transform` block:

```json
    {
      "calculate": "replace(datum.$.name, '((' + datum.ParametrizedBenchmarkDescription + '))', '')",
      "as": "Algorithm"
    }
```

### Final [Vega-Lite][VegaLite] JSON

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "description": "Benchmark Results",
  "data": {
    "url": "https://raw.githubusercontent.com/JohT/convolution-benchmarks/main/chart/AppleClang-macOS-arm64/benchmark-report.json",
    "format": {
      "type": "json",
      "property": "Catch2TestRun.TestCase[0].BenchmarkResults"
    }
  },
  "transform": [
    {
      "calculate": "split(split(datum.$.name, '((')[1],'))')[0]",
      "as": "ParametrizedBenchmarkDescription"
    },
    {
      "calculate": "replace(datum.$.name, '((' + datum.ParametrizedBenchmarkDescription + '))', '')",
      "as": "Algorithm"
    },
    {
      "calculate": "round(toNumber(datum.mean[0].$.value) / 1000)",
      "as": "MeanExecutionTime"
    }
  ],
  "facet": {
    "column": {
      "field": "ParametrizedBenchmarkDescription",
      "title": "Benchmarks"
    }
  },
  "resolve": {
    "scale": {
      "x": "independent",
      "y": "independent"
    }
  },
  "spec": {
    "mark": "bar",
    "encoding": {
      "x": {
        "field": "MeanExecutionTime",
        "title": "execution time in us",
        "type": "quantitative"
      },
      "y": {
        "field": "Algorithm",
        "title": "algorithm",
        "type": "nominal",
        "sort": "x"
      }
    }
  }
}
```


## Create an image file with the chart in command line mode

So far, we know how to create the benchmark chart in the browser using the [Vega-Lite Online Editor][VegaLiteEditor]. Now we want to create the chart as an image in command line mode. This enables us to use the charts in a static manner like a markdown document.

### Setup [Node.js][NodeJS]

[Vega-Lite][VegaLite] is written in TypeScript and needs a JavaScript environment to run. 
We will use [Node.js [8]][NodeJS] command line scripts to render the results into a SVG image. Therefore, the following tools need to be installed:

- [Node.js][NodeJS]
- A suitable package manager like [npm [9]][npm]
- [node-canvas [10]](https://github.com/Automattic/node-canvas#compiling) installation to create image files

### Create a benchmark chart SVG with npx

Let's prepare a folder to have all in one place:
- Create a folder with the name `chart`
- Create an empty file named `VegaLiteChart.json` in that folder
- Copy the contents of [Final Vega-Lite JSON](#final-vega-lite-json) into the `VegaLiteChart.json`

The following command contains everything that is necessary to create the SVG chart file as described in [Vega-Lite From the Command Line using npx [13]][VegaLiteSVG]:

```shell
npx --yes -p vega -p vega-lite vl2svg VegaLiteChart.json VegaLiteChart.svg
```

### Create a [Node.js][NodeJS] project

The following commands "upgrade" your folder to a [Node.js][NodeJS] project:

```shell
npm init
npm install --save-dev vega-lite
npm install --save-dev vega
```

Add a script to your `package.json` file to make it easy to create the SVG chart file:

```json
"scripts": {
  "test": "echo \"Error: no test specified\" && exit 1",
  "createChart": "vl2svg VegaLiteChart.json VegaLiteChart.svg"
}
```

The chart can then simply be created with:

```shell
npm run createChart
```

## Get [Catch2][Catch2] benchmarks results 

### C++ [Microbenchmarks][Microbenchmark]

There are quite a few [Microbenchmark][Microbenchmark] libraries. Since [Catch2 [1]][Catch2] is a well known unit test framework, it comes in handy that it also provides basic micro-benchmarking features. [Google Benchmark][GoogleBenchmark] might also be a great choice. Write a comment if you want to share your experience or if you've found a way to visualize it's results.

### What is [Catch2][Catch2]?

[Catch2][Catch2] is ...
> mainly a unit testing framework for C++, but it also provides basic micro-benchmarking features, and simple [BDD][BDD] macros.

### How do [Catch2 Benchmarks][Catch2Benchmarks] look like?

The following example shows an advanced [Catch2 Benchmark [6]][Catch2Benchmarks] with some extra features that are explained in the comments:

```cpp
#include <catch2/benchmark/catch_benchmark.hpp>
#include <catch2/catch_test_macros.hpp>

//[.] means that the test won't be run automatically.
//[my_benchmarks] is a freely named tag that can be used to only run those tests.
TEST_CASE("MicroBenchmarks", "[.][my_benchmarks]") {
    // Code that is placed here will be run before every benchmark. 
    // This is useful for setting up the environment.
    BENCHMARK_ADVANCED("function_to_measure")(Catch::Benchmark::Chronometer meter) {
        //Code outside of the measure block won't be measured
        set_up();
        //The return of a result inside the measure block 
        //prevents the optimizer from optimizing the code away
        meter.measure([] { return function_to_measure(); });
    };
}
```

### How to run [Catch2 Benchmarks][Catch2Benchmarks]?

Suppose the test build target is named `BenchmarkTests` and its executable is located in the `build/test` directory, then the following command will run the benchmarks (Linux/MacOS):

```shell
./build/test/BenchmarksTests '[my_benchmarks]'
``` 

For Windows, the command is:
```shell
./build/test/BenchmarksTests.exe "[my_benchmarks]"
``` 

The result will by default be printed to the console and might look like this:

```
benchmark name                       samples       iterations    estimated
                                     mean          low mean      high mean
                                     std dev       low std dev   high std dev
-------------------------------------------------------------------------------
function_to_measure                         100             1    65.3912 ms 
                                        655.656 us    654.324 us      657.3 us 
                                        7.51349 us    6.38275 us    9.90394 us 
```

### How to get machine readable results

To get the results additionally in a machine readable format, use the `--reporter` option:

```shell
./build/test/BenchmarksTests [my_benchmarks] --reporter XML::out=./build/test/benchmark-report.xml --reporter console::out=-::colour-mode=ansi
```

The result is a XML file that will look something like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Catch2TestRun name="BenchmarksTests" rng-seed="1523697892" catch2-version="3.1.0" filters="[my_benchmarks]">
  <TestCase name="MicroBenchmarks" tags="[.][my_benchmarks]" filename="....../build/test/BenchmarksTests.cpp" line="16">
    <BenchmarkResults name="function_to_measure" samples="100" resamples="100000" iterations="1" clockResolution="17.2221" estimatedDuration="6.53912e+07">
      <!-- All values in nano seconds -->
      <mean value="655656" lowerBound="654324" upperBound="657300" ci="0.95"/>
      <standardDeviation value="7513.49" lowerBound="6382.75" upperBound="9903.94" ci="0.95"/>
      <outliers variance="0.0099" lowMild="0" lowSevere="0" highMild="1" highSevere="0"/>
    </BenchmarkResults>
    <OverallResult success="true"/>
  </TestCase>
  <OverallResults successes="0" failures="0" expectedFailures="0"/>
  <OverallResultsCases successes="1" failures="0" expectedFailures="0"/>
</Catch2TestRun>
```

## Convert XML to JSON

The missing last step is to convert the benchmark results XML file into a JSON file, since [Vega-Lite][VegaLite] requires the data in JSON format. Within [Node.js][NodeJS], this can be done with the following command, assuming that the results are located in the same folder and are named `benchmark-results.xml`:

```shell
npx --yes convert-xml-to-json benchmark-results.xml benchmark-results.json
```

The following command adds it to your [Node.js][NodeJS] project as a development dependency:

```shell
npm install --save-dev convert-xml-to-json
```

Finally, adding a script for it in the `package.json` provides an easy to use command:
```json
"scripts": {
  "test": "echo \"Error: no test specified\" && exit 1",
  "convertXML2JSON": "convert-xml-to-json benchmark-results.xml benchmark-results.json",
  "createChart": "vl2svg VegaLiteChart.json VegaLiteChart.svg"
}
```

The file can then simply be converted with:

```shell
npm run convertXML2JSON
```

## Real world example

A comprehensive example can be found here: [https://github.com/JohT/convolution-benchmarks][ConvolutionBenchmarks] 

It contains all previously described steps and includes a fully automated pipeline running on different platforms and for different configurations.

{% endraw %}


<br>

----

## References
- [[1] Catch2][Catch2]  
https://github.com/catchorg/Catch2
- [[2] Vega-Lite][VegaLite]  
https://vega.github.io/vega-lite
- [[3] Microbenchmark][Microbenchmark]  
https://link.springer.com/referenceworkentry/10.1007/978-3-319-77525-8_111
- [[4] Introducing Behaviour-Driven Development (BDD)][BDD]   
https://dannorth.net/introducing-bdd
- [[5] Google Benchmark][GoogleBenchmark]   
https://github.com/google/benchmark
- [[6] Catch2 Benchmarks][Catch2Benchmarks]   
https://github.com/catchorg/Catch2/blob/devel/docs/benchmarks.md
- [[7] Vega][Vega]   
https://vega.github.io/vega
- [[8] Node.js][NodeJS]   
https://nodejs.dev
- [[9] npm][npm]   
https://www.npmjs.com
- [[10] node-canvas][node-canvas]   
https://github.com/Automattic/node-canvas#compiling
- [[11] Vega-Lite Online Editor][VegaLiteEditor]   
https://vega.github.io/editor/#/custom/vega-lite
[VegaLiteFacet]: https://vega.github.io/vega-lite/docs/facet.html
- [[12] Faceting a Plot into a Trellis Plot][VegaLiteFacet]   
https://vega.github.io/vega-lite/docs/facet.html
- [[13] Vega-Lite From the Command Line using npx][VegaLiteSVG]   
https://vega.github.io/vega-lite/usage/compile.html#using-npx
- [[14] Benchmark Convolution Implementations][ConvolutionBenchmarks]   
https://github.com/JohT/convolution-benchmarks

[BDD]: https://dannorth.net/introducing-bdd
[Catch2]: https://github.com/catchorg/Catch2
[Catch2Benchmarks]: https://github.com/catchorg/Catch2/blob/devel/docs/benchmarks.md
[GoogleBenchmark]: https://github.com/google/benchmark
[Microbenchmark]: https://link.springer.com/referenceworkentry/10.1007/978-3-319-77525-8_111
[NodeJS]: https://nodejs.dev
[node-canvas]: https://github.com/Automattic/node-canvas#compiling
[npm]: https://www.npmjs.com
[Vega]: https://vega.github.io/vega
[VegaLite]: https://vega.github.io/vega-lite
[VegaLiteEditor]: https://vega.github.io/editor/#/custom/vega-lite
[VegaLiteFacet]: https://vega.github.io/vega-lite/docs/facet.html
[VegaLiteSVG]: https://vega.github.io/vega-lite/usage/compile.html#using-npx
[ConvolutionBenchmarks]: https://github.com/JohT/convolution-benchmarks