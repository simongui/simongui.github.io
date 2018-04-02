---
layout: post
title: Fluent style testing in Go
draft: true
---
Recently my team has been working on Kubernetes eviction policies for [Navops Command](https://www.univa.com/products/navops.php). We have some exciting new features coming soon and I'm delighted to write a bit about how we tested them along the way. 

Navops Command is written in Go and we needed a few improvements to make the process of development, testing and verifying safety guarantees to be more efficient so that engineers could iterate quickly and test clustered scenarios thoroughly. 

The challenge with testing clustered features is that not everyone has the ability to spin up 1,000 node clusters and even if you did, running tests on them takes a lot of time and these environments aren't efficient for iterating quickly during development. Writing tests that use simulated clusters isn't a new concept but it's one that if your code is designed from the ground up to be easily simulated it becomes much easier. 

A few key improvements we put effort into that really improved our simulated cluster tests are the following.
1. Using a fluent style to make tests really easy to read and write.
2. Simulate and verify cluster convergence.

# Fluent style tests
To give a little bit of insight on our development process, we document what use cases our features need to satisfy and we write tests to verify them. Typically use cases are easier to read in document form but not easy to read in test form because of all the setup boiler plate. Even with traditional "helper" functions tests are usually hard to read. We wanted our simulated tests to be as easy to read as our documented use cases.

Simulated clusters of nodes and Kubernetes pods have a lot of structs and attributes involved and our goal was to hide all of that complexity. A traditional style helper you might be familiar with might do something like the following.
```
addNode("node1", foo, bar, 100, something)
```

No doubt `addNode()` is probably hiding a lot of complexity and makes configuring the test a lot shorter but it's not really that readable overall. It isn't obvious what the parameters mean. It takes a lot of digging to understand what the cluster state is.

We wanted to improve this experience and applied a fluent style to our test helpers that configure our simulations. If you're not familiar with fluent API's, it's a technique to create a DSL that reads clearly in english so that the code is self describing. 

The following is a basic example of one of our simple tests.

```go
// TestPackPolicySelectsCorrectCandidatesScenarioA verifies  
// that the following scenario succeeds.
//
// Scenario A
// Node1: Available slots: 6 running pods 1
// Node2: Available slots: 6 running pods 2
// Node3: Available slots: 6 running pods 3
//
// RESULT: Node1 and Node2 should get pods evicted to eventually run on Node3.
func TestPackPolicySelectsCorrectCandidatesScenarioA(t *testing.T) {
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
    candidate := cluster.Run()

    // Verify that the correct 3 pods are selected to be evicted.
    assert.EqualValues(t, 3, len(candidate.Preemptees))
    assert.Equal(t, "pod1", candidate.Preemptees[0].ID())
    assert.Equal(t, "pod2", candidate.Preemptees[1].ID())
    assert.Equal(t, "pod3", candidate.Preemptees[2].ID())
}
```

The helper functions are chained in an english sentence form. Go isn't the easiest language to implement fluent API's with but with a few things in place you can create a fluent style that allows you to add any number of parameters that you can chain together. 

There's 2 key improvements a fluent API gives us over  helper functions.  
1. Self describing, easy to read and chained in a sentence.
1. Unused optional parameters don't clutter the test.  
```
addNode("node1", true, nil, nil, nil, "", nil, "", nil, nil, nil)
addNode("node1", false, nil, nil, nil, "", nil, "", nil, nil, nil)
```
Isn't really pleasant or readable. Each new parameter just makes the function longer and harder to read. With our fluent style we can do the following.  
```
cluster.
    WithNode("node1").AsSchedulable().WithAllocatablePods(6).Build().
    WithNode("node2").AsUnschedulable().Build()
```
