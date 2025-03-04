---
menu:
title: 'Chapter 1: Brief RL Intro'
weight: 1
---

# Reinforcement Learning Quickstart

Before discussing Pokémon, lets start with brief (and hand wavy) introduction to Reinforcement Learning (RL). The introduction will motivate a framework for how to discuss the Pokémon in an RL setting. 

RL studies how an agent should take actions in an environment in order to maximize a reward. In a typical RL framework, there will be 2 components.

1. The Environment - Represents the world context.
2. The Agent - Responsible for generating actions to perform in the environment.

The agent contains a *policy* which generates an action given a state in an environment. The environment evaluates the action and returns an *observation* and a *reward*. These actions, observations and rewards can be used to update the agent's policy. [Deep Reinforcement Learning](https://en.wikipedia.org/wiki/Deep_reinforcement_learning) (DRL) uses a neural network for the policy. The Pokémon RL System uses DRL to beat Pokémon.

{{< mermaid >}}
---
config:
  theme: mc
  look: handDrawn
  layout: elk
---
flowchart LR
    Agent --> |Action| Environment(Environment)
    Environment --> |Observation, Reward| Agent

    subgraph Agent[Agent]
        Policy(Policy)
    end

{{< /mermaid >}}

When building an RL system, we break down the problem into:

- the goal
- the environment
- the observation
- the actions
- the reward
- the policy / the agent

because doing a little pre-work saves time later.

## Tic Tac Toe

Let's apply RL to Tic-Tac-Toe. RL does not need deep learning. A valid RL agent can be as simple as a look up table. Tic-Tac-Toe is a two player game where each player takes turns marking a single square in a 3x3 grid. The first player to have marked 3 squares making a horizontal, vertical or diagonal line wins. Let's breakdown Tic-Tac-Toe in the framework above:

```
Example Tic-Tac-Toe configurations:
   X Wins          Tie         Early Game
 X | O | X      X | O | X      X |   | O  
-----------    -----------    -----------
 O | X | O      X | X | O        | O |    
-----------    -----------    -----------
   |   | X      O | X | O        |   | X  

```

### Goal

The goal of Tic-Tac-Toe is to win. After a player has achieved 3-in-a-row, the game will reset. We call the time it takes from beginning to the end of the game an *episode* (more on this later).

### Environment

The environment is the Tic-Tac-Toe game. The environment will be responsible for handling actions, i.e., marking squares, ensuring invalid moves are not played, and determining when the game has ended.

### Observation

The observation will be the current 3x3 grid. It can be represented as a 3x3 array.

### Actions

The action will be a coordinate to play on the grid. For example [0, 0] means mark  the upper left hand corner. [2, 2] means mark the lower right hand corner.

### Rewards

We can start with a reward of

-  +1 if the player wins
-  -1 if the player loses

The act of modifying the reward for a better system is known as *reward shaping*. 

### Training the Policy

Now that we have created our Tic-Tac-Toe RL system, we can attempt to train a policy to be the best Tic-Tac-Toe player ever. At the start of training, the policy will play random moves, but as the policy obtains more experience, it will learn to exploit the reward function.

For Pokémon, we will similarly break down the game. Let's dig in.

