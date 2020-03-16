# Docker Networks and performance

Docker key concepts: **docker-compose**, **dockerfile**, **networks**. 

I will show you:

1. 2 `docker-compose.yaml` files with working networks.
2. Interconnection between containers.
3. A time performance comparison between 3 scenarios:
   * container connected with container using a docker network.
   * container connected with a local application.
   * local application connected with other local application.

## The basics

The `docker-compose` commands uses a `docker-compose.yml` file to build one or multiple images, run them and then start the containers that you need.

Usually you will need two commands:

1. `docker-compose build` → Creates all images.
2. `docker-compose up` → Starts all needed containers. If `build` was not used before it will automatically `build`.

**CAREFUL!**: If you change an image and run `docker-compose up` it **will not** rebuild it if it previously existed.

To avoid this I recommend to get used to using both commands.

There are other useful commands:

- `docker-compose down` → Gets rid of all the containers and images for a certain `docker-compose.yml` file. Leaves your system as if `build` and `up` were never executed.

If you have any **network problem** in the office network see next section.

### Proxy

In order to be able to use docker from Indra's network, you must:

1. Add proxy variables to docker desktop (right click on icon and then setting, proxies):
   - Web Server: `http://proxy.indra.es:8080`
   - Check "Use same for both"
2. Add proxy variables when executing a build command:

```bash
docker-compose build --build-arg http_proxy=http://proxy.indra.es:8080 --build-arg https_proxy=http://proxy.indra.es:8080
```

## Code provided

1. `pinger/docker-compose.yaml` &rarr; Starts two containers that are only used to ping each other and check the communication between them.
2. `pinger/Dockerfile.pinger` &rarr; Builds the image for a pinger container.
3. `webserver/docker-compose.yaml` &rarr; Starts two containers that are basic webservers that automatically start on port 80.
4. `webserver/Dockerfile.webserver` &rarr; Builds the image for a webserver container (httpd).
5. `webserver/curl-format.txt` &rarr; Format file for time statistics extracted from a curl.
6. README.md &rarr; The one you are reading :stuck_out_tongue_winking_eye:

## Steps to follow

### Do networks really work?

First let's run some containers:

```bash
# -d (daemon) starts both containers in the background
docker-compose up -d
```

Open two different terminals and execute all top commands in one and the bottom commands in the other:

1. Open a terminal in each container:

   ```bash
   docker exec -it pinger_a bash
   ```

   ```bash
   docker exec -it pinger_b bash
   ```

2. Check connectivity between containers:

   ```bash
   ping pinger_b -c 10
   ```

   ```bash
   ping pinger_b -c 10
   ```

   It works! They can see each other.

   Can I do the same with **hostnames**? No, container_names have to be unique but hostnames don't. As a consequence docker does not allow to use the hostname to redirect, it might blow up the network. [More info](https://github.com/docker/compose/issues/2925)

3. Write down the last lines somewhere for performance comparison, mine were:

   ```
   rtt min/avg/max/mdev = 0.160/0.211/0.283/0.047 ms
   rtt min/avg/max/mdev = 0.054/0.169/0.286/0.063 ms
   ```

4. Execute the ping locally (in a new terminal) and compare the results:

   ```bash
   ping localhost -c 10
   ```

   ```
   rtt min/avg/max/mdev = 0.033/0.103/0.156/0.030 ms
   ```

   If you put a lot of packets in the performance comparison will be much more accurate.

4. Now check the number of hops between containers:

   ```bash
   traceroute -m 40 pinger_b
   ```

   ```bash
   traceroute -m 40 pinger_a
   ```

   Only one for both cases, it makes sense:

   ```
   traceroute to pinger_b (172.18.0.3), 40 hops max
     1   172.18.0.3  0.016ms  0.011ms  0.011ms 
   traceroute to pinger_a (172.18.0.2), 40 hops max
     1   172.18.0.2  0.017ms  0.012ms  0.013ms 
   ```

### What if I don't use networks?

If you don't use networks you must be connecting a local running app with a container. You might think that is more efficient. Let's check it out!

For this example change to the `/webserver` directory we are going to start the server and compare connecting two containers vs. connecting a local application with a container.

```bash
# starting both containers
docker-compose up -d 
```

1. From that directory, execute:

   ```bash
   curl -w "@curl-format.txt" -s "http://localhost:9090"
   ```

   This is equivalent to not using a network and connecting a local application with a container. For example a java application with a containerized db.

   My results (those are cumulative timestamps):

   ``` 
   <html><body><h1>It works!</h1></body></html>
     time_namelookup:  0.002770
          time_connect:  0.003483
       time_appconnect:  0.000000
      time_pretransfer:  0.003710
         time_redirect:  0.000000
    time_starttransfer:  0.005924
                       ----------
            time_total:  0.006043
   ```

2. Get into the containers:

   ```bash
   docker exec -it webserver_a bash
   ```

   ```bash
   docker exec -it webserver_b bash
   ```

3. Test the connection:

   ```bash
   # From webserver_a
   curl -w "@curl-format.txt" -s "http://webserver_b"
   ```

   This is equivalent to using a network and connecting two containers. For example a [quarkus java containerized](https://quarkus.io/guides/building-native-image) image with a containerized db.

   My results:

   ```
   <html><body><h1>It works!</h1></body></html>
     time_namelookup:  0.001383
          time_connect:  0.001774
       time_appconnect:  0.000000
      time_pretransfer:  0.002072
         time_redirect:  0.000000
    time_starttransfer:  0.002905
                       ----------
            time_total:  0.003017
   ```

4. Comparison: 

   I **recommend** to try both curls a lot of times because the values are really variable. We cannot say that communication between containers is much faster but we can say it is, at least, as good if not better.

### What if I don't use docker?

We can install the same HTTP server in our local linux distribution and perform the same test without using docker.

1. Install the `httpd` server:

   ```bash
   # In Fedora:
   sudo dnf install httpd
   ```

2.  Start the server:

   ```bash
   # In Fedora:
   systemctl start httpd
   ```

3. Throw the test again:

   ```bash
   curl -w "@curl-format.txt" -s "http://localhost"
   ```

   Amazing we see similar results! 

   ```
     ...
     time_namelookup:  0.001647
          time_connect:  0.001965
       time_appconnect:  0.000000
      time_pretransfer:  0.002132
         time_redirect:  0.000000
    time_starttransfer:  0.002903
                       ----------
            time_total:  0.003762
   ```

4. Comparison: 

   Again, I **recommend** to try the curl a lot of times because the values are really variable. We cannot say that communication between containers is much faster than between local services but they are quite similar.

## Conclusion

Use docker and use networks!!! They are easy, useful and performance does not suffer at all.

## Before leaving

Leave your computer clean! Go to the base directory and execute:

```bash
docker-compose -f pinger/docker-compose.yml down && docker-compose -f webserver/docker-compose.yml down && systemctl stop httpd
```

## More info

1. How to write a `docker-compose.yaml` &rarr; https://docs.docker.com/compose/compose-file/
2. Docker networks in depth &rarr; https://docs.docker.com/network/