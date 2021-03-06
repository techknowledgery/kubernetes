# Profiling Kubernetes

This document explain how to plug in profiler and how to profile Kubernetes services.

## Profiling library

Go comes with inbuilt 'net/http/pprof' profiling library and profiling web service. The way service works is binding debug/pprof/ subtree on a running webserver to the profiler. Reading from subpages of debug/pprof returns pprof-formated profiles of the running binary. The output can be processed offline by the tool of choice, or used as an input to handy 'go tool pprof', which can graphically represent the result.

## Adding profiling to services to APIserver.

TL;DR: Add lines:
```
  m.mux.HandleFunc("/debug/pprof/", pprof.Index)
  m.mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
  m.mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
```
to the init(c *Config) method in 'pkg/master/master.go' and import 'net/http/pprof' package.

In most use cases to use profiler service it's enough to do 'import _ net/http/pprof', which automatically registers a handler in the default http.Server. Slight inconvenience is that APIserver uses default server for intra-cluster communication, so plugging profiler to it is not really useful. In 'pkg/master/server/server.go' more servers are created and started as separate goroutines. The one that is usually serving external traffic is secureServer. The handler for this traffic is defined in 'pkg/master/master.go' and stored in Handler variable. It is created from HTTP multiplexer, so the only thing that needs to be done is adding profiler handler functions to this multiplexer. This is exactly what lines after TL;DR do.

## Connecting to the profiler
Even when running profiler I found not really straightforward to use 'go tool pprof' with it. The problem is that at least for dev purposes certificates generated for APIserver are not signed by anyone trusted and because secureServer serves only secure traffic it isn't straightforward to connect to the service. The best workaround I found is by creating an ssh tunnel from the kubernetes_master open unsecured port to some external server, and use this server as a proxy. To save everyone looking for correct ssh flags, it is done by running:
```
  ssh kubernetes_master -L<local_port>:localhost:8080
```
or analogous one for you Cloud provider. Afterwards you can e.g. run 
```
go tool pprof http://localhost:<local_port>/debug/pprof/profile
```
to get 30 sec. CPU profile.
