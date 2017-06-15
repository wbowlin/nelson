+++
layout = "single"
title  = "Home"
+++

<h1 id="overview" class="page-header">Overview</h1>

Nelson is a fully automated deployment orchestration tool intended to work with a datacenter scheduling system, such as Hashicorp Nomad or Apache Mesos. Nelson is responsible for application lifecycle, traffic routing specification and providing fully automated security policy management.

<h3 id="overview-philosophy" class="linkable">
  Philosophy
</h3>

* Uniformity is highly desirable / advantageous
* As many aspects to deployments as possible should be automated
* Automated lifecycle management is key to providing a holistic, scalable solution
* Batch and streaming jobs need to be prioritized within the workflow
* Manual promotion to production is high-cost since the "last-mile" of release management can cause friction
* Nelson supports manual deployment for those not yet comfortable with continuous deployment processes)

<h3 id="overview-features" class="linkable">
  Features
</h3>

* Fully integrated with GitHub or GitHub Enterprise
* Developer-driven, automated build & release workflow revisioned as code
* Support for multiple cluster managers, including [Hashicorp Nomad](https://www.nomadproject.io/)
* Compatible with any [Docker](https://www.docker.com) container
* Can deploy applications to any number of datacenters
* State of the art runtime routing via [Envoy](https://lyft.github.io/envoy/)
* Integrated support for alert definition and propagation via [Prometheus](https://prometheus.io/)
* Utilizes secure introduction for safe distribution of credentials from [Vault](https://www.vaultproject.io/)

<h3 id="overview-workflow" class="linkable">
  Workflow
</h3>

Nelson splits the build->deploy->release phases into two major parts, each with distinctly different properties.
First, the building, testing and propagating of containers. Second, the deployment,
validation, and migration of traffic. The following diagram illustrates a high-level
view of the first major part of the workflow.

<div class="clearing">
  <img src="images/high-level-workflow.png" />
  <small><em>Figure 1.0: workflow overview</em></small>
</div>

There are a range of automated steps at every phase of the build and release pipeline. Nelson ensures that it is fetching what it needs from the tagged GitHub repository. Only the most minimal element of metadata generated in the resulting release is needed to conduct a deployment. Nelson is agnostic to what language your application is built with or how it is tooled.You are free to integrate however you prefer, but we provide some tools to make doing so easy.

<h1 id="quickstart" class="page-header">Quickstart</h1>

The QuickStart guide is a high level document intended to provide migrating users with essential information to successfully leverage Nelson for deployment.

As a user of Nelson, we are making a series of assumptions that impact the way the system is ultimately used (for operator-level concerns, please see [the operator guide](#operator-guide)). Nelson users should have a working knowledge of the following:

* GitHub (Pull Requests, Git branches etc)
* Docker (practical usage of, knowledge of the general workflow)
* Usage of a terminal (assuming Darwin or Linux)

It is strongly suggested you become familiar with these requirements before proceeding.

<h2 id="quickstart-git" data-subheading-of="quickstart">Git-centric</h2>

Everything must be versioned. Nelson adopts a hardline position on configuration and changes made to systems at runtime: with two explicit exceptions, no application shall change its operable configuration. This means that your configuration is code and should be checked in along with your application. This includes the declarative manifest file that Nelson uses to receive its instructions. If you need configuration changes, modify it, check it in and redeploy.

The two exceptions to this are:

+ **Credential Distribution**: the act of providing your application with credentials. This includes passwords for databases, SSL certificates and so forth. Credential distribution is discussed in the next section.

+ **Traffic Shifting**: when deploying a service Nelson will move traffic from version `A` to version `B` at runtime, for any given system. Whilst this is not strictly "part of the application", it is the only other aspect of a container that is changing at runtime. Everything else is immutable.

Adopting this approach has several advantages:

+ The output Docker images are essentially dumb and the registry does not need to be tightly secured, because no private information or system secrets are in the container a-priori.

+ Every single change to the system has an audit trail, thanks to Git.

+ Reduced deployment risk: by deploying smaller sets of changes more frequently, you can know exactly what change set went into a particular version. There are several development workflows available with Nelson (covered below), so you can choose your deployment cadence and the amount of code that goes into a specific release.

Nelson does not support ad-hoc deployments. This is explicit and by-design. With this frame, Nelson works best in a poly-repo environment, where builds focus on small, atomic units. Nelson can work with explicit tagging and releasing for those with a mono-repo style of source control (more on this later).

<h2 id="quickstart-credentials" data-subheading-of="quickstart">Credentials</h2>

Nelson leverages secure introduction. This means that the container does not hold any secrets until the very moment it is launched, and that the credentials given to the container dictate the access permissions the container has. **You cannot interact with a system or resource that you did not tell Nelson about**, and we'll cover how you inform Nelson about your dependencies in the next section.

Practically speaking, credentials are sourced from [Vault](https://www.vaultproject.io/), and then mounted to a tempfs attached to the container. You can only source credentials from Vault, for which you have a valid Vault Policy. As it would turn out, these policies are dynamically generated by Nelson, and provisioned for you automatically. The credentials themselves should either be provisioned or configured by a member of staff with Vault access, and this should be done ahead of time, before you try to deploy your application with Nelson.

<h2 id="quickstart-dependencies" data-subheading-of="quickstart">Dependencies</h2>

Nearly all systems - services or jobs - have dependencies. Here are some examples:

+ Databases
+ Message Queue
+ AWS S3
+ Third-party services (anything not deployed by Nelson)

Within these dependencies resides several nuances. There are those dependencies which are deployed within the same datacenter infrastructure as your applications, and are internally provisioned. Other services, databases and message queues are the most common examples of dependencies.

There is a second category of dependencies, and these are broadly referred to as "resources". These are systems that an application may require to run, but are "external" to the datacenter target. Typically these are systems that are global from the callers perspective - ones that have an opaque URL for the caller. Any public or third-party service operates in this manner. Concrete examples of resources include Amazon S3, Google search service etc.

<h2 id="quickstart-ssh" data-subheading-of="quickstart">Zero Access</h2>

When you deploy applications with Nelson, there's no SSH access. Period. Your application containers are deployed onto one or more target datacenter clusters. The operations staff who setup the cluster will have access to the host worker nodes, but the host only provides compute resources. Having access to the nodes themselves adds no additional value from a debugging perspective, as **all monitoring and log data should be transferred out of the container**.

<h2 id="quickstart-consistency" data-subheading-of="quickstart">Consistency</h2>

By virtue of the fact that *Nelson* is orchestrating application deployments over potentially many target datacenters, it is important to realize that there is a subsequent lack of transactionality in the operations *Nelson* takes. This is most apparent when deploying a new revision of a particular system: application code can never assume it is a "singleton" or in some way special, as the minimum number of versions running - even if it's for a very short time - will be greater than one. *Nelson* will eventually deliver on its promise and make one revision the primary revision of a system by cleaning up the others, depending on the particular circumstances, that process might not be immediate.

This means that application builders have the following constraints:

1. Applications which require singleton behaviour can either choose to implement application-layer leader election, or use convergent data structures to make sure that all overlapping changes will always commute.

2. Data corruption can - and will - eventually happen and applications need to be able to recover from this. Typically this means checkpointing data writes to limit the blast radius of any potential corruption (more appropriate for batch-style processes), engineering teams should properly evaluate the possibility for corruption and recovery in their particular use case.

The authors of *Nelson* full appreciate that these constraints require more engineering work. However, by applying these constraints it means *Nelson* can provide a guarantee around several critical behaviours, and set the right expectation from the start about application lifecycle.

<h1 id="user-guide" class="page-header">User Guide</h1>

This user guide covers the information that a user should make themselves familiar with to both get started with Nelson, and also get the most out of the system. Certain parts of Nelson have been covered in more detail than would be typical for an entry-level user guide, but this is to make sure users are fully aware of the choices they are making when building out their deployment specification.

<h2 id="user-guide-installation" data-subheading-of="user-guide">Installation</h2>

The primary route for interacting with Nelson is via a command line client. The command line client provides most of the functionality the majority of users would want of Nelson. A future version of Nelson will also have a web-based user experience, which will have more statistical reporting functions and tools for auditing.

If you just want to use nelson-cli, then run the following to install it:

```
curl -GqL https://raw.githubusercontent.com/Verizon/nelson-cli/master/scripts/install | bash
```

This script will download and install the latest version and put it on your `$PATH`. We do not endorse piping scripts from the wire to `bash`, and you should read the script before executing the command. It will:

1. Fetch the latest version from Nexus
2. Verify the SHA1 sum
3. Extract the tarball
4. Copy nelson to `/usr/local/bin/nelson`

It is safe to rerun this script to keep nelson-cli current. Before getting started, ensure that you have followed these instructions the first time you install:

1. [Obtain a Github personal access token](https://help.github.com/articles/creating-an-access-token-for-command-line-use/) - ensure your token has the following scopes: `repo:*`, `admin:read:org`, `admin:repo_hook:*`.
2. Set the Github token into your environment: `export GITHUB_TOKEN=XXXXXXXXXXXXXXXX`
3. `nelson login nelson.example.com`, then you're ready to start using the other commands! If you're running the Nelson service insecurely - without SSL - then you need to pass the `--disable-tls` flag to the login command.

Then you're ready to use the CLI. The first command you should execute after install is `nelson login` which allows you to securely interact with the remote Nelson service.

<div class="alert alert-warning" role="alert">
⛔&nbsp; Note that currently the Nelson client can only be logged into <strong>one</strong> remote <em>Nelson</em> service at a time.
</div>

<h2 id="user-guide-terminology" data-subheading-of="user-guide">Terminology</h2>

In order to understand the rest of this user guide, there are a set of terms that are useful to understand:

<table class="table table-striped">
  <thead>
    <tr>
      <td width="15%"><strong>Term</strong></td>
      <td><strong>Description</strong></td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><em>Repository</em></td>
      <td>References one source code repository: this may contain multiple unit kinds, but represents one canonical release workflow (i.e. all units in the repository are updated and released together).</td>
    </tr>
    <tr>
      <td><em>Scheduler</em></td>
      <td>The element within the datacenter that actually makes work placement decisioning, based upon the available resources (e.g. CPU, RAM etc).</td>
    </tr>
    <tr>
      <td><em>Datacenter</em></td>
      <td>Represents a single failure domain - it is a <em>logical</em> ascription of some computing resources to which work can be assigned. In practice, a "datacenter" from Nelson's perspective may either be a single <code>scheduler</code> endpoint that directly maps to a single physical datacenter (which internally has redundant power domains), or it could be multiple virtual datacenters within a geographic region (e.g. AWS Availability Zone). The key thing is that Nelson's concept of datacenter is all about scheduling failure modes.</td>
    </tr>
    <tr>
      <td><em>Namespace</em></td>
      <td>Defines a virtual space for units to operate. One namespace cannot talk to a unit in another namespace. The namespace itself represents a collection of units. More often than not, these namespaces take names such as "dev", "qa" and "production". While it is common to think of these as environments, this term was specifically avoided because it typically carries with it the idea of physical separation, which does not exist in a shared computing environment.</td>
    </tr>
    <tr>
      <td><em>Plan</em></td>
      <td>Defines how a unit is to be deployed. It specifies the resource requirements, constraints, and environment variables.</td>
    </tr>
    <tr>
      <td><em>Unit</em></td>
      <td>Defines logically what is to be deployed. Units pertain specifically to the concept of something that is deployable at the high level. For example, one could have a unit <code>accounts</code> or <code>api-gateway</code>. Units are not versioned, and do not discriminate between how they are deployed, i.e. as a service (long lived and not expected to terminate) or job.</td>
    </tr>
    <tr>
      <td><em>Stack Name</em></td>
      <td>Whilst units are a logical concept only, stacks are the direct implementations of units. Specifically, a stack represents a unique deployment of a very particular unit, of a particular version number, along with a uniquely provisioned hash. For example: <code>accounts--2-8-287--roiac45o</code>.</td>
    </tr>
    <tr>
      <td><em>Feature Version</em></td>
      <td>Feature versions are the first two <code>major</code> and <code>minor</code> digits of a <a href="http://semver.org/" target="_blank">semantic version number</a>. Feature versions are always of the form <code>X.Y</code>, for example: <code>1.2</code>, <code>4.6</code> etc.</td>
    </tr>
  </tbody>
</table>

<h2 id="user-guide-cleanup" data-subheading-of="user-guide">Lifecycle</h2>

Every stack that gets deployed using Nelson is living on borrowed time. Everything will be removed at some point in the future, as the deployment environment "churns" with new versions, and older versions are phased out over time. This principal is baked into Nelson, as every single stack has an associated expiry time. With this in mind, each and every stack moves through a variety of states during its lifetime. The figure below details these states:

<div class="clearing">
  <img src="images/stack-states.png" />
  <small><em>Figure 2.0: stack states</em></small>
</div>

At first glance this appears overwhelming, as there are many states. Some of these states will be temporary for your particular stack deployment, but nonetheless, this is the complete picture. The following table enumerates the states and their associated transitions:

<table class="table table-striped">
  <thead>
    <tr>
      <td><strong>From</strong></td>
      <td><strong>To</strong></td>
      <td><strong>Description</strong></td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>pending</code></td>
      <td><code>deploying</code></td>
      <td>Internally, Nelson queues workflow execution, and if the maximum number of workflows are currently executing it will buffer tasks in memory until resources are available to execute the requested workflow. Once the workflow starts executing the stack will move into the <code>deploying</code> state.</td>
    </tr>
    <tr>
      <td><code>deploying</code></td>
      <td><code>failed</code></td>
      <td>In the event the workflow being executed fails for some reason - whatever the cause - the workflow moves the stack into the <code>failed</code> state so that Nelson knows that this stack did not get to a healthy point in its lifecycle, and is pending proper termination.</td>
    </tr>
    <tr>
      <td><code>deploying</code></td>
      <td><code>warming</code></td>
      <td>Once a stack gets started by Nelson, there is an optional chance to send synthetic traffic to the deployed stack to warm the process. This is common in JVM-based systems, where the JIT needs warming before production traffic is rolled onto the deployment, otherwise users may notice perceptible latency in request handling for entirely technical reasons. This state is optional, and the manner in which its implemented is entirely workflow dependent.</td>
    </tr>
    <tr>
      <td><code>warming</code></td>
      <td><code>ready</code></td>
      <td>Once warmed (if applicable) the deployment moves into the <code>ready</code> state. This is the steady state for any given stack, regardless of unit type, and regardless of the operational health of a system (from a monitoring perspective), Nelson will perceive the stack as logically ready.</td>
    </tr>
    <tr>
      <td><code>ready</code></td>
      <td><code>deprecated</code></td>
      <td>Over time, a particular development team for a given system might want to phase out an older version of their system, for some technical reason (e.g. in order to make a completely incompatible API change or upgrade user data in a backward compatible manner). In order to do this a development team will first need to ensure that all consumers of their API have moved to a newer version. Nelson provides the concept of "deprecation" to do this - a deprecated service cannot be depended on by subsequent deployments, once it has been marked as deprecated. Systems that were already deployed will continue to use the deprecated service until such time as they themselves are phased out and upgraded with newer stacks. </td>
    </tr>
    <tr>
      <td><code>ready</code></td>
      <td><code>garbage</code></td>
      <td>Over time different services may be refactored and altered in a variety of ways. This can often lead to orphaned service versions that just waste resources on the cluster, so to avoid this Nelson actively figures out what stacks depend on what other stacks, and transitions stacks to the <code>garbage</code> state if nobody is calling that stack.</td>
    </tr>
    <tr>
      <td><code>deprecated</code></td>
      <td><code>garbage</code></td>
      <td>Once no other stack depends on the deprecated stack, it will move to the <code>garbage</code> state.</td>
    </tr>
    <tr>
      <td><code>garbage</code></td>
      <td><code>terminated</code></td>
      <td>For any stack that is residing in the garbage state, there is a grace period of 24hrs for which that stack will be left running, and after the grace period the garbage is taken out and stacks are fully terminated and removed from the cluster. Terminated stacks are entirely stopped and consume zero resources on the cluster.</td>
    </tr>
  </tbody>
</table>

<h2 id="user-guide-manifest" data-subheading-of="user-guide">Manifest</h2>

One of the core tenets of the Nelson workflow is that all changes are checked into source code - nothing should be
done out of band in an untrackable manner. Nelson requires users to define a manifest file and
check it into the root of the repository. This file is called `.nelson.yml` (note the preceding `.` since it is a
UNIX dotfile), in which the deployable units are defined, along with any additional rules for monitoring, alerting and
scaling those units. Consider a simple example that launches a Hello World service:

```yaml
---
units:
  - name: hello-world
    description: >
      very simple example service that says
      hello world when the index resource is
      called via http
    ports:
      - default->9000/http

plans:
  - name: default

namespaces:
  - name: dev
    units:
      - ref: hello-world
      - plans:
         - default
```

This is the simplest `.nelson.yml` that one could define for a unit that exposes a service on port 9000. It declares a unit called `hello-world`, and then exposes a port. This is an example of unit that is typically referred to as a service. By comparison, units that are intended to be run periodically define a `schedule`. Such a unit is typically referred to as a job. Below is a simple example of a unit that is run hourly:

```yaml
---
units:
  - name: hello-world
    description: >
      very simple example job that prints
      hello world and exits right away

plans:
  - name: dev-plan
    schedule: hourly

namespaces:
  - name: dev
    units:
      - ref: howdy
        plans:
          - dev-plan
```

Here the two units are similar, but the second defines a `schedule` under the plans stanza, which indicates that the unit is to be run periodically as a `job`. Suggested further reading [in the reference](reference.html#manifest) about the Nelson manifest answers the following common queries:

* How do I declare a dependency on another unit?
* How do I get alerted when my unit has a problem at runtime?
* How do I expose my service to the outside world?

Now that you have your `.nelson.yml` as you want it, add the file to the **root** of your source repository, and commit it to the `master` branch. Nelson will look at the repositories `master` branch when it first attempts to validate your repository is something that is Nelson-compatible.

<h2 id="user-guide-credentials" data-subheading-of="user-guide">Credentials</h2>

Nelson fully supports building systems that are secure-by-default. This is specifically enabled by support for secure introduction. Every unit that gets deployed via Nelson are granted a token that allows them to read from [Vault](https://www.vaultproject.io/docs), which is the credential management system integrated with Nelson. For a primer on Vault, please [see the dedicated documentation site](https://www.vaultproject.io/intro/).

For users, the recommended approach to gain credential access is to provide explicit details to your system administration / operations team about the credential you want access too. That team will then create the necessary configuration in the Vault backend. Subsequent deployments after those credentials are provisioned will allow you to access Vault and read the relevant credential (e.g. database password, certificate etc).

<div class="alert alert-warning" role="alert">
  If your credential has not been provisioned access in Vault, then your container will not have access to the credentials you are expecting. It is <strong>highly</strong> recommended that your application does a sanity check for its credentials during the boot / initialization phase and in the event those required credentials are not present, the application moves into a relevant failure mode. This might be to terminate itself, or to suspend its bootup until the correct credentials are provisioned. Either way, this state should be logged or reported via monitoring.
</div>

With this frame, any information you require at runtime that is sensitive, or is non-public must be provided via [consul template](https://github.com/hashicorp/consul-template). As it turns out, requiring runtime details (credentials, for example) is very common, so let's consider an example in which we want to render a small [Knobs](http://verizon.github.io/knobs/) configuration file that we can include to get our database credentials. To do this, we'll need to make a template in your project, that once rendered will put the populated configuration file inside the container at `/opt/application/conf/runtime.cfg`. This document assumes you have consul-template running inside your container image and Nomad is being used as the scheduler (to leverage secure introduction):

```
{{with $ns := env "NELSON_ENV"}}
{{with secret (print "nelson/" $ns "/test/creds/howdy-http")}}
db.password = {{ $secret.Data.password }}
{{end}}
{{end}}
```

Here, the path being used in the `vault` stanza would be provided to users from the operations team after submitting a request for credentials. Whilst this path is static, Vault will either dynamically provision credentials for your calling application, or read a set of securely stored values from the generic mount.

<h2 id="user-guide-enabling-deployment" data-subheading-of="user-guide">Configure</h2>

With your repository in good shape, you're ready to enable Nelson to deploy your project whenever it sees a new release. To do this, visit the URL where Nelson is deployed, and login using your Github. Upon doing this for the first time, you will see the following prompt (or something similar, for your account):

<div class="clearing">
  <img src="images/enable-nelson-github.png" />
  <small><em>Figure 2.1: approve github access page</em></small>
</div>

Once authorized, you will be redirected to the Nelson home page, where you can enable your repositories for deployment. Nelson might take a moment to sync up your repositories with GitHub on your first login, so give it a moment and then once the list appears, you should see something like this:

<div class="clearing">
  <img src="images/repo-list.png" width="100%" />
  <small><em>Figure 2.2: list of repositories</em></small>
</div>

Use the search bar to look for your repository, and just slide the button on the left hand side to the "on" position. If for some reason you cannot enable your repository, please use the command line client or sbt-plugin to validate your Nelson manifest file.

<h2 id="user-guide-routing" data-subheading-of="user-guide">Routing</h2>

Nelson supports a dynamic discovery system, which leverages semantic versioning and control input from manifest declarations to make routing choices. This part of the system is perhaps one of the most complicated elements, so a brief overview is included here, and specific details can be found within the [reference section](reference.html#routing). To give a general overview, figure 3.1 highlights the various moving parts.

<div class="clearing">
  <img src="images/routing-design.png" height="70%" />
  <small><em>Figure 3.1: container routing</em></small>
</div>

Reviewing the figure, the runtime workflow works as follows:

A. The container is launched by the scheduler and consul-template interacts with a consul agent running on the surrounding host. Consul Template obtains a certificate from the PKI backend in Vault after negotiating with its secure Vault token.

B. Envoy boots using the certificates obtained from the first step, and then looks for `consort.service` in the local consul domain. Consort acts as a mediator between data stored in Consul and the protocol that Envoy uses for both SDS (converting stack names into IPs) and CDS (discovering the list of stacks). Envoy periodically reaches out to Consort to perform SDS and CDS, but never does so in the runtime hot-path.

C. Application boots and goes about its business. In the event the application is attempting to make a call to another service, it actually calls to "localhost" from the containers perspective, and hits the locally running Envoy instance. Envoy then wraps the request in mutual SSL and forwards it to the target service instance. In this way, the application itself is free from knowing about the details of mutual authentication and certificate chains. All calls made via Envoy are automatically circuit-broken and report metrics via StatsD. This is rolled up to a central StatsD server.

<h2 id="user-guide-lbs" data-subheading-of="user-guide">Load Balancers</h2>

In addition to service to service routing, Nelson also supports routing traffic into the runtime via "load balancers" (LBs). Nelson treats these as a logical concept, and supports multiple backend implementations; this means that if Nelson has one datacenter in AWS, it knows about [ELB](https://aws.amazon.com/elasticloadbalancing/) and so forth. If however, you have another datacenter not on a public cloud, Nelson can have alternative backends that do the needful to configure your datacenter for external traffic. The workflow at a high-level is depicted in figure 3.2.

<div class="clearing">
  <img src="images/lbs.png" height="70%" />
  <small><em>Figure 3.2: load balancer overview</em></small>
</div>

Typically load balancers are static at the edge of the network because external DNS is often mapped to them. This is clearly a mismatch between statically configured DNS and the very dynamic, scheduler-based infrastructure Nelson otherwise relies upon. To bridge this gap, Nelson makes the assumption that the LB in question is capable of dynamically updating its runtime routes, either via API directly, or via consul-template triggering updates and reloads on LB configuration (for example, reloading HAProxy config).

In order to achieve this dynamism on the load balancer, Nelson writes out a whole set of information about source and destination routes to the Consul in the appropriate datacenter. This is intended to then be used / consumed by whatever process needs to keep the load balancer up-to-date. The protocol is as follows:

```
[
  {
    "port_label": "default",
    "backend_stack": "foo--1-0-301--aovcp784",
    "frontend_port": 8444,
    "frontend_name": "default-foo--1-0-301--aovcp784"
  }
]
```

Whilst the fields should be self-explanatory, here are a list of definitions:

<table class="table table-striped">
  <thead>
    <tr>
      <td><strong>field</strong></td>
      <td><strong>description</strong></td>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>port_label</code></td>
      <td>Port label used to describe this particular port in the manifest definition. All ports require a named label.</td>
    </tr>
    <tr>
      <td><code>backend_stack</code></td>
      <td>The runtime name of the service to route traffic too. This name must match a known record in the consul service catalog.</td>
    </tr>
    <tr>
      <td><code>frontend_port</code></td>
      <td>Port used to expose traffic on the outside of the load balancer. The available ports are restricted by the configuration parameter <code>proxy-port-whitelist</code>.</td>
    </tr>
    <tr>
      <td><code>frontend_name</code></td>
      <td>Name of this particular route; largely informational and a simple convenience that can be used to disambiguate routes in a proxy configuration (irrespective if its NGINX, HAProxy, Envoy etc).</td>
    </tr>
  </tbody>
</table>

<h3 id="user-guide-lbs-manifest" class="linkable">Usage</h3>

Providing the manifest definition for LBs is very similar to that of the unit definitions. Consider the following example:

```
loadbalancers:
  - name: foo-lb
    routes:
      - name: foo
        expose: default->8444/http
        destination: foobar->default
      - name: bar
        expose: monitoring->8441/https
        destination: foobar->monitoring
```

Load balancers have their own top-level block in the manifest, as they are not strictly unit types and form their own special constraints (see [cleanup](#user-guide-lbs-cleanup) for more on that). The key part here is the `routes` block, which allows users to expose inbound routes (using the same syntax as `unit` port declarations), along with a `destination` that allows you to declare which internal system this LB depends on, and which labeled port on the downstream system should be routed too. By using these labels and not explicit ports, the LB and the downstream unit are not tightly coupled; service expose a logical set of functionality on a port, and the load balancer always routes to that functionality - based the label - even if it changes ports in the future.

<h3 id="user-guide-lbs-cleanup" class="linkable">Cleanup</h3>

<div class="alert alert-warning" role="alert">
  <strong>IMPORTANT</strong> If you create a load balancer - and no longer need it - you are responsible for cleaning it up.
</div>

With Nelson using its internal routing graph to calculate cleanup actions for the deployed systems, the load balancers act as a form of anchor for the wider system. In practice, this means that LBs form the caller for other service on the graph, which prevents those services from being cleaned up because they have inbound traffic from Nelson's perspective. Due to this, Nelson is not able to cleanup LBs because its general policy application cannot work - LBs are the edge of the world.

In the event that the downstream systems are cleaned up forcefully by hand, users must be sure to also remove the LBs to conserve resources (external IPs are, for example, a finite resource). This is particularly relevant if you are operating on a public cloud where the LB cluster would continue to cost you money.

<h3 id="user-guide-lbs-aws" class="linkable">Amazon Web Services</h3>

When using Nelson with a datacenter located in AWS, the system has to work within a set of constraints to meet the aforementioned dynamic versioning ability for downstream routing destinations. In order to achieve this, the "load balancer" on AWS looks like:

<div class="clearing">
  <img src="images/lbs-aws.png" height="70%" />
  <small><em>Figure 3.3: load balancer on AWS</em></small>
</div>

The proxy implementation itself should typically be a large container that is run in host networking mode, but implementors are free to bake whatever AMI they see fit for this component. Provided the implementor consumes the Nelson LB protocol from Consul, Nelson won't care how the LB is implemented.

<h3 id="user-guide-lbs-alternatives" class="linkable">Alternative Implementations</h3>

At the time of writing only the AWS load balancers were natively supported. Supporting additional modes of load balancing is straightforward.

<h2 id="user-guide-troubleshooting" data-subheading-of="user-guide">Troubleshooting</h2>

If you added your `.nelson.yml` file, and manifest validation is passing and you're chugging along making GitHub releases, then you will want to know what Nelson is doing with your repo, right? Well, the Nelson CLI has a set of useful utilities into what happened with your project deployments. First, you would want to list the datacenters available on your instance of Nelson:

```
nelson datacenters list
```

This will give you the list of datacenters that Nelson knows about and the namespaces available in each. For the sake of this document, lets assume the name of the datacenter is `texas`. Now we know *where* our deployment might be, we can ask Nelson to display all the things it deployed into that location:

```
nelson stacks list -d texas -ns dev
```

Here, we're asking for the `dev` namespace, but use whatever is applicable for your particular enquiry. All being well, if your application was launched you should see it in the list. If however, your application does not appear to be in the list then its possible there was some kind of problem, so we need to ask Nelson to tell us about things it deployed which `failed` or were `terminated`:

```
nelson stacks list -d texas -ns dev -s failed,terminated
```

If Nelson failed to deploy your stack for a technical reason, it should appear on this list. If you would like to know what the specific error is you can inspect the Nelson stack logs like so:

```
# where XXXXXX is the GUID from the `stacks list` command above.
nelson stacks logs XXXXXX
```

This should tell you exactly what Nelson did with your deployment. Every step in the workflow is logged to the workflow logger; the log shows literally everything Nelson did on your behalf. If the logs do not show any particular errors, then you might want to ask the runtime what the current state of play is, which can be done with the `runtime` command:

```
nelson stacks runtime XXXXXX
```

Nelson will return a summary response from the scheduler about what state the stack is *currently* in. In the event the runtime says that your stack is `pending` this means the requested datacenter scheduler is still looking for resources to satisfy your request.

<h1 id="operator-guide" class="page-header">Operator Guide</h1>

Nelson is typically deployed on-premises and should be co-located with network line of sight
with either github.com or your GitHub Enterprise; wherever you are storing your source code.
Nelson itself deploys as a docker container, so can be deployed pretty much anywhere, regardless of if its on bare metal with a systemd unit, or on a cloud container service like [AWS EC2](https://aws.amazon.com/ec2/) or [GCE](https://cloud.google.com/container-engine/). It is important to note that Nelson itself is explicitly designed to **not be high availability**. Every subsystem that Nelson talks to (Consul, Vault, etc) is redundant within the target datacenter.

<h2 id="install-machine" data-subheading-of="operator-guide">Installation</h2>

Nelson does not have a huge set of requirements - it is a fairly lightweight process. Most of
the tasks that Nelson is doing require CPU cycles and memory. With that being said,
propagating the docker containers from the stage registry to the remote locations can result
in the docker daemon using a lot of space on disk. With this in mind, the following specs are recommended:

* 8-16GB of RAM
* 100GB of disk space (preferably SSDs)
* Ubuntu 16.04

It is strongly advised to not use a RedHat-based OS for running Nelson. After a great deal of testing, Debian-based OS appears to be orders of magnitude faster running Docker than their RedHat counterparts.

<h3 id="install-launching" class="linkable">Launching Container</h3>

Typically one will be operating Nelson using a `systemd` unit, but one can run Nelson however it makes most
sense for your environment. The docker command you use to start the system should look something like the following:

```
docker run --name nelson \
 --log-driver journald \
 --env-file nelson.env \
 -p 8080:9000 \
 -v /var/lib/nelson/db:/opt/application/db \
 -v /var/run/docker.sock:/opt/application/docker.sock \
 -v /path/to/.docker/config.json:/root/.docker/config.json:ro
 -v /etc/nelson/datacenters.cfg:/opt/application/conf/datacenters.cfg:ro
 verizon/nelson:latest
```

This command - or a command like it - can be run from any system init system (system, upstart etc). The file mounts supplied (using `-v`) are used so that the Nelson database is stored on the host system, and not within the container. This allows Nelson to persist state over process reboots.

<div class="alert alert-warning" role="alert">
Nelson's database is a simple <a href="http://www.h2database.com/">H2</a> file-based datastore. Nelson is intended to be running as a singleton and currently does not support clustering. Clustering is planned for later support.
</div>

Nelson requires a set of environment variables to be filled out, and the contents of the aforementioned `nelson.env` is as follows:

```
NELSON_SECURITY_ENCRYPTION_KEY=xxxxxxxxxxxxxxxxxxxx
NELSON_SECURITY_SIGNATURE_KEY=xxxxxxxxxxxxxxxxxxxx
NELSON_GITHUB_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxx
NELSON_GITHUB_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxx
NELSON_GITHUB_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxx
NELSON_DOMAIN=your.nelson.domain.com
```

<h3 id="install-authorize" class="linkable">Authorize with Github</h3>

Nelson is implemented as a Github OAuth application, and it requires a one-time setup when installing it into your Github organization. If you are not familiar with how to setup a Github application, please see the [GitHub documentation site](https://developer.github.com/guides/basics-of-authentication/) for details.

Nelson should be registered with the exact domain on which the nelson process runs (take care select the right protocol - http vs https), and the callback URL should be: `https://your.nelson.domain.net/auth/exchange`. From a networking standpoint, provided Github can reach the Nelson server, and the domain specified in the OAuth application and that being used by the client then the system should work.

Be sure to take the `clientId` and `clientSecret` supplied by Github and add them to the nelson configuration mentioned above. This process should work equally well on both [github.com](https://github.com) and Github Enterprise.

<h3 id="install-telemetry" class="linkable">Telemetry</h3>

Nelson ships with out of the box support for gathering runtime telemetry with [Prometheus](https://prometheus.io/). The metrics being exported here detail all manner of internal details. Here are some of the key metrics:

<table class="table table-striped">
  <thead>
    <tr>
      <td><strong>metric</strong></td>
      <td><strong>description</strong></td>
    </tr>
  </thead>
  <tr>
    <td><code>helm_requests_failures_total</code></td>
    <td>Number of requests that fail when calling Consul</td>
  </tr>
  <tr>
    <td><code>helm_requests_latency_seconds</code></td>
    <td>Latency of requests being sent to Consul</td>
  </tr>
  <tr>
    <td><code>docker_requests_failures_total</code></td>
    <td>Number of requests that fail when calling the Docker Registry</td>
  </tr>
  <tr>
    <td><code>docker_requests_latency_seconds</code></td>
    <td>Latency of requests being sent to the Docker Registry</td>
  </tr>
    <tr>
    <td><code>vault_requests_failures_total</code></td>
    <td>Number of requests that fail when calling Vault</td>
  </tr>
  <tr>
    <td><code>vault_requests_latency_seconds</code></td>
    <td>Latency of requests being sent to Vault</td>
  </tr>
</table>

In addition to these key metrics, there are a variety of general system and JVM metrics that are being exported, and these should also be carefully monitored for issues. Naturally, this goes for any application and not just Nelson.

<h3 id="install-done" class="linkable">Installation Complete</h3>

With those steps complete, you should be able to browse to the Nelson URL and login using your Github account. If you encounter problems during this setup process, please check the logs from the Nelson container using `journalctl` on the host. Alternatively, install any of the generic log forwarding products (splunk, fluentd, logstash etc) to export the logs into an indexed aggregator (this is highly recommended for production usage).

<h2 id="install-datacenter" data-subheading-of="operator-guide">Failure Domains</h2>

To run Nelson you will need access to a target datacenter. One can consider this logical datacenter - in practice - to be a [failure domain](https://en.wikipedia.org/wiki/Failure_domain). Every system should be redundant within the domain, but isolated from other datacenters. Typically Nelson requires the datacenter to be setup with a few key services before it can effectively be used. Imagine the setup with the following components:

<div class="clearing">
  <img src="images/atomic-datacenter.png" />
  <small><em>Figure 4.0: failure domain</em></small>
</div>

Logically, Nelson sits outside one of its target datacenters. This could mean it lives at your office next to your Github, or it might actually reside in one of the target datacenters itself. This is an operational choice that you would make, and provided Nelson has network line of sight to these key services, you can put it wherever you like. With that being said, it is recommended that your stage and runtime docker registries be different, as the performance characteristics of the runtime and staging registries are quite different. Specifically, the runtime registry receives many requests to `pull` images whilst deploying containers to the cluster, whilst the staging registry largely has mass-writes from your build system. Keeping these separate ensures that either side of that interaction does not fail the other.

Whatever you choose, the rest of this operators guide will assume that you have configured one or more logical datacenters. If you don't want to setup a *real* datacenter, [see the authors Github](https://github.com/timperrett/hashpi) about how to setup a [Raspberry PI](https://www.raspberrypi.org/) cluster which can serve as your datacenter.

<h2 id="install-dc-dns" data-subheading-of="operator-guide">Name Resolution</h2>

From an operations perspective, it is important to understand how Nelson is translating its concept of `Unit`, `Stack` and `Version` into runtime DNS naming. Consider these examples:

+ `accounts--3-1-2--RfgqDc3x.service.dc1.yourcompany.com`
+ `profile--6-3-1--Jd5gv2xq.service.massachusetts.yourtld.com`

In these two examples, the [stack name](#user-guide-terminology) has been prefixed to `.service.<datacenter>.<top-level-doamin>`. Whilst the author heartily recommends you [read the consul documentation](https://www.consul.io/docs/guides/forwarding.html) on setting up DNS forwarding, but assuming you have that available and your DNS records do the right delegation you should be able to reference your stacks by a simple DNS convention. The domain that Nelson chooses to use as the TLD for domains is specified by `NELSON_DOMAIN` when [launching](#install-launching) the Nelson container.

Regardless of how ugly the exploded stack names look with their double dashes, this is unfortunately one of the very few schemes that meets the domain name RFC whilst also still enabling users to leverage externally rooted certificate material if needed (the key part here being that your TLD must match the domain you actually own, in order for an externally rooted CA to issue a certificate to you for that domain). As an operator, you could decide that you did not want to use externally rooted material, and perhaps use a scheme like `<stack>.service.<dc>.local`. This would also work, provided the `local` TLD was resolvable for all parties (again, this is not recommended, but would work if you wanted / needed it too).

<h2 id="install-manual-deployment" data-subheading-of="operator-guide">Manual Deployments</h2>

Eventually, the time will come such that you must deploy something outside of Nelson - such as a database - but still need to let Nelson know about it in order to have applications depend on it. This is something that has full support and is easily accomplished with the CLI:

```
$ nelson stacks manual \
  --datacenter california \
  --namespace dev \
  --service-type zookeeper \
  --version 3.4.6 \
  --hash xW3rf6Kq \
  --description "example zookeeper" \
  --port 2181
```

In this example, the `zookeeper` stack was deployed outside of the Nelson-supported scheduler. It is common for such coordination and persistent systems to be deployed outside a dynamic scheduling environment as they are typically very sensitive to network partitions, rebalancing and loss of a cluster member. All that is needed is to simply tell Nelson about the particulars and be aware of the following items when using manual deployments:

+ The system in question is part of the Consul mesh and has implemented [DNS setup as mentioned earlier](#install-dc-dns). This typically means the systems will be running Consul agent, configured to talk to the Consul server in that datacenter / domain.

+ Nelson will never care about the current state of the target system being added as a manual deployment. It is expected the systems outside of Nelson are monitored and managed out of band. If an operator decides to cleanup the stack that Nelson was informed about, the operator **must** be sure to run `nelson stack deprecate <guid>` of the manual deployment so that Nelson will remove it from the currently operational stack list.

+ Any system that was **deployed with Nelson** should never be added as a manual deployment. This is unnecessary as Nelson is already maintaining its internal routing graph dynamically.

For the majority of users, manual deployments will be an operational implementation detail, and users will simply depend on these manual deployments just like any other dependency.

<h2 id="ops-credentials" data-subheading-of="operator-guide">Administering Vault</h2>

In order to provide user applications with credentials at runtime, a member of the operations staff must provision a backend in [Vault](https://www.vaultproject.io/). This can either by one of the provided secret backends which generate credentials dynamically, or a generic backend for an arbitrary credential blob. Nelson is using the following convention regarding how it resolves credential paths, from a `root` of your choosing. The following example assumes `myorg` is the `root`:

```
myorg/<namespace>/<resource>/creds/<unitName>
```

The variable parts of this path mean something to Nelson:

- `<namespace>`: one of `dev`, `qa`, `prod`.
- `<resource>`: the name of a resource, like `mysql`
- `<unitName>`: the name of the unit being deployed, for example `howdy-http`

If a unit does not have access to a resource within a given namespace, the secret will not exist.  All secrets at these paths can be expected to contain fields named `username` and `password`. Extended properties depend on the type of [auth backend](https://www.vaultproject.io/docs/auth/index.html) holding the secret, so as an operations team member you have a responsibility to ensure that the credentials you are providing are needed, and to engage security team advice if you are not certain about access requirements.

The following are examples of **valid** Nelson-compatible `mount` paths:

+ `acmeco/prod/docker-registry`
+ `foobar/dev/s3-yourbucket`
+ `acmeco/prod/accounts-db`

The following are examples of **invalid** `mount` paths:

+ `acmeco/prod/s3-prod-somebucket` - including the namespace name as part of the resource identifier makes it impossible for users to generically write consul-templates against the Vault path.

+ `acmeco/prod/s3-prod-somebucket-california` - in addition to the namespace, including a datacenter specific identifier again makes templating problematic, as it breaks Nelson's deploy target transparency... instead, user a generic path that can have a valid value in all datacenters.

+ `foobar/dev/accounts-db /` - the trailing slash causes issues both in Vault and Nelson

In order to obtain the credentials in your container runtime, it is typically expected that users will leverage [consul-template](https://github.com/hashicorp/consul-template) to render their credentials. Consul Template has built in support for extracting credentials from Vault, so the amount of user-facing integration work is very minimal.

<h1 id="contributing" class="page-header">Contributing</h1>

Contributing to Nelson is simple. If there is something you think needs to be fixed, either open an issue on GitHub or, better yet, just send a pull request with a patch. Typically speaking we are diligent about backward compatibility, both in the API and in the YAML specifications. Nelson can support breaking changes, but in doing so we have to coordinate upgrades to the command line client.

<h1 id="credits" class="page-header">Credits</h1>

Building Nelson was a multi-month effort by the Verizon Labs Infrastructure Engineering team. In addition to the specific engineers called out below, thanks to the other engineering staff internally who provided their useful feedback, advice and tollerance for early-adopter pain.

<h2 id="staff" data-subheading-of="credits">Staff</h2>

The enginering staff who origionally laboured over building Neslon are listed below:

* [Timothy Perrett](https://github.com/timperrett)
* [Stew O'Connor](https://github.com/stew)
* [Greg Flanagan](https://github.com/kaiserpelagic)
* [Ross A. Baker](https://github.com/rossabaker)
* [Cody Allen](https://github.com/ceedubs)
* [Andrew Morhland](https://github.com/andrewmohrland)
* [Alice Wu](https://github.com/berkeleybear)
* [Ryan Delucchi](https://github.com/ryanonsrc)

Finally, a shout out to the following teams at Verizon who helped make this project happen:

* Verizon executive management for believing in us, and affording the time for Nelson to be built
* Verizon DevOps team for handling all the storage and database systems
* Verizon Network Engineering for connecting us

<h2 id="entymology" data-subheading-of="credits">Entymology</h2>

[Admiral Nelson](https://en.wikipedia.org/wiki/Horatio_Nelson,_1st_Viscount_Nelson) was a famous British naval commander who fought off foreign advances during the Napoleonic Wars - most notably at the Battle of Trafalgar, as commander of [HMS Victory](https://en.wikipedia.org/wiki/HMS_Victory), where he defeated the French navy despite being outnumbered and outgunned.