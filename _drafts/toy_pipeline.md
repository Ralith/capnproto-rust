---
layout: post
title: a toy data pipeline with capnproto-rust and zeromq
---

4 January 2014

As capnproto-rust approaches full feature support
for Cap'n Proto serialization,
now is an apt time to validate its usefulness on a
slightly more involved example.


Thus I present
[zmq-explorers](https://github.com/dwrensha/capnproto-rust/tree/master/examples/zmq-explorers),
a toy data pipeline which uses
[ZeroMQ](http://zeromq.org/)
as a transport layer.

The pipeline looks like this:
<center>
<img src="{{site.baseurl}}/assets/zmq-explorers.png"
     width="225"/>
</center>

At the input end are
any number of "explorer" nodes gathering data.
In the middle is
a "collector" node aggregating and processing the data.
At the end is a "viewer" node consuming the processed data.
The explorers communicate to the collector via a publish/subscribe
pattern, and the viewer communicates with the collector via a request/reply pattern.
ZeroMQ makes this communication quite simple to express.

Concretely, the observations of the explorers
are color values for an image:

```
struct Observation {
    timestamp  @0 : Int64;
    x          @1 : Float32;
    y          @2 : Float32;
    red        @3 : UInt8;
    green      @4 : UInt8;
    blue       @5 : UInt8;
    diagnostic : union {
       ok @6 : Void;
       warning @7 : Text;
    }
}
```

The explorers randomly move withing the two-dimensional space
of the image, reporting the colors they observe,
fudged by some noise.

The collector maintains a grid that aggregates the reported observations:

```
struct Grid {
   cells @0 : List(List(Cell));
   numberOfUpdates @1 : UInt32;
   latestTimestamp @2 : Int64;

   struct Cell {
      latestTimestamp @0 : Int64;
      numberOfUpdates @1 : UInt32;
      meanRed         @2 : Float32;
      meanGreen       @3 : Float32;
      meanBlue        @4 : Float32;
   }
}
```

The collector sends its current grid to the viewer whenever
it is requested.

Here is what the viewer might see if there are three explorers:

<center>
<img src="{{site.baseurl}}/assets/rust_logo_colors.gif"
     width="120"/>
<img src="{{site.baseurl}}/assets/rust_logo_confidence.gif"
     width="120"/>
</center>

The image on left shows the current best estimate of the
data. The image on the right shows
a crude measure of confidence in the estimate;
green represents the value of the `numberOfUpdates` field,
and blue represents the value of the `latestTimestamp` field
relative to the current time.