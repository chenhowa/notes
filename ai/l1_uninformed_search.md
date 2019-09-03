# Uninformed Search

We need a formalism for letting rational agents plan ahead. When there are multiple ways to achieve a goal, a planning agent needs to simulate plans that have a high chance of success. The actions, and their consequences, are considered. It must model how the world evolves in response to actions.

In contrast, reflex agents choose their action based on the current state of the world, and maybe memory of the past states. This is a lot like closing your eyes when something tries to fly into it. This doesn't consider the future consequences of the action. Reflex agents can only maximize expected utility under a __very limited__ set of circumstances (when greedy algorithms fit the problem)

Optimal planning means the solution of the algorithm is one of the best solutions. Complete planning means that if a successful plan exists, the algorithm will definitely find it.

Planning refers to executing a plan deterministically. Replanning means you may replan in the middle of execution, because execution was not determinstic (the world was too complicated, or too expensive to plan that far ahead). Replanning is more likely to be suboptimal, but it performs better in real time.

## Search problems 

A search problem consists of 

* A state space -- all possible states, where the state keeps only the details needed for planning. This is separate from the *world states*, which include all details of the environment. You will typically exclude constants from state, which are incorporated into the successor function instead.
* A successor function - (Action) -> (NewState, Cost)
* A start state, and a goal test - how do we know whether the goal 
    has been met during the plan?

A solution is a sequence of actions (a plan) which transforms the start state to a goal state.

World state spaces can be HUGE.
State spaces, that cut out variables can cut downt the space size considerably.

### State Space Graphs and Search Trees

A State space graph is a mathematical representation of a search problem. Each node is a world configuration. Each edge represents successors (from the successor function). Each state occurs only once. It is too big to build in memory, however. We will only build a subset at any time.

A search tree has the start state as the root, and then grows one layer down for each step of lookahead (each successive application of the successor function). Each path from tree to node encodes a plan (one of many) to each that state. But for most problems, building the entire tree is usually intractable, just like building the entire state space graph is. Now, unfortunately a search tree can be infinite. What we want to do is avoid repeating states over and over again.

When building a search tree, you maintain a fringe of partial plans under consideration (you keep track of the nodes that you are still thinking of expanding with the succcessor function) - see __40:20__. If the fringe is ever empty, the search has failed to find a solution. If the fringe contains the Goal state, you are done. Otherwise, you remove a node from the fringe, expand it to successor nodes (possibly no successors), and add them back to the fringe (you have to maintain the entire plan of the nodes in the fringe though)

If using DFS to do the search (that is, maintain the fringe as a LIFO structure), we get completeness (will find solution if one exists, if we check for cycles), but the path may not be optimal. Time complexity, and space complexity, are hard to know.

If we define b as the maximum branching factor, and m is the maximum depth of the search tree, then space of search tree is O(b<sup>m</sup>) nodes, so worst case of search tree is looking through all nodes. So time complexity is O(b<sup>m</sup>). Space complexity of maintaining the fringe is O(b * m), because in worse case, you go all the way down the tree, and have added the b successors in each level to the fringe. Then that whole path gets removed.

If using BFS to do the search (maintain the fringe as a FIFO structure), you get completness (if one exists, and check for cycles), and the path is not optimal in general (because some actions have different costs, so the shortest path is not necessarily the cheapest path)

If we say that the shallowest goal state is at a depth of *s*, the search tree space is of size O(b<sup>m</sup>). Fringe space and time complexity is O(b<sup>m</sup>)

BFS outperforms DFS when you're looking for shortest path, or when goal is definitely close to initial state.

DFS outperforms BFS when the tiebreaker is really smart, or lucky. But how do we get a smart tiebreaker? DFS also outperforms BFS when memory is a concern, since fringe space complexity is better.

We can get a hybrid with something called "iterative deepening".

* Run a DFS with depth limit 1. If no solution,
* Run a DFS with depth limit 2. If no solution,
* Run a DFS with depth limit 3 .. and so on

This has a memory complexity of (b * s) for memroy, and O(b<sup>s</sup>) for running time, and you're guaranteed to find the solution closest to the root. You are redoing some work, but the redone work is small compared to the work at the next level (b^n - b^(n-1)) >>> (b^(n-1))

### Cost Sensitive Search

In uniform cost search, we expand the cheapest node first (fringe is a priority queue based on the cumulative cost of the path). This guarantees an optimal plan, and clearly it is complete (if minimum cost of an action is greater than 0. Otherwise an infinite loop may occur)

Define C* as the cheapest possible cost to reach the goal. Then before reaching the goal, we look at all nodes where cost to reach it is less than C*. Define e > 0 as the the minimum cost of actions. Then any plan with a cost C \<= C* has to be at a depth of C* / e. __This explanation makes no sense. Might need to look this up in Norvig__

Under these assumptions, the space and time complexity are both O(b<sup>C*/e</sup>)

Downsides of UCS is that it looks in every direction without any preference -- no tiebreakers (no heuristics to cut down the search time). When we do have good heuristics, we can do better - A* search.