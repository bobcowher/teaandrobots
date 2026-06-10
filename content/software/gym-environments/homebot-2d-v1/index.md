---
title: "HomeBot 2D V1"
date: 2026-06-07
draft: true
---

HomeBot 2D is a Gymnasium environment for reinforcement learning research. It's a top-down 2D simulation where an agent navigates a furnished home, collecting trash, fetching drinks from the kitchen fridge, and retrieving packages from the front door.

HomeBot 2D ships two Gymnasium environments: `HomeBot2D-v1` for standard training and evaluation, and `HomeBot2D-Goal-v1` for goal-conditioned training with Hindsight Experience Replay (HER).

![HomeBot 2D Environment](homebot-2d.png)

The environment provides RGB 84×84 observations suitable for CNN-based agents and supports both autonomous RL training and human keyboard play.

## Installation

**Version**: 0.1.0

```bash
pip install gym-homebot-2d
```

### With Training Dependencies

To train agents using Stable Baselines3:

```bash
pip install gym-homebot-2d
pip install stable-baselines3[extra]
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu129
```

## Quick Start

```python
import gymnasium as gym
import homebot

env = gym.make('HomeBot2D-v1', render_mode='human')
obs, info = env.reset()

for _ in range(1000):
    action = env.action_space.sample()
    obs, reward, terminated, truncated, info = env.step(action)
    if terminated or truncated:
        obs, info = env.reset()

env.close()
```

## Environment Specification

| Property | Value |
|----------|-------|
| Observation Space | `Box(0, 255, (84, 84, 3), uint8)` |
| Observation Type | RGB image |
| Action Space (discrete) | `Discrete(8)` |
| Action Space (continuous) | `Box([-1, -1], [1, 1])` |
| Max Episode Steps | 1,000 |
| Reward Range | [0, 1] per event |

### Actions (Discrete Mode)

| ID | Direction |
|----|-----------|
| 0 | North |
| 1 | Northeast |
| 2 | East |
| 3 | Southeast |
| 4 | South |
| 5 | Southwest |
| 6 | West |
| 7 | Northwest |

In continuous mode, the action is a 2D vector `[vx, vy]` in `[-1, 1]²` controlling steering direction and speed.

### Rewards

| Event | Reward |
|-------|--------|
| Pick up a piece of trash | +1 |
| Pick up drink from fridge and deliver to recliner | +1 |
| Retrieve package from door and deliver to recliner | +1 |
| Any other step | 0 |

## Goal-Conditioned Training (HER)

`HomeBot2D-Goal-v1` implements the `gymnasium.Env` interface with a Dict observation space for use with Hindsight Experience Replay algorithms.

### Observation Space

```python
{
    "observation":   Box(0, 255, (84, 84, 3), uint8),  # pixel image
    "achieved_goal": Box(0, inf, (2,), float32),        # robot (x, y)
    "desired_goal":  Box(0, inf, (2,), float32),        # target (x, y)
}
```

### Sub-Goals

`HomeBotGoalEnv` works in terms of sub-goals — atomic navigation targets that each resolve to a single pixel coordinate. The `goals` parameter accepts a list of sub-goal names; one is sampled uniformly per episode.

| Sub-goal | Target | Carry pre-loaded (training) |
|---|---|---|
| `go_to_fridge` | Fridge | — |
| `deliver_drink` | Recliner | drink |
| `go_to_door` | Door | — |
| `deliver_package` | Recliner | package |
| `collect_trash` | Random trash tile | — |

Delivery sub-goals pre-load the robot's carry state in training mode (`evaluate=False`) so the agent learns navigation to the delivery target independently from the pickup sequence. Set `evaluate=True` to disable this and test full pickup-to-delivery chains.

`goal_to_coordinates(name)` converts any sub-goal name to pixel `(x, y)`. It is available on both env classes and importable directly from `homebot.goals`.

### Quick Start

```python
import gymnasium as gym
import homebot

env = gym.make("HomeBot2D-Goal-v1", render_mode="rgb_array")

# Explicit goal:
obs, info = env.reset(seed=0, options={"goal": "go_to_fridge"})
# obs["desired_goal"] → fridge pixel coordinates

# Random goal each episode:
obs, info = env.reset(seed=0)
print(info["active_goal"])  # e.g. "collect_trash"

# Convert any sub-goal name to pixel coordinates:
x, y = env.goal_to_coordinates("deliver_drink")
```

### Reward

Sparse binary: `+1` when the robot reaches within 79 px of `desired_goal`, `0` otherwise. `compute_reward(achieved_goal, desired_goal, info)` supports batched inputs for HER relabeling.

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `goals` | all five | Subset of goal names to sample from |
| `evaluate` | `False` | When `True`, disables carry pre-loading |
| `render_mode` | `None` | `"human"` or `"rgb_array"` |
| `action_mode` | `"discrete"` | `"discrete"` (8 actions) or `"continuous"` |
| `obs_resolution` | `(84, 84)` | Observation image size in pixels |
| `n_trash` | `2` | Number of trash items to spawn |
| `max_steps` | `1000` | Episode step limit |

### Episode Termination

- **Terminated**: Active sub-goal reached (robot within 79 px of `desired_goal`)
- **Truncated**: Episode step limit elapsed

## Orchestrated Evaluation

For end-to-end evaluation with a multi-step curriculum, use `HomeBot2D-v1` with an external orchestrator. The orchestrator reads task state from `info`, selects the next sub-goal, and passes its pixel coordinates directly to the goal-conditioned policy.

```python
import gymnasium as gym
import homebot

env = gym.make("HomeBot2D-v1", render_mode="rgb_array")
obs, info = env.reset(seed=0)

while True:
    # Orchestrator selects sub-goal from currently active options
    sub_goal = orchestrator.decide(info["goals"], info["carrying"])

    # Convert sub-goal name to pixel coordinates for the policy
    gx, gy = env.goal_to_coordinates(sub_goal)

    action = policy(obs, goal=(gx, gy))
    obs, reward, terminated, truncated, info = env.step(action)

    if terminated or truncated:
        break
```

`info["goals"]` reflects the current task state — for example, once the robot picks up a drink, `"go_to_fridge"` is replaced by `"deliver_to_human"`. The orchestrator never needs to track carry state manually.

In the long term, the orchestrator can be replaced by an LLM that reads the same `info` fields and calls `goal_to_coordinates()` to ground its output before passing it to the policy. The environment interface is identical in both cases.

## Tasks

The robot operates in a fully furnished house with a living room, kitchen, hallway, and bedroom. Three concurrent task types are active each episode:

### Trash Collection

Two pieces of trash (a bottle and a can) spawn on random floor tiles at the start of each episode. The robot collects them by driving over them — no picking up required, it just needs to be within range.

### Drink Delivery

The robot fetches a drink from the kitchen fridge and delivers it to the person sitting in the living room recliner. Approach the fridge to pick it up, then approach the recliner to deliver.

### Package Retrieval

A package appears at the front door. The robot picks it up by approaching the door threshold and delivers it to the recliner.

The robot can only carry one item at a time. Trash collection still works while carrying a drink or package.

## Training with Stable Baselines3

A minimal PPO training example:

```python
import gymnasium as gym
import homebot
from stable_baselines3 import PPO

env = gym.make('HomeBot2D-v1', render_mode='rgb_array')

model = PPO(
    "CnnPolicy",
    env,
    learning_rate=3e-4,
    n_steps=2048,
    batch_size=64,
    verbose=1
)

model.learn(total_timesteps=500000)
model.save("homebot_ppo")
```

## Human Play

Play the environment yourself using keyboard controls:

```bash
conda run -n homebot python play.py
```

### Controls

| Key | Action |
|-----|--------|
| W / Arrow Up | Move north |
| S / Arrow Down | Move south |
| A / Arrow Left | Move west |
| D / Arrow Right | Move east |
| Diagonal combinations | Move diagonally |
| ESC | Quit |

## Environment Parameters

```python
env = gym.make(
    'HomeBot2D-v1',
    render_mode='rgb_array',   # 'human', 'rgb_array', or None
    action_mode='discrete',    # 'discrete' or 'continuous'
    obs_resolution=(84, 84),   # observation image size (H, W)
    n_trash=2,                 # trash items per episode
    max_steps=1000,            # steps before truncation
    map_name='default',        # map layout
)
```

## Info Dictionary

The `info` dict returned by `step()` depends on which environment is used.

**`HomeBot2D-v1`** returns full task state — useful for orchestrators:

```python
{
    'goals': list[str],          # active sub-goals given current carry state, e.g. ["go_to_fridge", "go_to_door"]
    'carrying': str | None,      # item the robot is currently carrying, or None
    'trash_remaining': int,      # pieces of trash not yet collected
    'drink_delivered': bool,     # whether drink has been delivered
    'package_delivered': bool,   # whether package has been delivered
    'package_present': bool,     # whether package is still at the door
}
```

**`HomeBot2D-Goal-v1`** returns goal-conditioned state for HER training:

```python
{
    'carrying': str | None,      # item the robot is currently carrying, or None
    'active_goal': str,          # name of the current sub-goal, e.g. "go_to_fridge"
}
```

## Reproducibility

For deterministic episodes:

```python
env = gym.make('HomeBot2D-v1')
obs, info = env.reset(seed=42)
```

## Requirements

- Python ≥ 3.9
- pygame ≥ 2.1.0
- gymnasium ≥ 0.26.0
- numpy ≥ 1.21.0

## Source Code

- **Environment**: https://github.com/bobcowher/gym-homebot-2d

## License

MIT License. See repository for full text.

## Contributors

**Author**: Robert Cowher
