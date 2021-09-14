# Container

- [Container](#container)
  - [Stateless](#stateless)
  - [Image](#image)
      - [Image pull policy](#image-pull-policy)
      - [Cache](#cache)
      - [Tag](#tag)

## Stateless

---

Before designing any solution, it's important to review all the basic and important points ~~we are tired of knowing~~:
scale, maintenance, architecture ... **infrastructure**. To build a service that will run in a container, you need to
keep in mind a basic and extremely important point:

- **Information must be stored externally**: The container has the basic characteristic of not caring about the state,
  that is, it can be restarted. Thus, it is necessary to ensure that the container works normally, even if this happens.
  If state storage is needed, such as a database, it's indicated that this is done on an external disk; In kubernetes
  this is done via [persistence volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

## Image

---

> Smallest and fastest possible!

In services that need to scale (in most cases), it's important to pay attention to the time it takes the service to be
ready for use, aka cold start. Specifically in kubernetes, we need to pay attention to the following steps:

#### Image pull policy

- **Never**: The agent (kubelet) starts the container if it has the image locally. Otherwise it fails to start.
- **Always**: The agent searches the image locally based on the same digest, if it doesn't exist, the agent searches
  externally. (**Preferred**)
- **IfNotPresent**: The agent looks for the external image only if it doesn't exist locally.

Default rules can be viewed [here](https://kubernetes.io/docs/concepts/containers/images/#imagepullpolicy-defaulting).

#### Cache

Just like pretty code, we don't want to repeat the same thing over and over again. So make sure the build follows at
least the following specifications:

- **If possible don't repeat the same thing over and over**: Imagine building a service that has external libs where
  they are downloaded every time to the container environment. This build tends to be very costly, as much of the
  process is consumed by downloading dependencies. In that case, cache the quirks and avoid rework.
- **Cache**: Each docker instruction is a layer with the result. Therefore, it is important to optimize this as much as
  possible, in addition to using other solutions like [Kaniko](https://github.com/GoogleContainerTools/kaniko). A
  crucial piece of information that few people pay attention to is that the layer order matters, that is, if the first
  instruction constantly changes, the remaining steps also need to be rebuilt, even if they are the same as the previous
  version. So keep the instructions that change frequently at the end of the build whenever possible.
- **Multi-stage build**: Separating the build into stages helps with readability and also helps to reduce the image size
  as the last stage is what really matters and the rest is discarded.

#### Tag

In productive environments, always keep tags explicit, this prevents image versions with unwanted ~~bugs~~ updates or
unexpected behavior from happening.

Here is an example of how to build a dockerfile:

```dockerfile
FROM debian:9 (Set image tag)
WORKDIR /myapp/
RUN apt-get update && apt-get install -y  \
  git \
  gcc (Avoid running commands that need local caching on different layers)
... 
(Always leave the constantly changing instructions like code last to take advantage of the cache)

FROM alpine:3.14 (Use multiple stages to simplify and reduce the image)
WORKDIR /root/
COPY --from=0 /myapp/ ./
CMD ...
```
