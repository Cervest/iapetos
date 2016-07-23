# iapetos

__iapetos__ is a Clojure wrapper around the [Prometheus Java
Client][java-client].

[java-client]: https://github.com/prometheus/client_java

## Usage

Don't. This is mostly me exploring Prometheus.

### Basic Usage

Metrics have to be declared on a collector registry before being used:

```clojure
(require '[iapetos.core :as prometheus])

(defonce registry
  (-> (prometheus/collector-registry)
      (prometheus/register
        (prometheus/histogram :app/duration-seconds)
        (prometheus/gauge     :app/last-success-unixtime {:lazy? true})
        (prometheus/gauge     :app/active-users-total    {:lazy? true})
        (prometheus/counter   :app/runs-total))))
```

Now, you can write an instrumented function using some of iapetos' helper macros
or by operating directly on the collectors:

```clojure
(defn run
  []
  (prometheus/inc registry :app/runs-total)
  (prometheus/with-duration (registry :app/duration-seconds)
    (prometheus/with-success-timestamp (registry :app/last-success-unixtime)
      ...
      (prometheus/set registry :app/active-users-total (count-users!))
      true)))
```

The metrics can then be either exported using a textual representation:

```clojure
(require '[iapetos.export :as export])
(print (export/text-format registry))
;; # HELP app_active_users_total a gauge metric.
;; # TYPE app_active_users_total gauge
;; app_active_users_total 10.0
;; # HELP app_last_success_unixtime a gauge metric.
;; # TYPE app_last_success_unixtime gauge
;; app_last_success_unixtime 1.469284587819E9
;; ...
```

Or pushed to the respective Prometheus gateway:

```clojure
(export/push! registry {:gateway "push-gateway:12345"})
```

### JVM Metrics

Some characteristics of your current JVM are always useful (e.g. memory
usage, thread count, ...) and can be added to your registry using the
`iapetos.collector.jvm` namespace:

```clojure
(require '[iapetos.collectors.jvm :as jvm])

(defonce registry
  (-> (prometheus/collector-registry)
      (prometheus/register
        ...
        (jvm/standard)
        (jvm/gc)
        (jvm/memory-pools)
        (jvm/threads))))
```

There is also `iapetos.collector.jvm/all` which will register all JVM
collectors.

__Note:__ You need to include the artifact `io.prometheus/simpleclient_hotspot`
explicitly in your project's dependencies.

### Function Instrumentation

To collect metrics about specific functions, you can use the functionality
provided in `iapetos.collector.fn`:

```clojure
(require '[iapetos.collectors.fn :as fn])

(defn- run-the-job!
  [job]
  ...)

(defonce registry
  (->> (prometheus/collector-registry)
       ...
       (fn/initialize)
       (fn/instrument #'run-the-job!)))
```

Now, every call to `run-the-job!` will update a series of duration, success and
failure metrics. Note that re-evaluation of the `run-the-job!` declaration will
remove the instrumentation again - which shouldn't be a problem in production,
though.

### More

Soon.

## License

```
MIT License

Copyright (c) 2016 Yannick Scherer

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
