# Running the umbra benchmarks

Notes for this fork. Upstream's own `run-benchmark.sh` is **not** used: it
renders `cloud-init.sh` as EC2 user data, and that script posts every log to
`play.clickhouse.com`, where upstream automation turns it into an "Automated
results" pull request against ClickHouse/ClickBench. Results stay in this fork
until they are deliberately submitted.

## Locally

```bash
./run-local.sh umbra                       # docker, image from local-env.sh
LOCAL=1 ./run-local.sh umbra               # locally built server in umbra/bin
LOCAL=1 TAG=numaoff NUMA_TRANSACTIONNODES=0 ./run-local.sh umbra
```

`run-local.sh` runs the system's `benchmark.sh` and writes
`<system>/results/<DATE>/<machine>.<version>[.<variant>][.<tag>].json` through
`parse-result.py`. `TAG` labels runs that differ only in configuration, `TRACE=1`
dumps a perftracer trace per phase, `BENCH_CONCURRENT_DURATION=0` skips the ten
minute concurrency window.

Umbra settings reach the server through the environment; the start scripts
forward the ones named in `UMBRA_FORWARD` (`PARALLEL`, `BUFFERSIZE`, `MEMORY`,
`NUMA_TRANSACTIONNODES` by default) into the container, so a setting can be
benchmarked without building an image.

## On EC2

Instances come from the `ec2` helper on PATH (`~/dotfiles/scripts/aws/ec2.sh`).
Its `clickbench` profile provisions what the benchmark needs: Ubuntu 26.04, a
500 GB gp2 root volume, the `aws` key pair, S3 credentials, and a clone of this
fork in `~/clickbench`. It also writes an SSH config entry, so the instance is
reachable under the name it was given.

```bash
ec2 --create c6a.metal --profile clickbench --name umbra-c6a-metal
ssh umbra-c6a-metal 'cd clickbench && git checkout schmidt/results'
ssh umbra-c6a-metal 'cd clickbench && MACHINE=c6a.metal VERSION=26.09 ./run-local.sh umbra'
scp 'umbra-c6a-metal:clickbench/umbra/results/*/c6a.metal.*.json' umbra/results/$(date -u +%Y%m%d)/
ec2 --delete <instance-id> -y
```

`ec2` with no arguments lists instances and their state. Metal instances are
billed by the hour, so delete them when the run is done.

A native umbra run takes about 25 minutes including the load and the
concurrency window; the parquet systems add the dataset download. Run several
systems by invoking `run-local.sh` once per system.
