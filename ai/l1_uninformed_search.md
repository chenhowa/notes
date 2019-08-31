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

31:00 




