User Workflow: Container to Container Networking

## Assumptions
- You have a CF deployed with silk release
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE

## What

In the HTTP route track of stories you learned about the dataflow for HTTP traffic. But what if appA (an app within CloudFoundry) wants to talk to appB (another app within CloudFoundry)? Before Container to Container networking (c2c) there was no shortcut for intra-CF traffic. Without c2c, if appA wanted to talk to appB, then the traffic had to travel outside of the CF foundation, hit the external load balancer, and go through the normal HTTP traffic flow.

```
Life Before C2C                                  +------+
                                                 |      |
                                                 |      |
          +------------------------------------+ | AppA |
          |                                      |      |
          |                                      |      |
          |                                      +------+
          |
          v                                      +------+
                            +-----------+        |      |
+------------------+        |HTTP Router|        |      |
|HTTP Load Balancer| +----> |(GoRouter) | +----> | AppB |
+------------------+        +-----------+        |      |
                                                 |      |
                                                 +------+

```

So many extra hops for apps within the same CF! Especially if the apps are on the same Diego Cell!

Those hops come with extra latency. It can also come with an extra security risk. If appB  *only* needs to be accessed by appA (for example if appA is the frontend microservice and appB is the backend microservice for appA), appB would still need to be accessible via an HTTP route! This is exposing appB to more attack vectors than it should be.

With container to container networking (c2c) apps within a foundation can talk directly to each other. Now appA can talk to appB without leaving the CF foundation. And appB doesn't need to be accessible via an HTTP route (just an internal one, we'll get to that later in the service discovery track).

```
Life with C2C

+------+        +------+
|      |        |      |
|      |        |      |
| AppA | +----> | AppB |
|      |        |      |
|      |        |      |
+------+        +------+
```

In order for appA to be *able* to talk to appB, it needs to have permission. You will need to create a network policy.

Let's ignore the technical implementation for now and go through the user workflow.

## How

📝**Use Container to Container Networking**

1. Push another [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app named appB. Use the `--no-route` so that no HTTP route is created for appB.
 ```
 cf push appB --no-route
 ```
1. Get the overlay IP for appB `cf ssh appB -c "env | grep CF_INSTANCE_INTERNAL_IP"`
   (An explanation of what "overlay" means awaits in future stories! For now just know that each app instance has a unique overlay IP that c2c uses.)

1. Get onto the container for appA and curl the appB internal IP and app port.
   ```
cf ssh appA
watch  "curl CF_INSTANCE_INTERNAL_IP:8080"
   ```
   You should get a `Connection refused` error because there is no network policy yet.

1.  In another terminal, add a network policy from appA to appB, with protocol tcp, on port 8080.
 ```
 cf add-network-policy appA --destination-app appB --protocol tcp --port 8080
 ```


### Expected Result
After you add the policy, the curl from inside of the appA container to appB should succeed.
If it doesn't work, check that you created the policy in the correct direction, from appA --> appB, not the other way around.

L: c2c
L: user-workflow
---
Overlay vs Underlay

## Assumptions
- None

## What

The **underlay network** is the network that you're given to work with, on top of which you build a virtual **overlay network**. Routing for overlay networks is done at the software layer (in CF we use iptables rules). Overlay networks are used to create layers of abstraction that can be used to run multiple separate layers on top of the physical network — the bottom underlay network. These are general definitions that are not specific to Cloud Foundry.

** Your ** underlay network is often someone else\'s overlay network, that engineer just works on a lower abstraction layer and might work on an IaaS rather than a PaaS, for example. It\'s all relative! 🤯

Routing to an app using the Diego Cell IP and port is done on what we will refer to as the **underlay network**. Container to container networking (c2c) is done on what we will refer to as the **overlay network**.

## Resources
- [Difference between overlay and underlay](https://ipwithease.com/difference-between-underlay-network-and-overlay-network/)

L: c2c
---

Iptables Primer

Please go to the section labeled "iptables-primer" and complete those stories before moving on.

L: c2c
---

Container to Container Networking - Part 0 - Dataflow Overview

## Assumptions
- None :)

## What
In the last story you learned the difference between underlay networks (physical) and overlay networks (software).

Let's see how both the overlay and underlay networks are used when one app talks to another using container to container networking (c2c).

Each step marked with a ✨ will be explained in more detail in its own story.

## How
Follow the steps below on the diagram. (Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).)

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. ✨ The packet exits the app container through the veth interface.
1. ✨ The overlay packet is marked with a ...mark... that is unique to the source app.
1. ✨ Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. ✨ The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. ✨ Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.
Yay!

### Expected Result

You should have a basic overview of the data path for container to container networking...even if you don't understand it all yet.
The next few stories will go through and explain each of the steps marked with a ✨.

## Resources
- [Difference between overlay and underlay](https://ipwithease.com/difference-between-underlay-network-and-overlay-network/)

L: c2c
---

Container to Container Networking - Part 1.1 - Network Namespaces

## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. **The packet exits the app container through the veth interface. <------- CURRENT STORY**
1. The overlay packet is marked with a ...mark... that is unique to the source app.
1. Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.

## What
Each CF app runs in a container, but what *is* a container? A container is a collection of **namespaces** and **cgroups**.

**Namespaces** "partition kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources" (thanks [wiki](https://en.wikipedia.org/wiki/Linux_namespaces)). There are different types of namespaces. For example, the mount namespace lets processes see different file trees. Another example (most related to this onboarding) is the network namespace. The network namespace isolates network interfaces (we'll get into what those are in the next story).

**Cgroups** (pronounced cee-groups) are resource limits. Cgroups let you say: "these processes can only use 1G of memory".

Most important in this onboarding context is the networking namespace. Container networking components are responsible for setting up the networking namespace for each app.

Even the bosh processes are run in containers! In CF this is done with [Bosh Process Manager (BPM)](https://github.com/cloudfoundry/bpm-release).

In this story you'll play around with BPM containers, network namespaces, and even make a network namespace for yourself.

## How

1. Read Julia Evan's post: [What even is a container](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)

📝 **Look at a container**
1. Ssh onto a Diego Cell and become root.
1. Look at the network interfaces (again, we'll go deeper in the next story).
 ```
ifconfig
 ```
1. Inspect the directories that the root user has access to. For example, look at all the log files.
 ```
ls /var/vcap/sys/log
 ```
1. Get into the BPM container for the vxlan-policy-agent.
 ```
bpm list
bpm shell vxlan-policy-agent
 ```
1. Look at the network interfaces. How do they compare to the host vm network interfaces?
1. Look at the log files you can access. How do they compare to the files accessible to root user?

 ❓Based on this information, does BPM create a [mount namespace](https://medium.com/@teddyking/linux-namespaces-850489d3ccf) for the vxlan-policy-agent container?
 ❓Based on this information, does BPM create a network namespace for the vxlan-policy-agent container?
1. Exit the container.

📝 **Make your own network namespace**

1. Still on a Diego Cell as root, create your own network namespace called meow.
 ```
ip netns add meow
```
1. List all of the networking namespaces
 ```
ip netns
 ```
 You should only see meow. Hmmm. You might think you would see the other networking namespaces for all the apps on this cell. (I certainly thought so when I first tried this.) You'll learn how to view an app's networking namespace one day, I promise.

1. Curl google.com from the Diego Cell. See that it works! This is because Application Security Groups allow it. (Remember ASGs?! You might or might not have done the ASG stories yet. tl;dr ASGs are iptables firewall rules for egress traffic.)

1. Curl google.com from inside of your networking namespace
 ```
ip netns exec meow curl google.com
 ```

What? It doesn't work!? You should see `curl: (6) Could not resolve host: google.com`. Try another URL. They will all fail.

### Expected Outcome
The meow networking namespace can't send any traffic out of the container. By default, network namespaces are completely isolated and have no network interfaces.
In the next story you'll explore network interfaces. You'll learn why the meow namespace needs one in order for you to curl google.com.

## Resources
[iptables netns man page](http://man7.org/linux/man-pages/man8/ip-netns.8.html)
[linux network namespaces/veth/route table blog](https://devinpractice.com/2016/09/29/linux-network-namespace/)
[network namespaces blog](https://blogs.igalia.com/dpino/2016/04/10/network-namespaces/)
[interface explanations](https://www.computerhope.com/unix/uifconfi.htm)
[linux namespaces overview](https://medium.com/@teddyking/linux-namespaces-850489d3ccf)

L: c2c
L: questions
---

Container to Container Networking - Part 1.2 - Network Interfaces

 ## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have the meow network namespace created
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and named appA (the fewer apps you have deployed the better)

## Review

This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. **The packet exits the app container through the veth interface. <------- CURRENT STORY**
1. The overlay packet is marked with a ...mark... that is unique to the source app.
1. Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.


## What
In this story you are going to look at network interfaces.

A network interface is ... the interface between two different networks, physical or virtual.  You can list network interfaces with `ifconfig` (old way) or `ip link list` (new way).
In order to have packets from a CF app leave an app container, there needs to be a network interface that can send packets elsewhere. In the CF case, we want them to go to the Diego Cell.
In order to have packets leave the Diego Cell, there needs to be a network interface to the underlay network.

Let's look at the interfaces on the Diego cell and in our meow network namespace.


## How
📝 **Look at network interfaces**

1. List all of the network interfaces in the Diego Cell (this output is edited for brevity and clarity)
  ```
$ ip link
1: lo                 <------------- The loopback network interface that lets the system communicate with itself over localhost.
2: eth0               <------------- A ethernet interface. Traffic goes here to leave the Diego Cell.
1555: silk-vtep       <------------- A VXLAN overlay network interface. Overlay packets go here to be encapsulated in an  underlay packet before exiting the Diego Cell.
1559: s-010255096003@if1558: link-netnsid 0   <-------------  The interface that links to the network namespace with id 0. The name is `s-CONTAINER-ID`. This is the veth interface.
                                                                 There will be one of these network interfaces per app on the Diego Cell.
  ```

1. Now list all of the networking interfaces in the meow networking namespace
  ```
ip netns exec meow ip link
  ```
Nothing! No wonder you can't curl google.com! There is no network interface for packets to travel through.

The solution is to create a veth (virtual ethernet) pair. A veth pair consists of two virtual ethernet interfaces. One is placed in the host network namespace, the other in the meow network namespace. The veth pair acts like a bridge between the network namespace and the host.
You already saw one side of the veth pair for the proxy app when you ran `ip link` inside of the Diego Cell.  Let's look at the other half of the veth pair.

Side note: Why can't each networking namespace connect directly to `eth0`? "One of the consequences of network namespaces is that only one interface could be assigned to a namespace at a time. If the root namespace owns eth0, which provides access to the external world, only programs within the root namespace could reach the Internet."This explanation comes the extra extra credit link about [making your own veth pair](https://blogs.igalia.com/dpino/2016/04/10/network-namespaces/).

🤔 **Look at network interfaces inside the proxy app container**
1. Ssh proxy app.
1. List all of the network interfaces.

### Expected Result
You should see an eth0 interface inside of the proxy app container. This is how traffic exits the app container.

## Extra Credit
Look at the [code](https://github.com/cloudfoundry/silk/blob/master/cni/lib/pair_creator.go) and [tests](https://github.com/cloudfoundry/silk/blob/master/cni/lib/pair_creator_test.go) in silk where veth pairs are set up.

## Extra Extra Credit
📝 **Make your own veth pair **
1. Follow [these instructions](https://blogs.igalia.com/dpino/2016/04/10/network-namespaces/) to create a veth pair to connect the meow network namespace. If successful, you will be able to curl google.com

## Resources
[interface explanations](https://www.computerhope.com/unix/uifconfi.htm)
[linux network namespaces/veth/route table blog](https://devinpractice.com/2016/09/29/linux-network-namespace/)

L: c2c
---

Container to Container Networking - Part 2 - Marks

 ## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. The packet exits the app container through the veth interface.
1. **The overlay packet is marked with a ...mark... that is unique to the source app.  <------- CURRENT STORY**
1. Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.


## What
In the last story we found out that packets leave the app container over the veth pair interface. In this story we are going to look at how overlay packets are marked.

First off, *why* are packets marked?
Packets are marked with a mark that is unique to the source app. Each instance of the source app has its packets marked with the same ID. If there is a c2c policy, then on the destination Diego Cell there are iptables rules that allow traffic with a certain mark to certain destinations. The policies look like the following diagram. Using one mark per source app, decreases the number of iptables rules needed per c2c policy, especially when there are large number of app instances.

![markful networking policy diagram](https://storage.googleapis.com/cf-networking-onboarding-images/diagram-of-silk-networking-policies.png)

If packets weren't marked we *could* use the overlay IP as a unique identifier. However, this would create the need for many more iptables rules, especially when there are a large number of app instances.

![markless networking policy diagram](https://storage.googleapis.com/cf-networking-onboarding-images/diagram-of-flannel-networking-policies.png)

You will learn more about how the c2c policies check the mark in later stories (hint: it uses iptables). For now, let's focus on how traffic is marked (hint: it uses iptables).

Here is an example iptables rule that sets a mark.
![setmark iptables rule example](https://storage.googleapis.com/cf-networking-onboarding-images/set-mark-iptables-rule-example.png)

In this story we are going to find the applicable set-xmark rules for our app and we're going to find out where that mark value comes from.

## How
📝 **Find set-xmark rules**
1. Delete all network policies. This time you are going to use the networking API because former policies from deleted apps can linger in the database, but not show up in the CLI.
 ```
 cf curl /networking/v1/external/policies > /tmp/policies.json
 cf curl -X POST /networking/v1/external/policies/delete -d @/tmp/policies.json

 # check that they are all deleted
 cf curl /networking/v1/external/policies
 ```

1. Ssh onto the Diego Cell where appA is located and become root
1. Look for the iptables rules that set marks.
 ```
iptables -S | grep set-xmark
 ```
 Nothing! This is because there are no policies. A mark is only allocated to an app when that app is used in a container to container (c2c) networking policy.

1. In a different terminal, add a c2c policy from appA to appB  (`cf add-network-policy --help`)
1. Back on the Diego Cell, look again for iptables rules that set marks
 ```
iptables -S | grep set-xmark
 ```

You should see something that looks like the colorful example above. Copy and paste it here.
```
# PASTE THE SET-XMARK RULE HERE
```

The source IP is the overlay IP for your app. The comment is the app guid for appA. And the mark is...well... where *does* the mark come from?

When a c2c policy is created, the policy server determines if the app has a mark already or not. If the app doesn't have a mark yet, it creates one. Let's look at all these marks.
The marks are an internal implementation of how c2c policies work, so they are not exposed on the external API (the API the CLI uses). But there is also an internal policy server API. The internal policy server API is the API that other CF components, like the vxlan-policy-agent, use.

📝 **Look at marks via internal policy server API**

1. You will need certs to call this API, those certs are located on the Diego Cell at `/var/vcap/jobs/vxlan-policy-agent/config/certs`
1. Follow the [docs](https://github.com/cloudfoundry/cf-networking-release/blob/develop/docs/policy-server-internal-api.md) for how to list all of the c2c policies.
You should see something like the following. The tag for appA should match the mark you saw in the iptables rule.
 ```
{
  "total_policies": 1,
  "policies": [
    {
      "source": {
        "id": "90ff1b89-a69d-4c77-b1bd-415ae09833ed",  <------- AppA guid
        "tag": "0004"                                  <------- AppA mark, should match what you saw in the iptables rule above
      },
      "destination": {
        "id": "0babce4f-6739-4fc8-8f74-01f11179bfe5",  <------- AppB guid
        "tag": "0005",                                 <------- AppB mark
        "protocol": "tcp",
        "ports": { "start": 8080, "end": 8080 }
      }
    }
  ]
}
 ```

❓Hey! AppB has a mark too. Why?
❓Marks on packets are limited to 16 bits. How many unique marks is this? Does this give you scaling concerns for c2c networking?

### Expected Outcome
The data about the tag for the source app from the internal policy server API should match the mark in the iptables rule.

## Look at the Code
In the vxlan policy agent (vpa), there is a component called the planner. The planner gets information from the internal policy server API about all of the c2c policies. The planner turns this policy information into proposed iptables rules.

[Here](https://github.com/cloudfoundry/silk-release/blob/develop/src/vxlan-policy-agent/planner/planner_linux.go#L297-L304) the VPA goes through all of the source apps and creates mark rules for them

[Here](https://github.com/cloudfoundry/silk-release/blob/develop/src/lib/rules/rules.go#L100-L105) is the implementation of *NewMarkSetRule*

L: c2c
L: questions
---

Container to Container Networking - Extra Credit - What is mark?

## Assumptions
- You have done the other stories in this track

## What
You may have noticed a discrepancy in the diagram and the steps about _where_ the mark is located. In the diagram it shows the mark on the _underlay_ packet. But the steps say that the _overlay_ packet is marked. Also the iptables rules seem to add the mark to the _overlay_ packet. So which is it? Well, both, kind of.

**The mark for the overlay packet is _not_ part of the packet itself.** This mark is just a bit of metadata about the packet that the kernel keeps track of. This mark exists only as long as it's handled by the Linux kernel. So if a packet is marked before it is sent to a different host, the host will not receive the mark information.

**When VTEP on the source host encapsulates the overlay packet, the mark gets recorded as a header in the unlay packet.** When the VTEP on the destination host decapsulates the underlay packet, it sets the mark on the kernel for the overlay packet.

L: c2c
---

Container to Container Networking - Part 3.1 - Linux Routes Table Primer

## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. The packet exits the app container through the veth interface.
1. **The overlay packet is marked with a ...mark... that is unique to the source app.
1. **Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.   <------- CURRENT STORY**
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.


## What

In order for all overlay packets to be sent to the correct interface, the linux routes table needs to be configured correctly. But, what is a linux routes table?

## How

1. 🎥I couldn't find a good blog post to read. So instead I found a youtube video. Watch [this](https://www.youtube.com/watch?v=g8eP4fhrx3I) video.

### Expected Outcome
You understand the basics of a route table.

L: c2c
---

Container to Container Networking - Part 3.2 - Diego Cell Routes Table

## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. The packet exits the app container through the veth interface.
1. **The overlay packet is marked with a ...mark... that is unique to the source app.
1. **Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.   <------- CURRENT STORY**
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.


## What

In the last story, you were introduced to what a routes table is. In this story we are going to look at the routes table on a Diego Cell and decipher what is there.

There are two easy ways to look at the route table. The old way is `route -n`, which displays the information nicely with headers. The new way is `ip route` which displays the information with no headers to make scripting easier. This story is going to use `route -n` because headers are good.

## How

📝 **Look at routes table**
1. Ssh onto the Diego Cell where appA is running and become root.
1. Look at the routes table
 ```
route -n
 ```

Below is the output from a Diego Cell with two apps running on it. The output is split so we can look at it one section at a time.
The output has been condensed for clarity and brevity.

⬇️This is the default rule that sends traffic to eth0 by default
```
Destination     Gateway         Genmask         Iface
0.0.0.0         10.0.0.1        0.0.0.0         eth0
```

⬇️This is the rule that sends all overlay traffic to the silk-vtep interface. (This is the key bit to this story!)
```
Destination     Gateway         Genmask         Iface
10.255.0.0      0.0.0.0         255.255.0.0     silk-vtep
```

⬇️This is the overlay IP range for the other Diego Cell on the network. (We'll talk more about this in a later story.)
```
Destination     Gateway         Genmask         Iface
10.255.82.0     10.255.82.0     255.255.255.0   silk-vtep
```

⬇️These are istio routers, which are also on the overlay network. If you don't have istio-release deployed, you won't see these.
These are not important for this onboarding.
```
Destination     Gateway         Genmask         Iface
10.255.0.160    10.255.0.160    255.255.255.255 silk-vtep
10.255.0.225    10.255.0.225    255.255.255.255 silk-vtep
```

⬇️These are the apps running on this Diego Cell. The interface is the host side of the veth pair.
```
Destination     Gateway         Genmask         Iface
10.255.77.3     0.0.0.0         255.255.255.255 s-010255077003
10.255.77.4     0.0.0.0         255.255.255.255 s-010255077004
```

### Expected Outcome
You look at the routes table on a Diego Cell and can decipher what you see.

L: c2c
---

Container to Container Networking - Part 4.1 - VLAN VXLAN VTEP V-What?

## Assumptions
- None

## What
In the explanation of the data flow for container to container (c2c) networking, step 4 says: *Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface. *

What does that even mean!??!!?
Let's define some terms.

**LAN** - Local Area Network - a small network that is usually for connecting personal computers. Each computer in a LAN is able to access data and devices anywhere on the LAN. This means that many users can share devices, such as laser printers, as well as data. ([paraphrased from here](https://www.webopedia.com/TERM/L/local_area_network_LAN.html)). Often, used by gamers in [LAN parties](https://en.wikipedia.org/wiki/LAN_party) and by offices.

**VLAN** - Virtual Local Area Network - a network that *appears* to be on the same LAN, even though the machines are physically separated. For example, the Pivotal LA and SF (and probably other west coast offices) are on the same VLAN. This allows Pivots in these offices to SSH onto other machines in the VLAN. But those outside of the VLAN cannot SSH onto machines inside of the VLAN.

**VXLAN** - Virtual eXtensible Local Area Network - VXLAN was developed to provide the same network services as VLAN does, but with greater extensibility and flexibility. Container to container (c2c) networking uses VXLAN to create the overlay network.

**VTEP**- VXLAN EndPoints - are VXLAN tunnels that encapsulate outgoing overlay traffic into underlay packets and decapsulate incoming underlay packets into overlay packets. The VTEP that c2c uses is called silk-vtep.


## Resources
[LAN](https://www.webopedia.com/TERM/L/local_area_network_LAN.html)
[VLAN wiki](https://en.wikipedia.org/wiki/Virtual_LAN)
[VLAN vs VXLAN](http://www.fiber-optic-transceiver-module.com/vxlan-vs-vlan-which-is-best-fit-for-cloud.html)


L: c2c
---

Container to Container Networking - Part 4.2 - Overlay Leases and ARP

 ## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. The packet exits the app container through the veth interface.
1. The overlay packet is marked with a ...mark... that is unique to the source app.
1. Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. **The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to the underlay IP of the Diego Cell where the destination app is located (appA in this case).    <------- CURRENT STORY**
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1. Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.
1. If traffic is allowed, the overlay network directs the traffic to the correct place.


## What

In the last story we learned that the VTEP is responsible for encapsulating the overlay packet and sending it to the correct Diego Cell. But how does it get that information?

Each CF Deployment is given a [range of IPs](https://github.com/cloudfoundry/silk-release/blob/develop/jobs/silk-controller/spec#L30-L33) to act as the overlay IPs. By default the overlay range is `10.255.0.0/16`. This is CIDR (pronounced, cider) notation for the IPs that range from 10.255.0.0 to 10.255.255.255. Every Diego Cell is given a subset of this range to give to the apps.

The Silk Controller is a database backed process that keeps track of which subnet of IPs is leased out to which Diego Cell. Every Diego Cell has a process running called the Silk Daemon, which constantly polls the Silk Controller and asks: "what overlay ranges map to what Diego Cells"? The Silk Daemon then writes that information to the Address Resolution Protocol (ARP) table.

ARP is at layer 2 in the OSI (Open System Interconnection) model. If you haven't heard of OSI, read [this](https://www.webopedia.com/quick_ref/OSI_Layers.asp) as a primer. The main bit, is that at layer 2 [MAC addresses](https://whatismyipaddress.com/mac-address) are used to address data. Because of this, in the ARP table and in the silk database, the Diego Cells are referenced by their MAC addresses and not their underlay IPs. If you aren't familiar with OSI or MAC addresses, read those links before continuing on.

In this story, you are going to look at the Silk Controller database to see what overlay subnet is leased to which Diego Cells. Then you'll going to look at the ARP table.


## How

🤔 **Look at silk controller database**

1. Get the mysql username and password for the silk controller db. Download your bosh manifest and see what is set for the properties: `silk-controller.properties.database.name` and `silk-controller.properties.database.password`. You might need to look up these values in credhub. This will differ based on your deployment.
1. Bosh ssh onto the database VM and become root.
1. Login to your database. For a pxc-mysql deployment, it looks like this
 ```
/var/vcap/packages/pxc/bin/mysql -u network_connectivity -p -h sql-db.service.cf.internal -D DATABASE_NAME
 ```
1. Run the following query to look at all of the subnets.
 ```
mysql> select * from subnets;
+----+---------------+---------------------------+-----------------------------+------------------+
| id | underlay_ip   | overlay_subnet            | overlay_hwaddr              | last_renewed_at  |
+----+---------------+---------------------------+-----------------------------+------------------+
|  0 | DIEGO_CELL_IP | DIEGO_CELL_OVERLAY_SUBNET | DIEGO_CELL_VTEP_MAC_ADDRESS | TIMESTAMP        |
|  1 | 10.0.1.12     | 10.255.77.0/24            | ee:ee:0a:ff:4d:00           | 1555520514       | <--- Let's call these values: DIEGO_CELL_0_IP, DIEGO_CELL_0_OVERLAY_SUBNET, DIEGO_CELL_0_VTEP_MAC_ADDRESS
|  2 | 10.0.1.13     | 10.255.82.0/24            | ee:ee:0a:ff:52:00           | 1555520512       | <--- Let's call these values: DIEGO_CELL_1_IP, DIEGO_CELL_1_OVERLAY_SUBNET, DIEGO_CELL_1_VTEP_MAC_ADDRESS
|  3 | 10.0.1.17     | 10.255.0.225/32           | ee:ee:0a:ff:00:e1           | 1555520513       | <--- This is an istio router, which is also on the overlay. Each istio router gets one overlay IP. You might or might not have these.
|  4 | 10.0.1.18     | 10.255.0.160/32           | ee:ee:0a:ff:00:a0           | 1555520515       | <--- Another istio router.
+----+---------------+---------------------------+-----------------------------+------------------+
```

📝**Look at the arp table**

1. Ssh onto the Diego Cell with DIEGO_CELL_0_IP as the underlay IP and become root.
1. Find the mac address of the silk-vtep interface
 ```
ip link show silk-vtep <---- Should match DIEGO_CELL_0_VTEP_MAC_ADDRESS
 ```

1. Look at the ARP table. This is how the VXLAN VTEP knows where each Diego Cell is located.
 ```
 arp
 ```
 You should see something like this. The output is split up so we can look at it section by section. Yours might be in a different order.


⬇️This is the entry for the other Diego Cell. This should match DIEGO_CELL_1_OVERLAY_SUBNET and DIEGO_CELL_1_VTEP_MAC_ADDRESS.
 ```
Address                  HWtype  HWaddress           Flags Mask            Iface
10.255.82.0              ether   ee:ee:0a:ff:52:00   CM                    silk-vtep
 ```

⬇️This is the entry for the Istio Routes. You might or might not have these. The MAC address and overlay ips should match the database.
 ```
Address                  HWtype  HWaddress           Flags Mask            Iface
10.255.0.225             ether   ee:ee:0a:ff:00:e1   CM                    silk-vtep
10.255.0.160             ether   ee:ee:0a:ff:00:a0   CM                    silk-vtep
 ```

⬇️This is the entries for the 2 apps running on this cell. The addresses match the overlay IPs of the two apps.
 ```
Address                  HWtype  HWaddress           Flags Mask            Iface
10.255.77.3              ether   ee:ee:0a:ff:4d:03   CM                    s-010255077003
10.255.77.4              ether   ee:ee:0a:ff:4d:04   CM                    s-010255077004
 ```

⬇️This is the entry for the eth0 interface.
 ```
Address                  HWtype  HWaddress           Flags Mask            Iface
10.0.0.1                 ether   42:01:0a:00:00:01   C                     eth0
 ```

## Resources
[The Layers of Networking - OSI](https://www.webopedia.com/quick_ref/OSI_Layers.asp)

L: c2c
---
Container to Container Networking - Extra Credit - How are only some packets decapsulated?

## Assumptions
- You have completed the previous stories in this track
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA on Diego Cell 1
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appB on Diego Cell 2

## What

So far the overlay packet has been encapsulated into an underlay packet and then it is sent to a second Diego Cell. Once the underlay packet gets to the Diego Cell it is gets decapsulated by the VTEP. But how does "it" know to send these specific underlay packets to the silk-vtep interface to be decapsulated and not other packets?

Let's figure it out by inspecting the packets with tcpdump! Tcpdump is a CLI tool that allows you to inspect all of the traffic flowing through your container.

### How

🤔 **Send traffic via the overlay from appA to appB**
1. In terminal 1, use watch to continuously curl appB from appA using appB's overlay IP and app port.

📝 **Look at the underlay traffic**
1. In terminal 2, ssh onto Diego Cell 2, where appB is running.
1. The underlay packet is from Diego Cell 1 to Diego Cell 2, so use tcpdump to look at all traffic from Diego Cell 1.
 ```
tcpdump -n src DIEGO_CELL_1_IP
 ```
❓What do you notice about all of the traffic? What do they have in common? Based on this information how do you think only this traffic is being decapsulated?
❓What protocol is this traffic using? Is that surprising to you?

### Expected outcome
You should see that all traffic to be decapsulated is sent to the same port. This is how some traffic is decapsulated by the VTEP but not others.

You should also notice that all of the traffic is sent via UDP. WHAT? Read [here](https://blog.ipspace.net/2012/01/vxlan-runs-over-udp-does-it-matter.html) for more details on _that_.

L: c2c
L: questions
---

Container to Container Networking - Part 5.1 - Enforce Policy

## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. The packet exits the app container through the veth interface.
1. The overlay packet is marked with a ...mark... that is unique to the source app.
1. Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to the underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1.  **Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.   <------- CURRENT STORY**
1. If traffic is allowed, the overlay network directs the traffic to the correct place.


## What

In the last story you learned how the VTEP is able to encapsulate overlay packets and send them to the correct Diego Cell via the underlay network.

Now that the packet has arrived at the correct Diego Cell, how is c2c policy enforced? (hint: iptables rules)

In this story, you are going to look at the iptables rules on the destination Diego Cell


## How
🤔 **Setup**
1. Make sure there are no c2c policies. (`cf network-policies --help`)

📝**Look at iptables rules**
1. Ssh onto the Digeo Cell where appB is running and become root.
1. Look at the iptables rules on the filter table
 ```
iptables -S
 ```
You should see a chain that looks like the following
 ```
-N vpa--1555607784864807      <----------- Let's call this VPA_CHAIN_NAME
 ```
VPA stands for the VXLAN Policy Agent. The VPA is a component from silk-release that is responsible for translating the c2c policies in the database into iptables rules on the Diego Cell.
Any rules related to enforcing c2c policies will be on a custom chain that starts with "vpa".

1. Look at the rules on the vpa chain.
 ```
iptables -S VPA_CHAIN_NAME
 ```
 There's nothing there. No rules attached. That is because there are no c2c policies.

🤔 **Create c2c policy and look at iptables rules**
1. Create a c2c network policy from appA to appB (`cf add-network-policy --help`). The diagram in the review section shows apps on two different Diego Cell. The same iptables rules will be created and enforced regardless of whether appA and appB are on the same Diego Cell or not. A future story will go into more detail about this. But for this story, just know that it doesn't matter what Diego Cell your apps are running on.
1. On the Diego Cell, look at the rules on the vpa chain again.
 ```
iptables -S VPA_CHAIN_NAME
 ```
 What! you should get the error.`iptables: No chain/target/match by that name.`.
 When the VPA sees that it needs to update the iptables rules on a Diego Cell, it *doesn't* append new rules to existing chains. Instead, it replaces ALL OF THE RULES. When the VPA replaces all of the rules, the VPA chain get's renamed with a new timestamp.
1. Find the new name of the VPA chain.
1. Look at the rules on that chain.  They should look something like this
```
-N vpa--1555608460726256
-A vpa--1555608460726256 -s 10.255.77.3/32 -m comment --comment "src:2f978b7f-b3d2-4f50-b08f-776634a6e411" -j MARK --set-xmark 0x2/0xffffffff
-A vpa--1555608460726256 -d 10.255.77.4/32 -p tcp -m tcp --dport 8080 -m mark --mark 0x2 -m conntrack --ctstate INVALID,NEW,UNTRACKED -j LOG --log-prefix "OK_0002_c7de6123-d906-4c65-9 "
-A vpa--1555608460726256 -d 10.255.77.4/32 -p tcp -m tcp --dport 8080 -m mark --mark 0x2 -m comment --comment "src:2f978b7f-b3d2-4f50-b08f-776634a6e411_dst:c7de6123-d906-4c65-9717-c5d040568c76" -j ACCEPT
```


Let's look at them line by line.

⬇️This is adding the VPA chain
```
-N vpa--1555608460726256
```

⬇️This is about marking outgoing traffic. (We went over this in the story *Container to Container Networking - Part 2 - Marks*). You might not have this rule. You will only have this rule if this app is also the source of a networking policy.
```
-A vpa--1555608460726256 -s 10.255.77.3/32 -m comment --comment "src:2f978b7f-b3d2-4f50-b08f-776634a6e411" -j MARK --set-xmark 0x2/0xffffffff
```

⬇️This is about logging when traffic hits this rule. Your deployment might have logging turned off, so this rule might or might not be present. A future story will go more into this.
```
-A vpa--1555608460726256 -d 10.255.77.4/32 -p tcp -m tcp --dport 8080 -m mark --mark 0x2 -m conntrack --ctstate INVALID,NEW,UNTRACKED -j LOG --log-prefix "OK_0002_c7de6123-d906-4c65-9 "
```

⬇️THIS IS THE RULES THAT ENFORCES C2C POLICY!!!!
```
-A vpa--1555608460726256 -d 10.255.77.4/32 -p tcp -m tcp --dport 8080 -m mark --mark 0x2 -m comment --comment "src:2f978b7f-b3d2-4f50-b08f-776634a6e411_dst:c7de6123-d906-4c65-9717-c5d040568c76" -j ACCEPT
```
❓Can you rewrite this rule in sentence form to explain what it is checking?


### Expected Results
 You can find and understand the iptables rule that enforces c2c policy.

## Resources
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)
L: c2c
L: questions
---

Container to Container Networking - Part 5.2 - Enforce Policy - Same Cell vs Different Cell

## Assumptions
- You have an OS CF deployed
- You have done the other stories in this track
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)

## Review
This track of stories is going to go through the steps (listed below) that were covered in the dataflow overview.
The steps and diagram will be at the top of each story in case you need to orient yourself. Higher quality diagram [here](https://storage.googleapis.com/cf-networking-onboarding-images/c2c-data-plane.png).

![c2c traffic flow](https://storage.googleapis.com/cf-networking-onboarding-images/overlay-underlay-silk-network.png)

1. AppB (on Diego Cell 1) makes a request to AppA's overlay IP address (on Diego Cell 2). This packet is called the overlay packet (aka the c2c packet).
1. The packet exits the app container through the veth interface.
1. The overlay packet is marked with a ...mark... that is unique to the source app.
1. Because the packet is an overlay packet, it is sent to the silk-vtep interface on the Diego Cell. This interface is a VXLAN interface.
1. The overlay packet is encapsulated inside of an underlay packet. This underlay packet is addressed to the underlay IP of the Diego Cell where the destination app is located (appA in this case).
1. The underlay packet exits the cell.
1. The packet then travels over the physical underlay network to the correct Diego Cell.
1. The packet arrives to the correct Diego Cell
1. The underlay packet is decapsulated. Now it's just the overlay packet again.
1.  **Iptables rules check that appA is allowed to talk to appB based on the mark on the overlay packet.   <------- CURRENT STORY**
1. If traffic is allowed, the overlay network directs the traffic to the correct place.

## What

The diagram above shows the container to container networking traffic flow between two apps on different Diego Cells. But what about when the apps are on the same Diego Cell?

In this story, you are going to do some investigation to figure out what the container to container traffic flow is for apps on the same Diego Cell.

## Questions

To answer these questions, it might be helpful to review [Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p).

❓What chain (INPUT, OUTPUT, or FORWARD) are the container networking policy iptables rules appended to?
❓Does overlay traffic between apps on different Diego Cells hit this chain?
❓Does overlay traffic between apps on the same Diego Cell hit this chain?
❓Does overlay traffic between apps on the same Diego Cell get encapsulated?
❓What steps listed in the review section apply for apps on the same Diego Cell?


## Expected Result

At the end of this story, you should know the traffic flow between two apps on the same Diego Cell.

## Resources
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)

L: c2c
L: questions
---

Logging with Container to Container Networking

## Assumptions
- You have an OS CF deployed
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB (the fewer apps you have deployed the better)
- There are no c2c policies

## What

When you use c2c policies you have the option of logging every time that one app attempts to send traffic to another app.

In this story you will learn (1) how to turn on this feature, (2) how to look at the logs, and (3) how this feature is implemented (hint: iptables).

## How

🤔 **Turn on logging**
1. Follow [these instructions](https://github.com/cloudfoundry/silk-release/blob/77ecbb775780d74c5a8b6e87c5554dab375a9235/docs/traffic_logging.md#traffic-logging) to enable c2c policy logging _AND_ ASG logging. (There is a bug where you need to enable both in order for c2c logging to work. Maybe you could be the one who fixes it?)


📝 **Look at logs**
1. In one terminal, Ssh onto the Diego Cell where appB is running and become root.
1. Watch the kern.logs (kern stands for kernel, as in the linux kernel).
 ```
tail -f kern.log
 ```
1. In another terminal, curl appB from appA. You should see log line similar to the one below.


 ```
2019-04-18T18:01:05.306552+00:00 localhost kernel: [22246.987902]
DENY_C2C_f13ffea0-9d0d-4ee9-                   <----- DENY means that the traffic was blocked by iptables rules. The GUID here is the beginning of the instance GUID that we have seen before.
IN=s-010255077003                              <----- The interface of the source app (This seems backwards given that this is IN, I'm not sure why this is).
OUT=s-010255077004                             <----- The interface of the destination app (This seems backwards given that this is OUT, I'm not sure why this is).
MAC=aa:aa:0a:ff:4d:03:ee:ee:0a:ff:4d:03:08:00  <----- This is a combination of the mac address of the source app's container network interface and the mac address of the source Diego Cell's VTEP.
SRC=10.255.77.3                                <----- The overlay IP of the source app
DST=10.255.77.4                                <----- The overlay IP of the destination app
LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=12504 DF PROTO=TCP SPT=39304 DPT=8080 WINDOW=27400 RES=0x00 SYN URGP=0
 ```


1. Add network policy from appA to appB (`cf add-network-policy --help`)
1. Curl appB from appA again. You should see log line similar to the one below (it's very similar to the one above).
 ```
2019-04-18T18:21:00.670494+00:00 localhost kernel: [23442.243710]
OK_0002_c7de6123-d906-4c65-9               <----- OK means that the traffic was allowed by iptables rules.
IN=s-010255077003
OUT=s-010255077004
MAC=aa:aa:0a:ff:4d:03:ee:ee:0a:ff:4d:03:08:00
SRC=10.255.77.3
DST=10.255.77.4
LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=35333 DF PROTO=TCP SPT=41330 DPT=8080 WINDOW=27400 RES=0x00 SYN URGP=0
MARK=0x2                    <----- Successful logs also include the mark of the source app.
 ```

🤔 **Find the implementation**
1. Look at the iptables rules on the Diego Cell
1. Find the rule that logs c2c traffic.

### Expected Result

You (1) turned on this feature, (2) looked at the logs, and (3) saw how this feature is implemented with iptables rules.

## Resources
[Traffic Logging with Silk Release](https://github.com/cloudfoundry/silk-release/blob/77ecbb775780d74c5a8b6e87c5554dab375a9235/docs/traffic_logging.md#traffic-logging)

L: c2c
---

Watch your c2c packets with tcpdump

## Assumptions
- You have a CF deployed
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and called appA and appB
- There are no c2c network policies

### What?
Sometimes a user comes to us and says "container to container networking is broken! AppA can't talk to AppB". After making sure that they have c2c policies set, the next thing you might do is use tcpdump.

Tcpdump is a CLI tool that allows you to inspect all of the traffic flowing through your container.

In this story we are going to look at the packets being sent from AppA to AppB. Then we'll watch the packets being sent in response!


### How?

📝 **Curl appB from appA**
1. Get the overlay IPs of appA and appB
1. Continually try to curl appB from appA
 ```
watch -n 15 curl APP_A_ROUTE/proxy/APP_B_OVERLAY_IP:8080
 ```

📝 **Look at those packets**
1. In another terminal, ssh onto the Diego Cell where appA is running and become root
1. Run `tcpdump`.
    Ahhhhh too much information! ctrl+c! ctrl+c!  On a Diego Cell there are many packets being sent around, and tcpdump gives information about ALL OF THEM. We need to figure out a way to filter this overwhelming stream of information.
1.  Filter by packets where the source IP is APP_A_OVERLAY_IP and where the destination IP is APP_B_OVERLAY_IP.
 ```
tcpdump -n src APP_A_OVERLAY_IP and dst APP_B_OVERLAY_IP
 ```

 You should see something like:
 ```
$ tcpdump -n src 10.255.77.3 and dst 10.255.77.4
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
 ```

 ...and nothing else. Where are those packets?
 Notice that tcpdump is looking for packets listening on the eth0 interface. That's not where overlay packets go!

1. Look for packets on any interface
 ```
tcpdump -n src APP_A_OVERLAY_IP and dst APP_B_OVERLAY_IP -i any
 ```
 Hey! Those are packets!

 Record the packets you see here from one curl.
 ```
# PUT TCPDUMP OUTPUT HERE
 ```

 If appB was successfully responding, then you should also see packets being sent in the opposite direction.

1. See that no packets are being sent from AppB to AppA
  ```
tcpdump -n src APP_B_OVERLAY_IP and dst APP_A_OVERLAY_IP -i any
 ```

🤔 **Add c2c policy**
1. Add c2c policy to allow traffic from appA to appB (`cf add-network-policy --help`)
1. Continually try to curl appB from appA


📝 **Look at those packets**
1. Look for packets from appA to appB
 ```
tcpdump -n src APP_A_OVERLAY_IP and dst APP_B_OVERLAY_IP -i any
 ```
 Record the packets you see here from one curl.

 ```
 PUT TCPDUMP OUTPUT HERE
 ```
❓How are these packets different from before?

1. Look for packets from appB to appA
 ```
tcpdump -n src APP_B_OVERLAY_IP and dst APP_A_OVERLAY_IP -i any
 ```

### Expected Result

You should see packets being sent in response from appB to appA. You should see `200 OK`.

### Resources
[tcpdump man page](https://www.tcpdump.org/manpages/tcpdump.1.html)
[helpful common tcpdump commands](https://www.rationallyparanoid.com/articles/tcpdump.html)
[debugging non-c2c traffic in CF](https://github.com/cloudfoundry/cf-networking-release/blob/develop/docs/troubleshooting.md#debugging-non-c2c-packets)

L: c2c
L: questions
---
Bypass container networking policies

## Assumptions
- You have a OSS CF deployed
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed and named appA and appB
- You have at least 2 Diego Cells
- There are no c2c network policies

## What

Before container networking and the overlay network, apps could talk to each other via the backend IP and port.

The old workflow was:
1. Create an ASG that allows apps to send traffic to the Diego Cell IPs
1. Bind that ASG to appA
1. AppA discovers appB's backend IP and port.
1. AppA sends traffic to appB.

Turns out that old workflow still (kind of) works. Even worse, some CF users still rely on this behavior! Fun!

In this story you are going to see when this workflow does and doesn't work and why.

## How
🤔 **Get Diego Cell IPs**
1. Get and record the following variables:
 ```
DIEGO_CELL_1_IP = ...
DIEGO_CELL_2_IP = ...
 ```

🤔 **Create a wide open ASG**
1. Create an ASG that allows traffic to DIEGO_CELL_1_IP and DIEGO_CELL_2_IP
1. Bind that ASG to appA.
1. Restart appA

🤔 **Setup**
1. Scale appB to 2 instances.
1. Make sure that you have the following configuration.
 - Diego Cell 1: 1 instance of appA, 1 instance of appB (appB_1)
 - Diego Cell 2: 1 instance of appB, this instance will be called appB_2

🤔 **Get environment variables**
1. Get and record the following variables:
 ```
APP_B_1_BACKEND_PORT = ...
APP_B_2_BACKEND_PORT = ...
 ```
hint: you can ssh onto a specific instance of an app, by passing the `-i` flag (`cf ssh --help`).

📝 **Bypass c2c rules and route integrity**
1. Ssh onto appA
1. See if you can access appB
 ```
curl DIEGO_CELL_1_IP:APP_B_1_BACKEND_PORT
 ```

1. See if you can access appB
 ```
curl DIEGO_CELL_2_IP:APP_B_2_BACKEND_PORT
 ```

### Expected Result
AppA should not be able to access appB_1. AppA should be able to access appB_2.

## Questions

❓Why can appA access apps on other Diego Cells, but not on its own? (Hint: look at the iptables rules and review the diagrams in _Route Propagation - Part 5 - DNAT Rules_)
❓Do you think this is a security concern?
❓Do you think we should "fix" this? What would you suggest?


## Resources
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)

L: c2c
L: questions
L: questions
---

[RELEASE] Container to Container Networking ⇧
L: c2c