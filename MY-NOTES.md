## _Building Kubernetes Operators from scratch_

---

### Kubernetes Operators

----

##### _Definition_


> An Operator is an application-specific `controller` that extends the `Kubernetes API`
> to create, configure and manage instances of (complex) applications on behalf of a Kubernetes user.
---
>_Operator_ `=` _Customer Resource Definition_ `+` _Custom controller_

All Operators are Controllers, but not every Controller is an Operator.

`Operators` have knowledge about the Application they are managing and take actions 
based on the status of the `Custom Resources` that define that Application.

There are 3 main frameworks for building Operators.

1) `RedHat Operator Framework`

- Operator SDK released in 2018
- Operator Lifecycle Manager
- Operator Registry
- Language: Golang
- You can also use Helm Charts or Ansible Playbooks

2) `Kubebuilder`
- Language: Golang

3) `Kopf`
- Zalando Open Source Project
- Language: Python

---

#### Brief Intro to K8S Architecture

`Kubernetes` can be seen from an application perspective as an `ApiServer`,
that has a bunch of control loops that interacts with the ApiServer or through the ApiServer.

These loops do _not_ talk to each other, neither they know the others exists. 
All communications are centrally managed by the ApiServer with each control loop.

There are 2 ways of communications:

- _Commands_
- _Events_

For example when you execute the CLI command `kubectl crete -f my_replicaset.yaml`,
you are making a `POST` request to the ApiServer, where you specify the number of replicas
for the replica set that should be used.

Then you have a `ReplicaSet Controller` which is continuously watching for events that changes the state 
of the ReplicaSet, which is the `Resource` it is interested in and for which it receives the `state-change` events.

##### _Commands_

- Requests (intent) to do something
- Named in the imperative, e.g. 'CREATE'
- Can be rejected
- Higher coupling between sender and receiver 
- Typically used in synchronous 1-1 request/response communication in the form of REST Client and Server

##### _Events_

- Facts, things that happened
- Named in past tense, e.g. 'CREATED'
- Cannot be (semantically) rejected, could be ignored
- Lowest coupling between sender and receiver
- Asynchronous 1-many communication, e.g. Publish/Subscribe

----

##### Orchestration Vs Choreography

Albeit Kubernetes is called a container `Orchastrator`, technically this is not true if we look at it 
from a Software Architecture standpoint. Indeed, the way that Kubernetes works is much more similar to an `Event Choreography`
you can easily find in a typical EDA-based system.

When you issue the command to create a new ReplicaSet, a new event is fired and consumed by the ReplicaSet Controller.
Then the ReplicaSet Controller processed the Event and send a request with a set of Commands back to the APIServer containing the CREATE POD command.

The APIServer will receive this request and fire a POD CREATED Event, which is consumed by the Scheduler.

The Scheduler will _bind_ a Pod to a Node in the cluster if able to do so when processing the event.
Then it will send a request with commands to the APIServer with the BIND POD Command.

The APIServer fire a POD BOUND Event, which will be consumed only by the Kubelet running on that Node,
which is the Kubelet interested in that kind of event.

Kubelet, based on the state-changes for the Pod, will make a request to the ApiServer updating the STATE of the Pod.
Then, whoever wants to look up the state of the Pod will now get an updated information.

This is an implementation of the `Choreography` pattern.

----

#### Act, Observe, Analyze

An `Action` is requested, then an `Event` is emitted. 
The Event is consumed by a process, and it's compared the `Current State` to the `Desired State`.
Based on the difference between the two, a subsequent action is trigger by a Command.

Usually you have that a Controller watch events and so manage state for 1 Root Object.
The term Root Object is used because usually in Kubernetes an Object is made of several 
objects nested inside it.

If you go to the kubernetes/pkg/controller (pck directory in Golang projects is used for 
sharable code libraries intended ot be used by others outside the scope of that repository),
you will find pretty all the basic Kubernetes Objects subpackages, like daemonSet, Deployment, etc...

With each of this Objects you will find a Controller whose job is to manage the state for that
kind of Root Object. You can say the Controller owns the Object.

----

#### Developing Controllers

Kubernetes comes with several features to make a life of a controller dev easier:

- Scheduling and Supervision (self-healing*)
- Configuration and Secret Management
- Service Discovery and Networking
- Storage Management
- Cloud Portability
- Declarative API Stability and Extensibility (CRDs)
- AuthN and AuthZ (RBAC)
- SDKs
- etc, etc, etc ...

However, depending on the complexity of your controller, there might be a steep learning curve:

- Lots of primitives and objects to learn
- client-go (de facto SDK) "is not for mere mortals" (Bryan Liles)
- Optimistic concurrency in an asynchronous eventually consistent system
- There is No Now
- The (global) state is always behind you (distributed, delayed and unknown to local observer)
- Fast moving project

Some of the Principles you should be aware of when writing a Controller:

> 1) Controllers are Autonomous Processes
- Single Responsibility
- Decoupling via event-driven communication
- No central coordinator

> 2) Concurrency and Asynchronicity
- Eventually Consistent by design
- Don't rely on order

> 3) Stateless over Stateful
- APIServer (etcd) is the source of truth
- In-memory caching via reconciliation. Keep the required information 
in memory cache without relying on of persistent disk storage.

> 4) Defensive Programming
- Things will go wrong (crashes will happen). No Atomicity.
- No shared (wall) clock. Your controller state is not global and your time neither
- Anticipate effect on the rest of the system

> 5) Side Effects
- Delivery and processing guarantees only within Kubernetes
