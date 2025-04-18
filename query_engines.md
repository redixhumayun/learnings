# Overview

Roughly two models:
* pull based model (volcano model)
* push based model

The volcano model is the predominant one. It works on the assumption that the driving force is the top level consumer which forces each of the underlying iterators to move forward. This proceeds recursively.

```
      +------------+
      |  Consumer  |
      +------------+
            ^
            |
      +------------+
      |  Filter    |
      +------------+
            ^
            |
      +------------+
      |   Scan     |
      +------------+
```

The push based model seems a more recent one. Here the driving force starts from the "bottom", with each node forcing its parents to take ownership of a result set.

```
      +------------+
      |  Consumer  |
      +------------+
            |
            v
      +------------+
      |  Filter    |
      +------------+
            |
            v
      +------------+
      |   Scan     |
      +------------+
```

## Links
1. [Volcano paper](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf)
2. [Push vs Pull-Based Loop Fusion In Query Engines](https://arxiv.org/pdf/1610.09166)
3. [Justin's Post On Push vs Pull Engines](https://justinjaffray.com/query-engines-push-vs.-pull/)
4. [Timely Dataflow](https://timelydataflow.github.io/differential-dataflow/introduction.html)