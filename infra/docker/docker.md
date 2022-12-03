# Spark-on-Docker Samples

This folder will be used to see how we could provision a Spark cluster using Docker. While this is an interesting exercise to reason about some of the implications, ask yourself first if this makes sense at all before going to production.

## Bare-bones Docker Image

By default, the image will run is pointing to `pyspark`, so running it without parameters will display directly the python repl:

```
$ docker run -ti luisbelloch/spark
```

### Running PySpark samples

We've included an script to easily run scripts in the [spark](../../spark) folder. To run any of the scripts, simply do:

```
$ cd data_processing_course/spark
$ ./spark compras_conversion_a_dolares.py
```

Please pay attention to the dot before the name of the script, `./spark`. The docker container has access to all the scripts in that folder, included the `data` folder on it:

```python
txt = sc.textFile('./data/compras_tiny.csv')
```

### Using the image without the "spark" helper script

Remember that inside the container you won't have access to the samples or data files we'll use in classroom. You'll have to mount a volume with them, [using -v option](https://docs.docker.com/engine/tutorials/dockervolumes). The local folder cannot contain relative routes, use `readlink` command to convert it to an absolute one.

```
$ docker run \
  -v $(readlink -f ../../spark):/opt/samples \
  -w /opt/samples \
  -ti luisbelloch/spark spark-submit /opt/samples/compras_con_mas_de_un_descuento.py
```

That should spawn a new container and run the job inside it. We've also mounted the samples folder in `/opt/samples` inside the container. All the executables from the Spark distribution are available in the container's path.

### How to build the images

Images are available in [Docker Hub](https://hub.docker.com/r/luisbelloch/spark/), you can easily modify and rebuild them:

```
$ docker build -t luisbelloch/spark .
$ docker tag luisbelloch/spark:2.10 luisbelloch/spark:latest
```

### Running Spark Master \ Workers

Variable `SPARK_NO_DAEMONIZE` is already set in the `Dockerfile`, it will make start scripts to run foreground instead of leaving the process in the background.

First step should be to start the master node. We've exposed ports 8080 (UI) and 7077 (Spark).

```
$ docker run -p 8080:8080 -p 7077:7077 -d luisbelloch/spark start-master.sh
```

Note that workers connect to master node through 7077 exposed to actual physical machine. Remember to configure port forwarding if you run docker inside a virtual machine.

After it starts, go to [localhost:8080](http://localhost:8080) and get the master URL. In our case is `spark://11168790f9c1:7077`. You will also need the container alias, `nervous_noyce`, to enable a link between master and worker containers. List containers with `docker ps` to retrieve it.

```
$ docker ps
CONTAINER ID   IMAGE               NAMES
11168790f9c1   luisbelloch/spark   nervous_noyce

$ docker run -p 8081:8081 \
    --link nervous_noyce \
    -d luisbelloch/spark start-worker.sh spark://11168790f9c1:7077
```

The worker node should be displayed in the master UI.

Remember that if you want to run jobs against those containers you need to point `spark-submit` or `pyspark` to the master node. To do it, add the option `--master` and set the URL that you copied from master node web page:

```
$ docker run -p 8081:8081 \
    --link nervous_noyce \
    -ti luisbelloch/spark pyspark \
    --master spark://11168790f9c1:7077
```

## Using Docker Compose

To bring up a mini-cluster with a master node and one worker:

```
$ docker compose up
```

The master UI should be available at [localhost:8080](http://localhost:8080).

Then you can also connect to it via `pyspark`:

```
$ docker compose run -p 4040:4040 master pyspark --master spark://master:7077
```

Running `docker ps` will show containers and their ports mapped. Workers can connect to master using internal DNS resolution, we've exposed the master node as `master`. Note that exposing worker nodes port is not straight-forward, we'll discuss that in class.

To scale up/down the cluster:

```
$ docker compose scale worker=3
```

Beware desired state persist between runs.
