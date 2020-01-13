---
layout: post
title: "Upside down reinforcement learning"
date:   2020-01-11 00:00:00
---

Summary
-------

How to make an upside down RL. I used  `transform: rotateX(180deg)`.
This has the downside of only working in a div. But it's still something...

   <div style="transform: rotateX(180deg)">
       RL
   </div>

But moving on, in this post I'm going to take you through building a simple example of reinforcement using *upside down RL*.
Upside down reinforcement learning is detailed in [this report](https://arxiv.org/abs/1912.02877), and I found [this paper](https://arxiv.org/abs/1912.02877) to be useful in implementation.  The broad outline is contained in the title of the original report: *Don't Predict Rewards -- Just Map Them to Actions*.

 - Typically, most systems predict the returns of actions, and execution proceeds by picking the best action
 - In upside down RL, the system predicts reads in a state and a **command** and predicts the action to take.

I've trained an example of upside down RL on the very easy cartpole problem.

<video controls>
  <source src="https://gitlab.com/ririw/upside-down-rl/raw/master/cartpole.webm" type="video/webm" />
</video>

Running it yourself
-------------------
If you'd like to run it *right now*, all you need to do is:

 1. Install via pip: `pip install upside-down-rl`
 2. Run via python: `python -m "upside_down_rl.cartpole" --render`
 3. Run tensorboard to monitor the output: `tensorboard --logdir=experiments serve`. This will show the performance of the model improving over time.

You should see a window with the cartpole, and you can watch as it slowly improves.


Alternatively, you can run using `docker`, as follows:

 1. Either:
  - Clone the repo and cd into the deploy directory, or
  - Download the [docker compose file](https://gitlab.com/ririw/upside-down-rl/blob/master/deploy/docker-compose.yml) and put it into a directory somewhere
 2. Run `MINIO_ACCESS_KEY=somethingyoumakeup MINIO_SECRET_KEY=somesecretkeyyoulike docker-compose up`

This will run 25 runs of the cartpole tool. Model params are saved to minio (an S3 compatible data store), and you can access them by going to [http://localhost:80/](http://localhost:80). The runs will also appear in tensorflow, accessible at [http://localhost:6006](http://localhost:6006)


Approach
--------

In general, we want to train an algorithm to take a state and a command and tell us the action to take that would achieve that state and command. 
A command is of the form "Achieve reward *x* in *n* steps".
To do this, we train a supervised algorithm to predict the action that will (eventually) lead to the desired command and state. 
To gather commands, states and actions, we use previous play episodes to build a dataset of commands and states, and train a model to predict the actions taken in those runs.
Finally, when running the model to gather data, we send commands that ask for progressively higher and higher rewards.

Overall algorithm
=================

The core algorithm, as I've applied it is as follows:


 1. Play the game with random actions many times to initialise the system. Save each play into two buffers:
   - One is a FIFO buffer, that holds the most recent plays.
   - One is a priority queue, that keeps the best plays. Note that plays are compared first on total reward, then on length of play, in both cases, higher is better.
 2. For as many epochs as necessary:
    1. Train a classifier to map from inputs and command to action, as described below
    2. Execute plays using the classifier to pick actions based on the output of the classifier
    3. Collect executed plays in the same FIFO and priority queue as described in step one

The classifier takes two inputs:

 - The state, as reported by the game being played
 - A "command", comprised of:
   - The reward we desire
   - And the number of steps to achieve it

The classifier then suggests the action that would achieve that command, given the state.

Training algorithm
==================

This generates training data, from previous episodes in the FIFO and the priority queue. Note that an episode is a start to end play of the game in question, so it has multiple states and multiple rewards accumulated over the play. 

 1. For all episodes in each of the buffers
    1. Sample a starting time between the start and end of the episode
    2. Sample an ending time. The ending time is:
      - With 75% probability, set to the end of the episode
      - With 25% probability, set to a random time between the start time and the end time
    3. Generate the command:
      - the Δt is end time minus start time
      - the reward is the sum of all rewards accumulated between the start time and end time
    4. Gather the state input, the game state at the sampled start time
2. Train the model based on these X and Y values. My model as as follows
   - A state embedding:
     - A Linear layer, from state size (4) to a vector of length 64
     - A PReLU layer
   - A command embedding:
     - A Linear layer, from command size (2) to a vector of length 64
     - A PReLU layer 
   - Then, I take the elementwise product of state and command embeddings 
   - And feed it through a network with structure:
     - Linear 64 → 64
     - PReLU
     - Linear 64 → num actions (2)
   - Models were trained with Adam, for up to 100 epoch, with early stopping, patience 10.
   - Loss was crossentropy loss.

Play algorithm
==============

When playing, the algorithm is simply:

 1. Sample a desired reward and horizon based on the top episodes in the priority queue:
   - desired horizon is the average of the lengths of top episodes
   - desired reward is between the average and the average plus one standard deviation
 2. While the game is not finished:
   - Get an action from the model, by feeding in game state, desired horizon and desired reward 
   - Enact the action and receive reward
   - Decrement desired horizon by one and reward by the reward from the last step. Clip at zero.
 3. Save each full episode into the FIFO and priority queue.


Hyperparameters
===============

For cartpole, my hyperparameters were as follows:

 - Both the FIFO and queue had max size 200
 - Each run is 100 episodes
 - I gather 500 random plays at the beginning, before the first training
 - To calculate my desired reward and horizon, I use the top 100 plays in the queue

Code tour
---------

Trying new environments
=======================

To try a new environment, I suggest the following steps:

 0. Clone the code
 1. Get [poetry](https://python-poetry.org/) and install, then run `poetry install` in the codebase directory.
 2. copy `cartpole.py` to a new file
 3. Modify your new file in two ways:
   1. Return a different environment in the `cartpole` function. Consider renaming this function.
   2. Modify the `CartPoleTorchModel` to match your chosen environment
 4. Run the code, and see how it all goes
 5. Tweak the hyperparameters to improve performance.
 

Implementation details in `udrl.py`
===================================

 - The `ReplayBuffer` class tracks episodes
 - The `TrainItem` class represents an item passed to the model to train on
 - The `ModelInterface` is the interface a model must support for the `udrl` code to use it:
   - `train` takes a list of `TrainItems` and trains the model, and returns nothing
   - `run` runs the model, this should return something that can be fed into the environment
   - `random_action` should sample a random action
 - The `URLDConfig` class configures the tool
 - An `Episode` represents a full play of the game in question, including 
   - Actions
   - Observations
   - Rewards
   - The total reward for the episode
   - Ident: a random integer between 0 and 999, useful for doing train-test splits
  - `_run_episode` will run a single episode
  - `_run_multi_episodes` will run many episodes
  - `_par_run_episodes` will run many episodes in parallel
  - `_random_fill` will do step 1 of the overall algorithm, using the `random_action` function
  - `_run_model` will do step 4 of the overall algorithm (running the model and tracking episodes)
   - If configured, this function will write out to tensorboard. Simply pass a `tensorboardX.SummaryWriter` to the `UDRLConfig.writer`
   - If configured, this function will render the environment. Simply set `render` to true in `UDRLConfig.render`
  - `_train_model` will build a training dataset and pass it to the `train` function of the model
  - `_save_model` will call the function `save_model` if provided in the `URLDConfig`
  - `train_udrl` the main loop. Simply provide:
   - A function that returns an OpenAI gym compatible environment. This needs to be a factory, rather than an instance, so that multiple instances can be spawned for parallel operation
   - A `UDRLConfig` instance

Discussion
----------

I found this approach very easy to implement, which was quite satisfying. The main advantages I see are:
 
 - Simplicity: this is a reasonably simple approach to reinforcement learning
 - Directness: this seems like a very straightforward algorithm, rather than having multiple interacting systems as in other approaches
 - The command is quite interesting: the idea of commands seems like it would provide a lot of ways to tune and interact with the system.

Downsides:
 
 - Hyperparameters: there are quite a few hyperparameters, and they significantly affect the results.
 - I wasn't able to get the lunar lander gym experiment working. I'm going to look at the hyperparams noted in the paper and see how well they work.

Regardless, it was neat to set the whole thing up, and I plan on playing around more with the command encoding and looking for alternative algorithms that require intrinsically fewer hyper parameters. 

