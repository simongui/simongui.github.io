---
layout: post
title: Fluent style testing in Go
draft: true
---
Recently my team has been working on Kubernetes eviction policies for [Navops Command](https://www.univa.com/products/navops.php). Navops Command offers advanced scheduling, multi-tenancy and advanced policy management for Kubernetes. We are excited to bring these new eviction policies to the Kubernetes world and I'm delighted to write a bit about how we tested them along the way.

Briefly there's some other exciting news you might be interested in as well. Recently Univa in partnership with Amazon AWS demonstrated [extreme acale automation by deploying more than one million cores in a single Univa Grid Engine Cluster using AWS](https://blogs.univa.com/2018/06/univa-demonstrates-extreme-scale-automation-by-deploying-more-than-one-million-cores-in-a-single-univa-grid-engine-cluster-using-aws/).

Containerization is popular these days and at Univa we have a lot experience with large scale containerized clusters that dates back before solutions like Kubernetes existed. We are excited to bring our experience to the Kubernetes community with Navops Command. Just like Kubernetes, Navops Command is written in Go and we needed a few improvements to make the process of development, testing and verifying safety guarantees to be more efficient so that engineers could iterate quickly and test clustered scenarios thoroughly.

We are always trying to improve how we test our solutions. The challenge with testing clustered features is that not everyone has the ability to spin up 1,000 or 1 million node clusters and even if you did, running tests on them takes a lot of time and these environments aren't efficient for iterating quickly during development. Writing tests that use simulated clusters isn't a new concept but it's one that if your code is designed from the ground up to be easily simulated it becomes much easier. 

A few key improvements we put effort into that really improved our simulated cluster tests that I would like to talk about are the following.
1. Using a fluent style to make tests really easy to read and write.
1. Simulate and verify cluster convergence.
1. Simulate and verify cluster fault tolerance.

# Fluent style tests
To give a little bit of insight on our development process, we document what use cases our features need to satisfy and we write tests to verify them. Typically use cases are easier to read in document form but not easy to read in test form because of all the setup boiler plate. Even with traditional "helper" functions you may be faimilar with, tests are usually hard to read. We wanted our simulated tests to be as easy to read as our documented use cases. Over time we've started to use these fluent style tests both in those simulated clusters but also in automated acceptance tests running against live Kubernetes clusters on GKE (Google Kubernetes Engine).

Simulated clusters of nodes and Kubernetes pods have a lot of structs and attributes involved and our goal was to hide all of that complexity. A traditional style helper you might be familiar with might look something like the following.
```
addNode("node1", foo, bar, 100, something)
```
_Figure 1. Example of a common test helper function._

No doubt `addNode()` in _figure 1_ is probably hiding a lot of complexity and makes configuring the test a lot shorter but it's not really that readable overall. It isn't obvious what the parameters mean. It takes a lot of digging to understand what the cluster state is.

We wanted to improve this experience and applied a fluent style to our test helpers that configure our simulations. If you're not familiar with fluent API's, it's a technique to create a DSL that reads clearly in english so that the code is self describing. 

The following is a basic example of one of our simple tests.

```go
// TestPackPolicySelectsCorrectPodsScenarioA verifies  
// that the following scenario succeeds.
//
// Scenario A
// Node1: Available slots: 6 running pods 1
// Node2: Available slots: 6 running pods 2
// Node3: Available slots: 6 running pods 3
//
// RESULT: Node1 and Node2 should get pods evicted to eventually run on Node3.
func TestPackPolicySelectsCorrectPodsScenarioA(t *testing.T) {
    cluster := NewClusterBuilder()
    cluster.
        WithNode("node1").AsSchedulable().WithAllocatablePods(6).Build().
        WithNode("node2").AsSchedulable().WithAllocatablePods(6).Build().
        WithNode("node3").AsSchedulable().WithAllocatablePods(6).Build().
        WithPodOnNode("pod1", "node1").AsRunning().Build().
        WithPodOnNode("pod2", "node2").AsRunning().Build().
        WithPodOnNode("pod3", "node2").AsRunning().Build().
        WithPodOnNode("pod4", "node3").AsRunning().Build().
        WithPodOnNode("pod5", "node3").AsRunning().Build().
        WithPodOnNode("pod6", "node3").AsRunning().Build()

    cluster.WithPodPlacementStrategy("pack")

    // Simulate the running of the pack policy.
    podsToEvict := cluster.Run()

    // Verify that the correct 3 pods are selected to be evicted.
    assert.Equal(t, 3, len(podsToEvict)
    assert.Equal(t, "pod1", podsToEvict[0].ID())
    assert.Equal(t, "pod2", podsToEvict[1].ID())
    assert.Equal(t, "pod3", podsToEvict[2].ID())
}
```
_Figure 2. Example of a simulated test in Navops Command._

The helper functions are chained in an english sentence form. Go isn't the easiest language to implement fluent API's with but with a few things in place you can create a fluent style that allows you to add any number of parameters that you can chain together. 

There's 2 key improvements a fluent API gives us over  helper functions.  
1. Self describing, easy to read and chained in a sentence.
1. Unused optional parameters don't clutter the test.  
```
addNode("node1", true, nil, nil, nil, "", nil, "", nil, nil, nil)
addNode("node1", false, nil, nil, nil, "", nil, "", nil, nil, nil)
```
_Figure 3. Example of a common test helper function with several parameters._

_Figure 3_ Isn't really pleasant or readable. Each new parameter just makes the function longer and harder to read. With our fluent style we can do the following.  
```
cluster.
    WithNode("node1").AsSchedulable().WithAllocatablePods(6).Build().
    WithNode("node2").AsUnschedulable().Build()
```
_Figure 4. Fluent style test helper function chaining._

_Figure 4._ is an example of how much we can improve the readability and understandability of tests.

# Testing convergence
Cluster state especially at large scale is rarely ever predictable and more closely exhibit the properties of an eventually consistent system. Testing convergence and verifying whether cluster state behaves how we expect is important in Navops Command. We want to ensure our policies are considering the following.

1. Cluster eventually converges to an expected state.
1. Convergence is timely and efficient but not overly sensitive that we cause cluster instability (thrashing, flapping, etc).

With our cluster state simulator we can add fluent style functions to inject state changes and we can loop through these changes and verify our policies make the correct decisions along the way.

# Testing fault tolerance
Simulated tests don't replace acceptance tests running against a real cluster but our cluster simulator does allow us to test failure scenarios and ensure the right decisions are made under these failures. We can also combine fault and convergence tests into a new test that survives failures and tests that cluster state is eventually converged to the state we expect.

# How to create a fluent style API
In my experience the implementation of a fluent style API in Go is specifically tailored to your needs. Because of that I'm going to keep this section fairly short and just give you a small sample of where to begin.

Here are a few guidelines to remember.
1. You will need sub objects for separating fluent API's. For example `ClusterBuilder` will need a `ClusterBuilderNode` and a `ClusterBuilderPod`.
1. Fluent API's need to return the parent object they are associated with so you can chain function calls. `ClusterBuilder.WithNode()` returns `ClusterBuilderNode` so you can chain calls like `ClusterBuilder.WithNode().AsSchedulable()`
1. You need a way to return back up the object chain. One way to do this is create a `Build()` function on each object. For example. 
```
ClusterBuilder. 
        WithNode("foo").AsSchedulable().Build().
        WithNode("bar").AsUnschedulable().Build()
```
_Figure 5. Build function returning parent object._

Here's how you implement some of the above. I'm leaving a lot of implementation details out to keep the example easy to read.

```
type ClusterBuilder struct {
    // Some state you need to retain.
    Nodes map[string]*types.Node
    Pods  map[string]*types.Pod
}

type ClusterBuilderNode struct {
    // Some state you need to retain.
    builder *ClusterBuilder
    node    *types.Node
}


func (builder *ClusterBuilder) WithNode(nodeName string) *ClusterBuilderNode {
    node := builder.createTestNode(nodeName)
    clusterBuilderNode := NewClusterBuilderNode(builder, node)
    return clusterBuilderNode
}

func (node *ClusterBuilderNode) Build() *ClusterBuilder {
    return node.builder
}
```

# Concerns
Once you go down this fluent API path you may realize there are a few gotchas such as the following.
1. This style treats all calls as optional and you may identify some calls you need are required.
1. Some chained calls may require specific order.

One place you can enforce rules is in the `Build()` function of each type.

# Closing
I've really enjoyed the evolution of our simulated and acceptance tests. It has greatly improved the productivity and the understandability of our tests while giving us a foundation to build more advanced simulation features without increasing the difficulty of reading and understanding our tests. It's a breath of fresh air to open up a complex test and be able to understand it quickly.

If you have any questions or feedback feel free to [contact me](mailto:simon@nureality.ca). Shameless plug, if you're using Kubernetes and interested in Navops Command [check us out](https://www.univa.com/products/navops.php).