---
layout:           post
title:            "A Control Plane for Containerized Envoy Proxies"
category:         "Cloud Native Computing"
tags:             envoy-proxy control-plane service-mesh container docker swarm
permalink:        /blog/a-control-plane-for-containerized-envoy-proxies
last_modified_at: "2023-07-14"
---

In this post, I would like to document briefly how I implemented my own [Envoy control plane](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/configuration-dynamic-control-plane) utilizing [Envoy's data plane API](https://github.com/envoyproxy/go-control-plane). The demo presented here is built for Docker swarm where dynamic resources for Envoy are updated by parsing labels of swarm services. Thus, the control plane should be run on a manager node. The demo code can be found in [this GitHub repository](https://github.com/xyxj1024/envoy-playground/tree/main/demo6). Another demo for Envoy running natively can be found [here](https://github.com/xyxj1024/envoy-playground/tree/main/demo3). The blog post, "[Guidance for Building a Control Plane to Manage Envoy Proxy at the edge, as a gateway, or in a mesh](https://blog.christianposta.com/envoy/guidance-for-building-a-control-plane-to-manage-envoy-proxy-based-infrastructure/)," by [Christian Posta](https://github.com/christian-posta) is highly recommended before jumping into the topic of Envoy dynamic configuration.

<!-- excerpt-end -->

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Docker Swarm Mode and Overlay Network

Docker Swarm mode enables multiple containers across multiple host machines to run together in a cluster (a "*swarm*"). A *node* is an instance of the Docker engine participating in the swarm. We can use the following command to create a swarm:

```bash
docker swarm init
```

Docker Swarm mode uses [overlay network](https://book.systemsapproach.org/applications/overlays.html). The [`overlay` network driver](https://docs.docker.com/network/drivers/overlay/) creates a distributed network among multiple Docker daemon hosts. This network sits on top of (overlays) the host-specific networks, allowing containers connected to it (including swarm service containers) to communicate securely when encryption is enabled. We can use the following command to create a named overlay network ("`mesh-traffic`") to which our manually started containers attach:

```bash
docker network create --driver=overlay --attachable mesh-traffic
```

In my demo setup, Envoy is always created as a swarm service specified with three port publishing rules: one for the admin port, one for the listener port, and one for the gRPC port[^1].

## A Watcher for Docker Label Updates

A swarm service may carry a label, which is actually a key-value pair storing some metadata for the service. A service-level label can be updated with:

```bash
docker service update --label-add $KEY=$VALUE $SERVICE_NAME
```

I implemented a simple watcher for my Envoy control plane operating in the way that it generates a new [snapshot](https://pkg.go.dev/github.com/envoyproxy/go-control-plane/pkg/cache/v3#Snapshot) whenever it detects a swarm service update.

Below are some data fields that I would like to configure dynamically:

```go
type ServiceStatus struct {
	NodeID string
}

type ServiceListener struct {
	Port types.SocketAddress_PortValue
}

type ServiceEndpoint struct {
	RequestTimeout time.Duration
	Protocol       types.SocketAddress_Protocol
	Port           types.SocketAddress_PortValue
}

type ServiceRoute struct {
	UpstreamHost string
	Domain       string
	PathPrefix   string
}

type ServiceLabels struct {
	Status   ServiceStatus
	Listener ServiceListener
	Endpoint ServiceEndpoint
	Route    ServiceRoute
}
```

The `ServiceLabels` structure will be passed to the following `Discover` function via a [Go channel](https://dave.cheney.net/2014/03/19/channel-axioms):

```go
type Manager struct {
	snapshotCache               cache.SnapshotCache
	clusters, listeners, routes []types.Resource
}

func (m *Manager) Discover(updateChannel chan ServiceLabels, ctx context.Context) {
	for {
		update := <-updateChannel
		if reflect.DeepEqual(update, ServiceLabels{}) {
			continue
		}

		m.updateConfiguration(update, ctx)

		time.Sleep(30 * time.Second)
	}
}
```

The `main.go` program, which serves as the entrypoint of this demo, starts two [goroutines](https://notes.shichao.io/gopl/ch8/), one running the above function and the other running the xDS [management server](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/mgmt_server).

The [Go client for the Docker Engine API](https://pkg.go.dev/github.com/docker/docker/client) provides a [`ServiceList`](https://pkg.go.dev/github.com/docker/docker/client#Client.ServiceList) method that can be used to retrieve the labels information we want:

```go
func StartWatcher(ctx context.Context, cli docker.APIClient, ingressNetwork string, updateChannel chan snapshot.ServiceLabels) {
	events, errorEvent := cli.Events(ctx, types.EventsOptions{
		Filters: filters.NewArgs(filters.KeyValuePair{Key: "type", Value: "service"}),
	})
	/* Docker services report the following events:
	 * - create
	 * - remove
	 * - update
	 */
	for {
		select {
		case err := <-errorEvent:
			logrus.Errorf(err.Error())
			StartWatcher(ctx, cli, ingressNetwork, updateChannel)

		case event := <-events:
			logrus.WithFields(logrus.Fields{"type": event.Type, "action": event.Action}).Debugf("Docker swarm service event received")

			if event.Action == "create" {
				continue
			}

			ingress, err := getIngressNetwork(ctx, cli, ingressNetwork)
			if err != nil {
				return
			}

			fmt.Println("Please enter the service name to be updated: ")
			userInput := bufio.NewScanner(os.Stdin)
			userInput.Scan()
			serviceName := string(userInput.Text())

			args := filters.NewArgs()
			args.Add("name", serviceName)
			services, err := cli.ServiceList(context.Background(), types.ServiceListOptions{Filters: args})
			if err != nil {
				return
			}
			if len(services) != 1 {
				logrus.Println("Something went wrong")
				logrus.Println("Count:", len(services))
				logrus.Printf("Service %s did not found", serviceName)
				return
			}

			for _, service := range services {
				if !isInIngressNetwork(&service, &ingress) {
					logrus.Warnf("Service is not connected to the ingress network, stopping processing")
					return
				}

				labels := snapshot.ParseServiceLabels(service.Spec.Annotations.Labels)
				if err = labels.Validate(); err != nil {
					logrus.Debugf("Skipping service because labels are invalid: %s", err.Error())
					return
				}

				updateChannel <- *labels
			}
		}
	}
}
```

In the `main.go` program, I write this `generateWatcher` helper function to generate a new watcher and obtain the initial (empty) update channel:

```go
func generateWatcher(ctx context.Context) chan snapshot.ServiceLabels {
	updateChannel := make(chan snapshot.ServiceLabels)

	go watcher.StartWatcher(
		ctx,
		newDockerClient(),
		ingressNetwork,
		updateChannel,
	)

	go watcher.InitUpdateChannel(updateChannel)

	return updateChannel
}
```

## Upstream Service

In [my previous control plane demo](https://github.com/xyxj1024/envoy-playground/tree/main/demo3), one single, standalone Envoy proxy is configured to send requests to upstream IP addresses such as `www.google.com/robots.txt` and `www.wustl.edu/robots.txt`. We might also want Envoy to connect to our own application services. To illustrate this, I use the following program:

```go
package main

import (
	"fmt"
	"log"
	"net"
	"net/http"
)

func redServer(rw http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(rw, "red")
}

func main() {
	l, err := net.Listen("tcp", ":8080")
	if err != nil {
		log.Fatal("listen error:", err)
	}

	go http.Serve(l, http.HandlerFunc(redServer))

	select {}
}
```

If running inside a Docker container, it needs a Dockerfile like this:

```dockerfile
FROM golang:1.20.5-alpine3.18

EXPOSE 8080

WORKDIR /app
COPY app.go ./

CMD ["go", "run", "app.go"]
```

Deploy it as a swarm service:

```bash
docker image build --tag app-1:v1 $APPDIR
docker service create \
    --publish 8080:8080 \
    --network mesh-traffic \
    --name app-1 app-1:v1
```

Finally, we can update our Envoy service as follows:

```bash
docker service update \
    --label-add envoy.status.node-id=local_node_1 \
    --label-add envoy.listener.port=10000 \
    --label-add envoy.endpoint.port=8080 \
    --label-add envoy.route.domain=example.com \
    --label-add envoy.route.upstream-host=app-1 \
    envoy-1
```

where the application service name is used as the upstream host.

[^1]: The control plane communicates with Envoy instances via the [xDS protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol). In my case, this communication takes the form of bi-directional gRPC streaming. Therefore, the gRPC port is exclusively used for dynamic configuration. If multiple Envoy instances want to communicate with one single control plane instance, they need to configure the same port value. When Envoy instances are Dockerized while the control plane instance is running natively, in Envoy's bootstrap config YAML file, we need to explicitly set the control plane's endpoint address to host machine's IP address. Since I'm using Docker Desktop for macOS, which actually launches a Linux VM for containers, `host.docker.internal` can be used. On a Linux machine, one might want to use [this program](https://github.com/xyxj1024/envoy-playground/blob/main/utils/linux_host_ip.sh) to determine this address.