# promethues-profiling

## how do I profile prometheus
essentially this part is quite easy since prometheus include pprof by default so
you don't have to instrument your app

all needed is to browse the prometheus route
`curl -s https://prometheus-k8s-openshift-monitoring.apps.1018-e8s.example.com/debug/pprof/heap > heap1.pprof`

generate heap used csv
```
while true; do
  h=$(curl -s http://localhost:9090/debug/pprof/heap?debug=1 |grep HeapInuse |awk -F "=" '{print $2}')
  echo "$(date "+%s"), ${h}" >> goheap.csv
  sleep 15
done
```

prometheus monitor for openshift-monitoring stack  

```
cname=$(docker ps --filter name=k8s_prometheus_prometheus-k8s --format "{{.Names}}")
ppid=$(pgrep -f '/bin/prometheus --web')

stamp=$(date "+%d_%m")
# start monitor
while true; do
  mkdir -p ./prometheus_profiler/${stamp} # create new dir per day
  d=$(date "+%s")
  nsenter --target $ppid --mount --uts --ipc --net --pid curl http://localhost:9090/debug/pprof/heap > ./prometheus_profiler/${stamp}/heap_${d}.pprof
  for arg in inuse_space inuse_objects alloc_space alloc_objects; do
    file="${arg}-${d}"
    echo "$(go tool pprof -${arg} -top ./prometheus_profiler/${stamp}/heap_${d}.pprof)" > ./prometheus_profiler/${stamp}/$file
  done
  sleep 120
done
```

# analysis
pick two heap dump as base (something around the beginning of the app time) and another from last one.
use the pprof tool to compare the two
`go tool pprof -inuse_objects -base heap_1539815522.pprof /usr/bin/prometheus heap_1540294587.pprof`

Note: make sure to have the right binary installed

hit top to identify the biggest Objects
```
Showing top 5 nodes out of 156
      flat  flat%   sum%        cum   cum%
   -314256 12.32% 12.32%    -270564 10.61%  github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb.(*SegmentWAL).Truncate /go/src/github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb/wal.go
    266509 10.45%  1.87%     266509 10.45%  github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb/index.(*uint32slice).Less <autogenerated>
     65537  2.57%   0.7%      65537  2.57%  sync.(*Map).Store /usr/local/go/src/sync/map.go
     43692  1.71%  2.41%      43692  1.71%  github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb/index.(*Writer).WritePostings /go/src/github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb/index/index.go
     32768  1.28%  3.69%      32768  1.28%  github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb/index.(*Writer).Close /go/src/github.com/prometheus/prometheus/vendor/github.com/prometheus/tsdb/index/index.go
```
drill down into a function to identify what did not pass the GC.
list

```
list Truncate
 ...
-314256    -334959 (flat, cum) 13.13% of Total
       .          .    341:		if err != nil {
       .          .    342:			return errors.Wrap(err, "open old WAL segment for read")
       .          .    343:		}
       .          .    344:		candidates = append(candidates, &segmentFile{
       .          .    345:			File:      f,
       .      43692    346:			minSeries: sf.minSeries,
```
