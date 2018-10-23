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
