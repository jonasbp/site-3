---
title: 12 factor revisited
author: Cameron Gray
twitter: camerondgray
---

It’s been over three years since we published [this](https://convox.com/blog/modern-twelve-factor-apps-with-docker/). 
In that time [12 factor](https://12factor.net/) has continued to be the dominant philosophy for building scalable, secure, and maintainable web apps. 
It has even been applied to the rising popularity of [serverless applications](https://aws.amazon.com/blogs/compute/applying-the-twelve-factor-app-methodology-to-serverless-applications/)

In that time Convox has come a long way as well and we have made building 12 factor apps using Docker even easier. It seems like it’s time to take stake of where we stand today and where we are going.

<!--more-->

### I. Codebase: - One codebase tracked in revision control, many deploys
This one seems pretty straightforward. If your team isn't using source control just stop reading now and go fix that! 
That said, in world of microservices and distributed applications it is important to remind ourselves that each app should have it's own repository 
and shared code should be consolidated into libraries that are built outside of the application build process. Of course Convox makes tying this all together easy using workflows and our [integrations](https://docs.convox.com/console/integrations) with Github and GitLab

### II. Dependencies: - Explicitly declare and isolate dependencies
This is often one of the most misunderstood factors. The aspect of this that we think is critical is 

`"A twelve factor app never relies on implicit existence of system wide packages"`

This is of course leads to why we believe containers are the right way to  build apps. A container is explicitly defined and configured. 
Diving a bit deeper, using things like [base images](https://docs.docker.com/develop/develop-images/baseimages/) and [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) 
can help you follow this rule while still having fast build times both locally and in production.

Convox takes this principle one step further  by introducing the [Convox.yml](https://docs.convox.com/reference/convox-yml) manifest which allows you to explicilty define not just your app(s) 
but the entire ecosystem of supporting resources and services that make your app work. Paired with a [local rack](https://docs.convox.com/development/running-locally) you can ensure that all your developers not only have identical configurations 
but also that their local configurations precisely mirror your production environment. 

Our goal is that, with a properly structured app, a new developer on your team simply needs to pull your latest code, install convox, and run `Convox start` to have a fully configured and working locally environment that exactly mirrors your production environment. 

### III. Config: - Store config in the environment
If you have usernames, keys, or passwords stored in your codebase go read [this](https://medium.com/@magoo/responding-to-typical-breaches-on-aws-28d6fe4071d0) now. There is no good argument for storing your configuration in your code period!
In addition to protecting against a breach, storing environment specific configurations in the target environments keeps your codebase clean and ensures that you don't have leakage between environments.
It also makes it really easy to spin up a new environment without changing code.

At Convox we try and make it easy to do this. First we automatically inject your local environment variables into your locally running container. 
Second, we allow you to specify local development defaults for your [environment](https://docs.convox.com/management/environment) variables in your convox.yml.
For your staging and production [Racks](https://docs.convox.com/introduction/rack) that are running on AWS we securely store your environment variables in [KMS](https://aws.amazon.com/kms/)
which we then inject into your containers as they are spun up. Finally we help you manage changes to your environment variables by automatically creating and promoting a new [release](https://docs.convox.com/deployment/releases) whenever an environment variable is changed which allows you to rollback environment changes as well as keep your code and environment in sync.

### IV. Backing services: - Treat backing services as attached resources
Almost all apps these days make use of at least a database and typically many more services such as a cache or an SMTP service.
The twelve factor guidelines state that whether a service is local or external they should be treated the same and accessed using either a URL configuration or 
some other set of configuration variables stored in the environment.

The benefits of this approach become clear when you think about how in a local development environment you might be using database server running in a docker container
while in production you are more likely connecting to a service such as [RDS](https://aws.amazon.com/rds/). If you use URL connection strings stored in the environment  your code will seamlessly move from local to production. 
This will also make it much smoother if you need to perform a database recovery or region move. 

Convox makes this complete seamless. When you define a database resource in your [convox.yml](https://docs.convox.com/reference/convox-yml) and run locally, Convox automatically spins up a docker container running the requested resource 
and injects the URL to that resource into the environment of all services that depend on that resource. When you deploy your application to production Convox will automatically provision the required database resource on RDS and inject that URL into your containers.

Convox also provides a robust set of [resources](https://docs.convox.com/resources/about-resources) as well as the ability to proxy to remote resources for people with more complex needs.

### V. Build, release, run: - Strictly separate build and run stages
The 12 factor approach dictates separating production deploy into the three distinct stages of Build, Release and Run. The critical aspect of these stages is that building a release should be a separate task from running a release. 
* In the build step we are bundling/compiling our code from the repository into a single deployable unit. In the case of a containerized application this includes creating an image of the container.
Convox isolates this step with [build](https://docs.convox.com/deployment/builds) which either be executed from the Convox cli with `convox build` or can be integrated into a [workflow](https://docs.convox.com/console/workflows). 
By isolating this step we create an immutable version of our application including the container image. Convox then combines that build with the desired environment configuration to create a promotable [release](https://docs.convox.com/deployment/releases). 
Most importantly, we can always rollback to previous release if we introduce a code or configuration change that cases problems in production

* In the run stage we are promoting a specific release into an environment. This may be the current release we just built or it may be a previous release that we are rolling back to.
Convox handles release with a [rolling deployment](https://docs.convox.com/deployment/rolling-updates) where new processes must check-in as healthy before they will start receiving traffic and old processes are only terminated as the new processes are available. 
Checking in as [healthy](https://docs.convox.com/deployment/health-checks) is an important feature of the Convox deploy. We encourage you to create a custom health check endpoint that truly verifies the stability of your application
(ie: database connectivity), this can prevent broken code from ever been deployed to production because it will never check-in healthy

One important note on rolling deploys and the ability to rollback with one click is you need to ensure changes to persistent services such as your database are backwards compatible by doing things like not removing columns without deprecating them first and making sure new colums accept null values or have default values.
You can read more about various approaches to this problem [here](https://blog.philipphauer.de/databases-challenge-continuous-delivery/).
The critical thing to keep in mind is that during every release your new code and your old code will be running at the same time and your application should be able to handle that.

### VI. Processes: - Execute the app as one or more stateless processes
One of the core aspects of running a containerized application is that all local storage is ephemeral. Taking this one step further the containers, and in the case of ECS, the hosts themselves are also ephemeral.
This generally forces you to follow this principal. At Convox we realize that certain applications may need to share data via a filesystem, 
or maintain a persistent cache which is why we support shared EFS [volumes](https://docs.convox.com/deployment/volumes) and automatically configured S3 [buckets](https://docs.convox.com/resources/s3)

### VII. Port binding: - Export services via port binding
This is one is something that most developers probably don't spend a lot of time thinking about but the gist of it is your application should contain it's own webserver and be bound to and listening on specific ports.
With a containerized application you are forced to do this because your application is bundled with your container image and you must explicitly define what ports you are exposing. 
The container approach does allow you a bit of flexibility however and many developers choose to include a proxy such as [NGINX](https://www.nginx.com/) or [HAProxy](http://www.haproxy.org/) as part of their image.

With Convox you define what [ports](https://docs.convox.com/deployment/port-mapping) your application listens on in your convox.yml file and Convox will automatically create an [application load balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) 
including provisioning any required [SSL](https://docs.convox.com/deployment/ssl) certificates so you can ensure your application is secure and only listening on the ports you want it to.
### VIII. Concurrency: - Scale out via the process model
With the rise of the popularity of microservices, this approach has become pretty well understood but even for simple, monolithic web applications there are some simple approaches that can be taken to achieve scale with this approach.
One popular tactic is to take your long running requests and make them asynchronous using a background job runner such as [celery](http://www.celeryproject.org/) or [sidekiq](https://github.com/mperham/sidekiq/).
This allows you to have a separate set of worker processes which you can scale independently from your web processes as opposed to trying to do something like managing threads within a single application.

Convox allows you define as many services as you need in your convox.yml manifest and they can all be either manually or automatically [scaled](https://docs.convox.com/management/scaling) independently. 
You can also define a service to be [internal](https://docs.convox.com/management/internal-services) if you only want it exposed to the other applications within your rack.
### IX. Disposability: - Maximize robustness with fast startup and graceful shutdown
In a world of rolling deployments, autoscaling, and one click rollbacks it's critical that your applications startup quickly and shutdown gracefully. 
While much of this is entirely dependant on your particular application stack, there certain decisions like using async job queues with built in retries, and avoiding things like sticky sessions that can make this approach easier for any stack.
### X. Dev/prod parity: - Keep development, staging, and production as similar as possible
(works on my machine image)
How many times has a developer on your team fully tested something locally after weeks of work and then realized it has to be redesigned once it its pushed to staging or production because of some fundamental difference between local development and production?
How long does it take your team to get a new developer up and running on your codebase? Do you have parts of your application that use backing services like a cache or queue that can't be tested locally?

Most of the platforms out there that make it super easy to deploy your application to production leave you completely on your own to figure out local development. At Convox we believe dev/prod parity is fundamental to modern application development. 
That's why we use a single manifest file to describe your complete environment for both local development and production. It's also why we automatically spin up local versions of your backing services in your local rack during development. 

For resources where it's simply not practical to run them in a local docker container, we offer [remote resources](https://docs.convox.com/development/remote-resources) where you can connect your local environment with a remote services through a secure proxy.

### XI. Logs: - Treat logs as event streams
With the ephemeral nature of containers it's important to not store things you might need in the future on the local filesystem.
This is particularly important for things like logs. In a world where our application is made up of many containers running on many hosts, logs are longer a system level concern. You need to be able to look at your application holistically by agreggating the logs from all the pieces that make up your application ecosystem if you are going to be able to get an accurate view of what's happening.

Convox makes this easy by automatically sending everything from stdout and stderr to [cloudwatch](https://aws.amazon.com/cloudwatch/) as well as all your rack level and ECS logs.
We also support sending your logs to any service that supports [syslog](https://docs.convox.com/resources/syslog) such as [papertrail](https://papertrailapp.com/).


### XII. Admin processes: - Run admin/management tasks as one-off processes
There are many reasons you might need to run a management task within your application. Some tasks such as running migrations or bundling static assets are done at release times. Other items such as cron jobs or maintenance scripts are run as part of the everyday function of your application.

The critical principle is that these task should not be run on the same processes that are handling your everyday traffic. Instead, you should fire up a one-off process using an identical release (code+environment) to prevent one off tasks from causing issues with your running applications. If you take this approach one step further you can run a one off  tasks against any specific release, including a release that is not yet promoted, which allows you to do things like run migrations before you release allowing zero-downtime deploys.

Convox of course has this built in with [one-off commands](https://docs.convox.com/management/one-off-commands) including support for detached processes for those long running data science jobs!

### What's next?
While frameworks, container strategies, and other application development trends come and go, the 12 factor approach seems to really be standing the test of time.
At Convox we remain all in on 12 factor and we continue to improve our platform to make it even easier to develop 12 factor apps. We are particularly focused on improving [workflows](https://docs.convox.com/console/workflows) for even faster and smoother deployments.
As well as improving our [local rack](https://docs.convox.com/development/running-locally) to make sure it 100% mimics production while still providing solid performance and not setting your laptop on fire! 

Our goal is that you shouldn't even have to think about the principles of 12 factor but they are just seamlessly enforced for you as develop and deploy your applications on Convox.

