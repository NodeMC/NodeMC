# NodeMC's Architecture

NodeMC is built with one idea in mind: it can any direction. If you need 10 Minecraft servers that emit 1000x events per second, you should be able to scale any component at will. This is achieved by the de-coupled microservices arch we use.


# Data Flow

![arch diagram](https://cdn.rawgit.com/NodeMC/NodeMC/5697e7db/.github/arch.png)

While this diagram may look confusing at first, let's break it down. We have currently 4 actual code services:

  * **api** - This builds endpoints and translates traffic to flow into our infra.
  * **router** - This routes internal jobs / events back into their relevant services.
  * **scheduler** - scheduler sets up minecraft containers and manages deploys
  * **metrics**- asynchronously consumes metrics and stores / drops them

Services communicate over redis by using `kue` a job queue based over redis, as well as using redis as a reguluar plain pub-sub interface. An
example flow over this could be:

1. Client hits `/v2/server/create`
2. API looks up translates this into an event, i.e send to router to aggregate service responses, or directly poll ArangoDB (i.e metrics)
3. API hits router with `server.create`, and data.
4. Router determines that `server.create` needs to hit the scheduler service and creates a new scheduler job, as well as emits a metrics job
5. (async) metrics does shit with the metrics event
6. An available scheduler instance picks up the job and provisions a new server, creating a `router.response` job with `success: true`
7. router hits API back with `success: true`
8. API responds with 200

Hopefully that makes it seem less complicated.

## Scaling (how we do it)

With this system we are able to tear down and throw up new instances without conflict, job queues operate on a first-come first-served basis eliminating any chance of a race condition, and if a job fails, it is immediately re-attempted until it hits a certain point in which we then have to return an error to the user.

When a scheduler has too many servers, or has too much load, it can easily un-subscribe itself from the job queue / ignore incoming jobs, and then will not create any more servers on that instance, while still being able to emit vital metrics and events to it's children. 
