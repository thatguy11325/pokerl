+++
title = 'The Environment'
weight = 2
+++

# The Pokémon Environment

## Game Overview

Before discussing observations, rewards and the policy we will provide a brief overview of the game's challenges. Pokémon is a complex game with multiple tasks and puzzles that can be accomplished nonlinearly. We cover Pokémon's challenges in greater depth in the [Appendix]({{<ref "/docs/appendix/breakdown/index" >}} "appendix").

At a high level, to beat Pokémon, you must:

1. Beat the 8 Gym Leaders (a form of video game boss).  
2. Acquire `HMs` (items) to teach the field moves `CUT`, `STRENGTH`, and, `SURF` and Pokémon to teach the field moves to. Field moves are abilities that can be used outside of battle to unlock a new area or make it easier to traverse an existing area.  
3. Teach capable Pokémon the field moves `CUT`, `STRENGTH`, and `SURF`.
4. Acquire any items (not HMs) required for field interactions. Like field moves, there are items that are required to unlock new areas.  
5. Use field moves or items for field interactions to remove any game blocking obstacles.  
6. Complete the Team Rocket storyline.     
7. Beat the 6 required rival battles.
8. Beat the Elite 4 and Champion.

Or, if we want to be more detailed...

{{< mermaid >}}
---
config:
  theme: mc
  look: handDrawn
  layout: elk
---
flowchart TD
    AA(Acquire starter Pokémon from Prof. Oak)
    AA --> AAA(Acquire Parcel from Viridian Mart) --> AAAA(Deliver Parcel to Prof. Oak)
    AAAA --> A(Defeat Brock) --> B(Traverse Mt. Moon)
    B --> C(Defeat Misty)
    B --> D(Nugget Bridge) --> E(Get the SS Anne ticket from Bill) --> F(Defeat Cerulean Rocket Grunt) --> G(Defeat Rival on the SS Anne) --> H(Obtain HM01 - CUT from the Captain on the SS Anne)
    C --> I
    C --> J
    C --> K
    C --> M
    H --> HH(Defeat Rocket grunt protecting Rocket Hideout) --> HHH(Flip switch in Celadon Game Corner to unlock the Rocket Hideout)
    HHH --> KK(Defeat Rocket grunt with Lift Key) --> KKK(Obtain the Lift Key) --> K(Defeat Giovanni in Rocket Hideout) 
    K --> LL(Collect the Silph Scope from Giovanni) --> LLL(Use the Silph Scope on the ghost in Lavender Tower) 
    LLL --> L(Save Mr. Fuji at the top floor of Lavender Tower) 
    H --> I(Defeat Lt. Surge)
    H --> J(Defeat Erika) 
    H --> MM(Obtain a drink from the vendeing machines at the top of Celadon Mart) 
    MM --> M(Deliver Drink to Saffron Guard) --> O
    L --> N(Obtain Pokéflute)
    L --> OO(Defeat Rival 5 in Silph Co) --> O(Defeat Giovanni in Silph Co) --> P(Defeat Sabrina)
    N --> Q(Use Pokéflute on at least one Snorlax)
    Q --> R(Obtain HM03 - SURF from the Safari Zone) --> T(Acquire the Secret Key from Pokémon Mansion) --> U(Defeat Blaine)
    Q --> V(Defeat Koga) --> T
    Q --> SS(Obtain the Gold Teeth from the Safari Zone) --> S(Deliver the Gold Teeth to the old man in Fuchsia City to acquire HM04 - STRENGTH)
    I --> W(Defeat Giovanni in Viridian Gym) --> X(Defeat Rival 6) --> Y(Traverse Victory Road)
    J --> W
    P --> W
    U --> W
    S --> Y
    Y --> Z(Defeat the Elite 4) --> ZZ(Defeat the Champion)
{{< /mermaid >}}

## Defining a "Route"

For the agent to complete all objectives, we wanted to simplify the number of game as much as possible to maximizing the likelihood of success. To limit non-determinism, we started all environments after the "Parcel Delivery" event. Additionally, we want to guarantee the agent acquires the gift Lapras. Starting the agent with a specific Pokémon would guarantee later stages of the game would be possible. Given the previous breakdown, here’s a simplified *route* we had the agent to learn:

{{< mermaid >}}
---
config:
  theme: mc
  look: handDrawn
  layout: elk
---
flowchart TD

START("Start with Bulbasaur or Charmander to guarantee a Pokemon who can use CUT")
BROCK("Defeat Brock")
MISTY("Defeat Misty")
HM01("Acquire HM01")
TEACH_CUT("Teach CUT")
LTSURGE("Defeat Lt. Surge")
ERIKA("Defeat Erika")
KOGA("Defeat Koga")
SABRINA("Defeat Sabrina")
BLAINE("Defeat Blaine")
GIOVANNI("Defeat Giovanni")
HM03("Acquire HM03")
TEACH_SURF("Teach SURF")
HM04("Acquire HM04")
TEACH_STRENGTH("Teach STRENGTH")
TEAM_ROCKET("Complete Team Rocket Storyline")
LAPRAS("Acquire the gift Lapras")
RIVAL6("Defeat Rival 6")
E4("Defeat E4 and Champion")

START --> BROCK 
BROCK --> MISTY
MISTY --> TEACH_CUT
BROCK --> HM01 --> TEACH_CUT --> LTSURGE
MISTY --> LTSURGE
TEACH_CUT --> ERIKA
TEACH_CUT --> TEAM_ROCKET
TEAM_ROCKET --> LAPRAS --> TEACH_STRENGTH
LAPRAS --> TEACH_SURF
TEAM_ROCKET --> KOGA
TEAM_ROCKET --> SABRINA
TEAM_ROCKET --> HM04
TEAM_ROCKET --> HM03
KOGA --> TEACH_SURF
HM03 --> TEACH_SURF --> BLAINE
ERIKA --> TEACH_STRENGTH
HM04 --> TEACH_STRENGTH

LTSURGE --> GIOVANNI
SABRINA --> GIOVANNI
BLAINE --> GIOVANNI

GIOVANNI --> RIVAL6 
TEACH_STRENGTH --> E4
RIVAL6 --> E4


{{< /mermaid >}}

This route is *extremely* complex. The average human player will take 25 hours to beat Pokémon on the first try. Incentivizing exploration for a space this big took a lot of effort.

### Risk Management

Even though we simplified the game, we still needed risk mitigation. Pokémon contains numerous game ending risks including:

* Permanently losing vital Pokémon  
* Catching Pokémon and not having enough space for the Lapras or any other Pokémon that can learn `SURF`  
* If the agent obtains too many items, then there may not be enough room for key items  
* It is possible to soft-lock if you cannot obtain any more money at the time the Safari Zone objectives need to be completed.  
* Only having Pokémon with non-damaging moves.

We’d like to emphasize that *none* of these issues require teaching the agent how to best Pokémon battles.

### Scripting vs. Emergence

To handle these risks and difficulties, we split agent behavior into two classes:

* **Scripted:** Meaning actions taken by the agent not controlled by the policy.  
* **Emergent:** Allowing the agent to discover its own strategies.

Ideally, no scripted behavior would be needed. However, we needed scripts. We aimed to only script behaviors that require human intuition or tasks that are not really a core part of Pokémon. These included.

* Item management - the agent will toss all non-key items if the agent fills their inventory.
* Money management - the agent will have infinite money.  
* Solving puzzles that require `STRENGTH`. 
* Blocking the Indigo Plateau exit at the end of the game.

To make development easier, we wrote scripts to remove complexity and speed up development. When writing RL systems, this is common practice. Make the simplest environment possible, then slowly roll back the assumptions. These scripts are now unneeded, but were invaluable during development. They included

* Teaching HMs.
* Automatically using HMs outside of battle.
* Automatically Pokéflute.
* Maxxing the Pokémon’s stats.
* Automatic insertion of drinks for the Saffron guards if the agent *entered* the Celadon Mart.
* Automating elevator usage. If an agent entered an elevator, the elevator would go to the next floor modulo number of floors.
* Disabling wild battles to simplify dungeons.

## Playing the Environment

The Python library [Gymnasium](https://gymnasium.farama.org/) provides a fairly straightforward API for defining an environment. The environment can be simplified into two functions: `Step` and `Reset`

## Steps

`Step` is a function which is given actions for the environment and returns an observation, any logging info and whether or not to the environment should reset.

Our step function for Pokémon can naively be written as:

- Receive a button press action.
- Send the button press to the gameboy emulator.
- Wait some frames for the action to have an effect on the environment.
- Sample the environment.
- Return data based on the sample.

<div style="border:1px solid black;">
{{< highlight python >}}

def step(self, action: int):
    """
    A simplified version of the step function run during training
    """
    self.pyboy.send_input(VALID_ACTIONS[action])
    self.pyboy.send_input(VALID_RELEASE_ACTIONS[action], delay=8)
    self.pyboy.tick(self.action_freq - 1, render=False)
    return self.get_obs(), self.get_reward()
{{< /highlight >}}
</div>

## Resets, Episodes and the Goal

Reset handles the initialization for an *episode*. An episode is a sequence of actions that ends when some terminal state is reached.

A reset will occur when an environment:

- Achieves a major milestone, such as defeating a boss.
- Encounters a failure state, such as losing all their lives.
- Reaches a predefined time or action limit, such as 100 tetriminos in Tetris,

Pokémon is in the realm of "long episodic RL." There is no strict rule on what a long episode is, but Pokémon takes 25 hours for the average person to beat. That's much longer than a session of Pac-Man or Breakout. Long episodes mean that the environment may not return large rewards for a very long time. If the environment only returns large rewards at the end of an episode, the agent may have trouble learning long term policies.

For example, if we were playing 100x100 Tic-Tac-Toe, the environment wouldn't return a reward for up to a max 5000 steps. Imagine having to plan out a 5000 step strategy. It's not easy! Later, we'll go over how we handled rewards for Pokémon's long episodes.

Based on Peter Whidden's prior work, we began with an episode as a fixed number of steps. Over time, we tried other strategies such as dynamically increasing the number of steps for per episode as important milestones occur within an environment. However, we decided a Pokémon Red episode's terminal state is when an unrecoverable state (soft-lock) is met, e.g., such as running out of money or when the game is complete. 

Because our **goal** was focused on becoming the Champion (a very long task), we compromised. We created "mini-episodes.” An episode would be the duration of an entire game. The environment's state would periodically reset mid-episode, but the emulator state would not:

<div style="border:1px solid black;">
{{< highlight python >}}

class OnResetExplorationWrapper(gym.Wrapper):
    """
    Wrapper for experimenting with different reset strategies. The OnResetExplorationWrapper,
    used in the majority of experiments, resets all memory every get_max_steps() steps + jitter.
    """
    def __init__(self, env: RedGymEnv, reward_config: DictConfig):
        super().__init__(env)
        self.full_reset_frequency = reward_config.full_reset_frequency
        self.jitter = reward_config.jitter
        self.counter = 0

    def step(self, action):
        if self.env.unwrapped.step_count >= self.env.unwrapped.get_max_steps():
            if (self.counter + random.randint(0, self.jitter)) >= self.full_reset_frequency:
                self.counter = 0
                self.env.unwrapped.explore_map *= 0
                self.env.unwrapped.reward_explore_map *= 0
                self.env.unwrapped.cut_explore_map *= 0
                self.env.unwrapped.cut_tiles.clear()
                self.env.unwrapped.seen_coords.clear()
                self.env.unwrapped.seen_map_ids *= 0
                self.env.unwrapped.seen_npcs.clear()
                self.env.unwrapped.valid_cut_coords.clear()
                self.env.unwrapped.invalid_cut_coords.clear()
                self.env.unwrapped.valid_pokeflute_coords.clear()
                self.env.unwrapped.invalid_pokeflute_coords.clear()
                self.env.unwrapped.pokeflute_tiles.clear()
                self.env.unwrapped.valid_surf_coords.clear()
                self.env.unwrapped.invalid_surf_coords.clear()
                self.env.unwrapped.surf_tiles.clear()
                self.env.unwrapped.seen_warps.clear()
                self.env.unwrapped.seen_hidden_objs.clear()
                self.env.unwrapped.seen_signs.clear()
                self.env.unwrapped.safari_zone_steps.update(
                    (k, 0) for k in self.env.unwrapped.safari_zone_steps.keys()
                )
            self.counter += 1
        return self.env.step(action)
{{< /highlight >}}
</div>


