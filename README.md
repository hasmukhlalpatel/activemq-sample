# activemq-sample
[ActiveMQ Artemis](https://artemis.apache.org/components/artemis/documentation/latest/docker.html) and dotnet sample


## Run container manually
```sh
docker run --detach --name mycontainer -p 61616:61616 -p 8161:8161 --rm apache/artemis:latest-alpine
```
### Attached to running a container
You can also use the shell command to interact with the running broker using the default username & password artemis, e.g.:
```sh
docker exec -it mycontainer /var/lib/artemis-instance/bin/artemis shell --user artemis --password artemis
```
You can view the container’s logs using:

```
docker logs -f mycontainer
```

Stop the container using:
```sh
docker stop mycontainer
```
