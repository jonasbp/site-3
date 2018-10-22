---
title: 12 factor apps revisited
author: Cameron Gray
twitter: camerondgray
---

It’s been over three years since we published [this post on twelve factor apps](https://convox.com/blog/modern-twelve-factor-apps-with-docker/).  In that time [12 factor](https://12factor.net/) has continued to be the dominant philosophy for building scalable, secure, and maintainable web applications. It has even been applied to the rising popularity of [serverless applications](https://aws.amazon.com/blogs/compute/applying-the-twelve-factor-app-methodology-to-serverless-applications/)

In that time Convox has come a long way as well and we have made building 12 factor apps using Docker even easier. It seems like it’s time to revisit the principles of 12 factor apps and how we can follow them when creating containerized applications.

<!--more-->

### I. Codebase: - One codebase tracked in revision control, many deploys
This one seems pretty straightforward. If your team isn't using source control just stop reading now and go fix that! That said, in world of microservices and distributed applications it is important to remind ourselves that each app should have its own repository and shared code should be consolidated into libraries. Convox makes tying this all together easy using [workflows](https://docs.convox.com/console/workflows) and our [integrations](https://docs.convox.com/console/integrations) with Github and GitLab

### II. Dependencies: - Explicitly declare and isolate dependencies
This is one of the most misunderstood factors. The aspect of this that we think is critical is: 

`"A twelve factor app never relies on implicit existence of system wide packages"`

This rule leads to why we believe containers are the right way to  build apps. A container is explicitly defined and configured. Using Docker features such as [base images](https://docs.docker.com/develop/develop-images/baseimages/) can help you package up and share common dependencies.

Convox takes this principle one step further by introducing the [convox.yml](https://docs.convox.com/reference/convox-yml) manifest which allows you to explicitly define your application including the entire set of services and resources that work together to support it. Paired with a [local rack](https://docs.convox.com/development/running-locally), you can ensure that all your developers have identical development environments and that those environments precisely mirror production. 

With Convox, a new developer on your team simply needs to pull your latest code, and run `convox start` to boot the application and immediately start being productive. 

### III. Config: - Store config in the environment
Security best practices dictate that you should never store secrets directly in your code<sup>[1](https://medium.com/@magoo/responding-to-typical-breaches-on-aws-28d6fe4071d0)</sup>. Separating secrets from your codebase also makes it much easier to deploy different versions of your application such as staging and production.

At Convox, we make this easy. Convox allows you to easily configure application-specific environments and takes care of all the heavy lifting such as encryption-at-rest, runtime injection, and service-specific configurations. We also allow you to rollback environment changes if something goes awry.

### IV. Backing services: - Treat backing services as attached resources
Most applications these days make use of a database or other external services such as a cache or an SMTP server. The twelve-factor guidelines state that services should be accessed via a URL provided by an environment variable.

The benefits of this approach become clear when you think about how in a local development environment you might be using database server running in a docker container while in production you are more likely connecting to a service such as [RDS](https://aws.amazon.com/rds/). If you use URL connection strings stored in the environment  your code will seamlessly move from local to production. This will also make it much easier to perform a database recovery or upgrade. 

Convox makes this seamless. When you define a database resource in your [convox.yml](https://docs.convox.com/reference/convox-yml) Convox automatically spins up a container running the requested database when running locally and injects the URL to that resource into the environment. When you deploy your application to production Convox will automatically provision the required database resource on RDS and inject that URL instead.

Convox provides a robust set of [resources](https://docs.convox.com/resources/about-resources), as well as the ability to proxy to [remote resources](https://docs.convox.com/development/remote-resources), for teams with more complex needs.

### V. Build, release, run: - Strictly separate build and run stages
The twelve-factor approach dictates separating a production deploy into the three distinct stages of Build, Release and Run. The critical aspect of these stages is that building a release should be a separate task from running a release. 

* In the build stage you are compiling your code from source into a single deployable unit. For a containerized application, this means creating an image. On Convox this is as simple as running `convox build` in a directory containing your code or create a [workflow](https://docs.convox.com/console/workflows) to build automatically when you push new code to github or gitlab. 

* For the release stage, Convox combines your build with the application's current environment to create a [release](https://docs.convox.com/deployment/releases). Using atomic releases makes it incredibly easy to roll back to a previous version of your application if a problem is detected.

* In the run stage you are promoting a specific release to become the currently running version of your application. Convox handles releases with a [rolling deployment](https://docs.convox.com/deployment/rolling-updates) where new processes must check-in as healthy before they will start receiving traffic and old processes are terminated as the new processes are available. This provides you a great deal of security because code that does not pass a health check will never make it live.

### VI. Processes: - Execute the app as one or more stateless processes
One of the core aspects of running a containerized application is that the individual containers should all be treated as stateless. Having stateless containers allows you to scale your application up and down at will without negatively impacting the end users experience. Stateless containers also prevent a container failure from causing data loss. 
 
When you deploy an application that stores state within the containers you typically have to do things like enforce sticky sessions, where all requests for a specific client need to be directed to the same container because that container is the only place where that client's current state is stored. When you use sticky sessions it makes it extremely difficult to terminate a specific container because we either need to wait until all the clients whose states are stored on that container are gone or be willing to lose their state. Sticky sessions can also make scaling difficult because a recently added container can only take traffic from new clients because it lacks the state data for the existing clients. With a stateless application you can distribute your traffic evenly amongst your containers because each container can handle any request from new or existing clients.

In order to create a stateless application we need to treat all local storage as ephemeral. It is an especially a good idea to be stateless when we deploy a containerized application in the cloud because the containers, and often the hosts themselves, are also ephemeral. In order to ensure our containers are stateless we need to store any state data in a Database or some other backing service that all the containers have access to.

### VII. Port binding: - Export services via port binding
This principle is something that most developers probably don't spend a lot of time thinking about, but the gist of it is your application should contain its own webserver and be bound to, and listening on, specific ports. When you construct your application this way it makes it extremely easy to scale. You can simply run as many application instances as your traffic demands and put them all behind a single load balancer. Having a self contained application also makes it so your local and production environments are identical because you are not relying on any service within the environment in order for your application to respond. 

Overall this approach creates a simple, modular application, that is easy to scale and can be easily added as a resources to a larger system if need be. When creating a composite system made up of multiple applications or services, if each part exposes a single URL or port it’s much easier to tie all the pieces together.

With Convox you define what [ports](https://docs.convox.com/deployment/port-mapping) your application listens on in your convox.yml file and Convox will automatically create an [application load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) including provisioning any required [SSL](https://docs.convox.com/deployment/ssl) certificates so you can ensure your application is secure and only listening on the ports you want it to.

### VIII. Concurrency: - Scale out via the process model
With your application now composed of dependency-isolated processes that are exposing a single port, it becomes easy to horizontally scale your application by simply adding more copies of each service.

Convox allows you define multiple services in your convox.yml manifest and [scale](https://docs.convox.com/management/scaling) them independently. Convox also makes it easy to setup autoscaling based on metrics such as CPU load or targeted requests per minute.

### IX. Disposability: - Maximize robustness with fast startup and graceful shutdown
In a world of rolling deployments, autoscaling, and one click rollbacks it's critical that your applications startup quickly and shutdown gracefully.

### X. Dev/prod parity: - Keep development, staging, and production as similar as possible
How many times have you finished a feature only to have it fail spectacularly due to some difference between your local machine and production? How long does it take your team to get a new developer up and running on your codebase? Do you have parts of your application that use backing services like a cache or queue that can't be tested locally?

Many platforms that make it easy to deploy your application to production leave you completely on your own to figure out local development. At Convox we believe dev/prod parity is fundamental to modern application development. That's why we use a single manifest file to describe your application for both local development and production. 

For resources where it's simply not practical to run them in a local docker container, we offer [remote resources](https://docs.convox.com/development/remote-resources) where you can connect your local environment with remote services via a secure proxy.

### XI. Logs: - Treat logs as event streams
With the ephemeral nature of containers, it's important to not store things you might need in the future on the local filesystem. This is particularly important for logs. In a world where our application is made up of many containers running on many hosts, you need to be able to look at your application holistically by aggregating the logs to get an accurate view of what's happening.

Convox makes this easy by automatically capturing all the output from your containers, resources, and underlying AWS services and letting you view them with a simple `convox logs`


### XII. Admin processes: - Run admin/management tasks as one-off processes
There are many reasons you might need to run a one-off task within your application. Some tasks such as running migrations or packaging static assets are done at release time. Other items such as cron jobs or maintenance scripts are run as part of the everyday function of your application.

The critical principle is that these tasks should not be run on the same processes that are handling your everyday traffic. Instead, you should fire up a one-off process using an identical release (code+environment) to prevent one off tasks from causing issues with your running applications. If you take this approach one step further you can run a one-off tasks against any release, including a release that is not yet promoted, which allows you to do things like run migrations before you release allowing zero-downtime deploys.

Convox makes this easy with [scheduled tasks](https://docs.convox.com/gen1/scheduled-tasks) and [one-off commands](https://docs.convox.com/management/one-off-commands) including support for backgrounding processes for those long running data science jobs!

### What's next?
While frameworks, container strategies, and other application development trends come and go, the twelve-factor approach appears to be standing the test of time. At Convox, we remain all in on twelve-factor and we continue to improve our platform to make it even easier to develop 12 factor apps. We are particularly focused on improving [workflows](https://docs.convox.com/console/workflows) for even faster and smoother deployments. 

Our goal is to gently enforce these principles for you starting with your first line of code, so that deploying to production feels no more complicated than developing on your local machine.

