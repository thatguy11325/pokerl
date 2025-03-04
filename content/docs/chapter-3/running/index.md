+++
title = 'Running'
weight = 40
+++

# Actually Coding the RL System

So we have a policy, observations, an environment and a reward. Now it's time to discuss how we wrote the RL system. Along the way, we'll provide relevant snippets of code and tricks done to make the system run as fast as possible.

For compute, Joseph Suarez, the creator of [PufferAI](https://puffer.ai/) and the [PufferLib](https://github.com/PufferAI/PufferLib) RL Python library, graciously donated 4 machines to this effort. Each machine contained a NVIDIA 4090 GPU, Intel Core i9-14900K, 125 GB of RAM and 2TB of disk space. Before his contribution, we were paying $200/month using [vast.ai](https://vast.ai/) for training with worse hardware.

Good hardware although not necessary, massively sped up training.

But what was more important than good hardware? Good software! Lets discuss some of the challenges of the Pokémon environment and mitigations for those challenges. 

In the end, we 10x’d the steps per second (sps) of training to a peak of 10000 sps with a lot of extra engineering effort. Now, it's possible to beat Brock in less than 30 minutes! Potentially more effort was put into making the system performant than making the system beat Pokémon.

## Pokémon is CPU Bound

### The Emulator

[PyBoy](https://github.com/Baekalfen/PyBoy) happens to provide a very convenient interface for RL and is performant for Python. However, using an emulator comes with its downsides. Namely, an emulator emulates every instruction from the ROM. That means fewer compiler optimizations, no link time optimizations etc. We could have attempted to rewrite Pokémon in a modern language, but that would have been a Herculean effort on its own. Instead, we made PyBoy faster.

A slow PyBoy *was* the case. But we've been in frequent communication with the creator of PyBoy for a while and we (mostly the PyBoy creator) have put a lot of effort into improving PyBoy since late 2023. PyBoy has released a number of updates that have improved runtime speed and developer ergonomics dramatically including:

- Better usage of compiler optimization flags.
- Better usage of Cython’s gil release options.
- Data layout optimization.
- The hooks API.
- JIT compiling the ROM (coming soon™).

Most of these changes are in PyBoy’s core. Hooks are a special addition that let us inject Python code at specified instructions. Although the context switch back to Python slows down execution, the less frequent RAM reads hooks allow have led to an overall speed up in Pokémon. However, even now, the environment still uses the majority of training time!


<div style="text-align: center; ">

{{< figure
  src="assets/flamegraph.png"
  caption="A profile of an environment taken mid-experiment using [Austin](https://github.com/P403n1x87/austin). Notably, most processing is spent in `run_action_on_emulator`. The PyBoy emulator's tick function is called multiple times in that function."
  class="ma0 w-75"
  link="assets/flamegraph.png"
>}}

</div>

### Collecting Screen Data

Do we need to collect data *every* frame? No. In fact, it is suboptimal. Most animations do not provide value to the player. This is well documented in Peter Whidden’s video. In order to use PyBoy effectively: 

- We render the game headless (not every frame is rendered). No UI means less CPU time spent rendering the game.  
- For every action, we tick PyBoy 24 times and record the game screen on the *last* step. 24 about the time it takes for the player to take one step in the overworld. In the future, we could make different buttons tick PyBoy different amounts.
- We don’t collect information from memory every step.

<div style="border:1px solid black;">
{{< highlight python >}}

def run_action_on_emulator(self, action):
    # press button then release after some steps
    if not self.disable_ai_actions:
        self.pyboy.send_input(VALID_ACTIONS[action])
        self.pyboy.send_input(VALID_RELEASE_ACTIONS[action], delay=8)
    self.pyboy.tick(self.action_freq - 1, render=False)

    self.update_seen_coords()

    # Wait until control is given back to the user
    for _ in range(1000):
        if not self.read_m("wJoyIgnore"):
            break
        self.pyboy.button("a", 8)
        self.pyboy.tick(self.action_freq, render=False)

    # One last tick just in case
    self.pyboy.tick(1, render=True)

{{< /highlight >}}
</div>

We additionally tick until the player is given back control. Waiting for control to return to the player removes non-determinism, e.g., the extra animation when moving a boulder with `STRENGTH`.

## When to Care About the GPU

Generating data on the CPU has generally been the bottleneck, but training and inference do occur. During the training phase, we do not run the emulator.

The policy is currently written in PyTorch and we have taken time to improve training speed where possible. Over time we have looked into:

* Decrease the model size.
* Decrease the data size.
* Improve the GPU utilization.
* Improve sample efficiency.

Some improvements have been contributed back to PufferLib (since early 2024 we have provided PufferLib with a 2x speed up). 

Of the easier to add improvements, We achieved a 30% GPU speed improvement on inference and improved GPU utilization with [torch.compile](https://pytorch.org/tutorials/intermediate/torch_compile_tutorial.html). Torch compilation traces the execution graph of a PyTorch module and will create an optimized GPU execution graph.

Another speed optimization easy win came making sure we used [pinned memory](https://developer.nvidia.com/blog/how-optimize-data-transfers-cuda-cc/#pinned_host_memory) for any tensors on GPU. Pinning prevents an extra memory copy when moving data from the host CPU to the GPU device.

## Observation Size Matters

In the end, the biggest bottleneck happened to be the data transfer of the observation. Not just from the CPU to GPU, but also from each agent back to the buffer holding all agent experiences for the current policy.

{{< mermaid >}}
---
config:
  theme: mc
  look: handDrawn
  layout: elk
---
flowchart LR

  subgraph Workers["Workers"]
  direction LR
     subgraph Worker["Worker"]
     direction TB 
     Env1("Environment 1") -.- Env2("Environment N")
     end
     Worker2("Worker N")
     Worker -.- Worker2
  end

  Trainer("Trainer")
  GPU("GPU")

  Workers -->|Sends observations infos rewards| Trainer
  Trainer -->|Sends actions| Workers

  Trainer -->|Sends data for inference| GPU
  GPU -->|Sends inference results| Trainer

{{< /mermaid >}}

Since the model is 20MB, we can fit huge batches onto the GPU. Higher batch size means tons more data being passed around the system.

We looked into ways to decrease the memory size of the batch sent to the GPU. Along with easy wins like choosing smaller data types, we developed some techniques that were really neat and we want to brag about them.

### Compressing Data

The GameBoy playable area is 144x160 pixels. PyBoy always returns 4 channels, RGBA. However, GameBoy games are grayscale so we really only need 1 channel. 

{{% hint info %}}
--> 1/4 reduction in screen data size or 92kB --> ≈23kB
{{% /hint %}}

Additionally, as Peter Whidden showed, the agent doesn't need the *full* screen. Half resolution or downscaling the image to 72x80 pixels is fine.

{{% hint info %}}
--> 1/4 reduction in screen data or 23kB --> ≈6kB
{{% /hint %}}

We can go one step further. GameBoy’s color system is 2 bit. Every pixel takes 2 bits representing one of 4 values. Even though PyBoy does not return the dense layout of the game screen, we could compress the data such that every 8 bits, that is 1 byte, represents 4 pixels. Then, we could decompress the data on the GPU which is fast since GPUs are designed for massively parallel operations.

{{% hint info %}}
--> 1/4 reduction in data size or 6kB --> ≈2kB
{{% /hint %}}

And there we had a potential near 64x speed up in host\<-\>device communication. How do we unpack on the GPU quickly?

To unpack on the GPU, we store two buffers, a bit mask and a number of shifts. We shift the bytes using a [broadcast operation](https://pytorch.org/docs/stable/notes/broadcasting.html) then apply the bit mask in another broadcast operation and voila hyper parallel decoding for both the visited mask and the game screen.

<div style="border:1px solid black;">
{{< highlight python >}}

PIXEL_VALUES = [0, 85, 153, 255]

class Env(gym.Env):
    def collect_screen_data(self):
        # (144, 160, 3)
        game_pixels_render = np.expand_dims(self.screen.ndarray[:, :, 1], axis=-1)
        ...
        # Don't worry about aliasing effects
        game_pixels_render = game_pixels_render[::2, ::2, :]
        game_pixels_render = (
            (
                np.digitize(
                    game_pixels_render.reshape((-1, 4)), 
                    PIXEL_VALUES, 
                    right=True,
                ).astype(np.uint8)
                << np.array([6, 4, 2, 0], dtype=np.uint8)
            )
            .sum(axis=1, dtype=np.uint8)
            .reshape((-1, game_pixels_render.shape[1] // 4, 1))
        )
        ...


class Policy(nn.Module):
    def __init__(self):
        ...
        self.register_buffer(
            "screen_buckets", 
            torch.tensor(PIXEL_VALUES, dtype=torch.uint8), 
            persistent=False
        )
        self.register_buffer(
            "unpack_mask",
            torch.tensor([0xC0, 0x30, 0x0C, 0x03], dtype=torch.uint8),
            persistent=False,
        )
        self.register_buffer(
            "unpack_shift", 
            torch.tensor([6, 4, 2, 0], dtype=torch.uint8), 
            persistent=False
        )

    def unpack_screen_data(self, screen: torch.Tensor):
        screen = torch.index_select(
            self.screen_buckets,
            0,
            ((screen.reshape((-1, 1)) & self.unpack_mask) >> self.unpack_shift).flatten().int(),
        ).reshape(restored_shape)
        visited_mask = torch.index_select(
            self.linear_buckets,
            0,
            ((visited_mask.reshape((-1, 1)) & self.unpack_mask) >> self.unpack_shift)
            .flatten()
            .int(),
        ).reshape(restored_shape)

{{< /highlight >}}
</div>

This data technique also works for the events vector. Instead of representing every event flag as 1 byte, we send the entire events vector to the GPU, i.e., 1 byte held 8 flags. On the GPU, and performed a similar shift and mask! Fast decompression of over 2000 event flags.

## A Good Vectorized Env is Worth its Weight in Gold

In between the emulator and the model, we had code managing a vectorized environment. A vectorized environment runs many copies of the same environments in parallel. Originally, we used the library [Stable Baselines 3](https://stable-baselines3.readthedocs.io/en/master/). It’s a great library for prototyping with RL, but it handles env vectorization suboptimally. SB3 will wait for all environments to finish their current step before calling the next step.

In early 2024, we adopted PufferLib from PufferAI. PufferLib provides an asynchronous vectorized environment implementation. Our PufferLib-based implementation collects data from environments until enough examples have been collected. Once enough examples have been collected, a few epochs of training occurs. 

## Good Hyperparameters can mean faster training

The last dimension for a faster training experience isn’t making the overall iteration time faster, but improving sample efficiency. Once we had a game winning model, we spent over a month sweeping hyperparameter to find parameters that would produce a winning run in the fastest amount of time. Hyperparameters define any configurable part of the system. These are parameters that are not intended to be learned.

Most hyperparameter sweep libraries will optimize for a single metric, such as reward. We wanted to optimize for reward with a time penalty. [CARBS from Imbue](https://github.com/imbue-ai/carbs) is a *cost-aware* hyperparameter sweep tool. With proper hyperparameters, we reduced a single training run (that uses scripts) from 2 days to 7 hours. A nearly 7x reduction in training time.

<div style="text-align: center; ">

{{< figure
  src="assets/sweep.png"
  caption="One of many hyperparameter sweeps we ran. Our goal was to find a run that beat the game in as few steps as possible."
  class="ma0 w-75"
>}}

</div>