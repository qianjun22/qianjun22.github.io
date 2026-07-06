---
title: How a Robot Actually Works
description: The robot software stack, from Linux to a learned brain, followed through one forward command.
date: 2026-07-05
tags:
  - robotics
  - ai
---

*One forward command, followed all the way down from "walk" to current in a motor.*

A Unitree G1 can walk using a neural network that reads 47 numbers and emits 12. That's the whole policy — at least in Unitree's own reinforcement-learning example. Every 20 milliseconds it gets the body's angular velocity, which way gravity is pointing, the joint angles and velocities, the command from the joystick, and a few bookkeeping signals. Out come 12 target leg angles. Something much faster turns those targets into motor torque, and somehow the ~35 kg machine stays upright.

"Somehow" is carrying a lot of weight in that sentence. The first time I went looking through a robot's software, I made the obvious mistake: I searched for the brain — the one program where pixels go in and intelligent motion comes out. There is no such program. There's a pile of loops running at wildly different speeds. The motor servo cares about the next fraction of a millisecond. The walking policy cares about the next footstep. The task planner cares about the far side of the room. And if the planner wanders off to think for a moment, the servo still has to keep the robot off the floor.

That timing hierarchy *is* the robot. Linux, ROS, GPUs, planners, policies, simulators — all of it is implementation detail hung around one stubborn fact: **gravity does not wait for software.** This is also why robot infrastructure is hard: you are not deploying one model, you are deploying a *timing-critical stack*, where every layer runs at a different clock and fails in a different way. So let's follow a single "walk forward" command down the stack and watch each layer earn its place. The best way to understand the machine is to ask, at every step: what comes in, what goes out, how often, and what happens if this part is late.

Here's the whole descent on one page — one command, top to bottom, getting faster as it falls:

```
"walk forward"
   │
   ▼  Intent / task planner        ~1 Hz     where in the room
   ▼  Footstep planner             ~20 Hz    where the feet go
   ▼  Walking policy (neural net)  ~50 Hz    47 numbers in → 12 leg targets
   ▼  Joint controller (PD)        ~200 Hz+  desired vs actual angle → effort
   ▼  Motor servo / current loop   ~1 kHz+   the loop that must never be late
   ▼
 gravity  (does not wait)
```

*Each arrow down loses information and speeds up. The slow layers are allowed to think; the fast layers at the bottom are not allowed to be late.*

---

## The robot cannot wait for its brain

Start with the numbers, because the numbers force the whole architecture.

A balancing controller runs at something like 1 kHz. That's one millisecond per cycle to read the sensors, decide, and command the motors — every millisecond, not on average. The walking policy runs far slower, around 50 Hz, so a fresh set of leg targets shows up only every 20 ms. In the gap between two policy updates, the fast loop runs about twenty times on its own, holding the last set of targets. And if a high-level model upstairs pauses for 200 ms to reason about a task, the inner loop just executed 200 times without it. It had better have something safe to do in the meantime.

This is why "real-time" in robotics doesn't mean *fast* — it means *predictable*. What kills a balancing robot isn't a slow average; it's an unbounded worst case, the one cycle in ten thousand that arrives 40 ms late because the operating system decided to do something else. So the fast loop needs a *bounded, measured* latency, and that's a property of the whole system — the kernel, the drivers, the memory behavior, the way work is pinned to cores — not something you get from one magic setting. The `PREEMPT_RT` patch makes Linux far more preemptible and can pull scheduling latency down enough for many control loops; a dedicated microcontroller running a small real-time OS next to the motors gets you there another way. Either path, the point is the same: carve out a lane where interruptions are tightly bounded, and let everything slower live outside it.

That single split — a protected fast lane, everything else outside — is the seed the rest of the stack grows from.

---

## Before it can act, it has to know where it is

Here's the part the block diagrams quietly skip. The robot does not actually know its own pose. Nothing on board measures "I am tilted 4 degrees forward and drifting left at 0.2 m/s." The IMU measures accelerations and rotation rates, drifting over time. The encoders measure joint angles, not where the body is in the world. The foot sensors say *maybe* that foot is loaded. These signals disagree, arrive at different times, and live in different coordinate frames.

Turning that mess into one trusted estimate of *where am I, how fast, at what angle, which feet are down* is **state estimation**, and it's arguably the hardest bridge in the whole machine. It's the reason a robot can look confident and be one bad estimate away from the floor. When people say "bad state estimation is the silent killer of locomotion," this is what they mean: the controller is only ever as good as the reality it's handed, and if the estimate says the foot is planted when it's actually sliding, the controller will confidently do the wrong thing at 1 kHz.

So the real chain isn't *sensors → brain → motors*. It's sensors → **estimator** → decision → controller → motors, and the estimator is where noisy fiction becomes something you can act on.

---

## A plan is not a reflex

Now the "walk forward" command finally does some work. On the simplest learned humanoids, a bare velocity command like this goes straight into the walking policy, which quietly folds planning and control together. But ask the robot to cross a cluttered room and you need an explicit **planner**, which turns the wish into a timed proposal: put the left foot *here* at 1.2 seconds, shift the body *there*, then swing the right. The operative word is *proposal*. The world has not agreed to any of it.

On the first step the foot lands two centimeters short. Maybe the carpet gave. Maybe the estimate was stale. Maybe the simulator that trained the policy lied a little about friction. The planner can fix the *next* step, but it's far too slow to save this one — at ~20 Hz it gets a new thought every 50 ms, and a balancing robot can become a falling robot well inside that window.

That gap is the entire reason the **controller** exists. While the planner is still admiring its trajectory, the controller is asking a much more boring question, over and over: *where did the joints actually go?* It compares the measured state to the reference and drives the error toward zero. On Unitree's open G1 example, the learned policy emits 12 desired leg angles at 50 Hz, and a faster low-level PD loop turns the gap between desired and actual angle into motor effort in between. Some controllers command torque directly; many, like this one, hand position or velocity targets to an even faster motor servo that closes the current loop. There's almost always another, faster loop hiding beneath the one you're looking at.

The driving analogy mostly works: the planner draws the racing line, the controller keeps the car on it. But add one correction — the road is moving, the tires are imperfect gearboxes, and the car only knows where it is by guessing from noisy sensors. **The planner owns intention. The controller owns error.**

Between them sits the middleware, and this is where **ROS 2** shows up — not because a stack diagram has a box for it, but because two programs written by two people need to exchange a stream of typed messages without being welded together. Each logical participant is a *node*, and several can share one OS process; one node publishes to a named channel, another subscribes, and neither knows the other exists. That decoupling is genuinely useful: you can swap a simulated sensor for a real one by changing a driver, and you can record every message and replay a fall that happened once, last Tuesday. But the decoupling isn't free — it costs serialization, queues, quality-of-service choices, and scheduling — which is exactly why the innermost loop usually doesn't ride a casual publish/subscribe path. (You *can* run real-time control through ROS with a carefully engineered `ros2_control` process; you just don't get determinism for free by publishing a topic faster.) The useful rule is simpler than "ROS vs no ROS": the high-level software is allowed to be occasionally late, and the stabilizing loop is not.

And when the planner does go quiet — a crash, a hang, a model that stalls — the controller must not heroically replay "walk forward" forever on a stale command. It needs a watchdog, a timeout, and a boring fallback: hold, sit, or stop. Intelligence proposes; the fast loop enforces; a separate safety layer decides when both have failed.

![[blog-signal-path.png]]
*One forward command, top to bottom. Each arrow loses information and changes clock speed; the label under each layer is what breaks if it stalls.*

---

## Why so many new gaits are learned in simulation

I said the G1 walks with a neural network, and it's worth being careful about what that does and doesn't mean. Engineers did not stop writing control software. Plenty of modern locomotion is still model-based — trajectory optimization, model-predictive control — and even the learned kind is usually wrapped in engineered estimators, filters, constraints, and safety logic. What a growing fraction of new legged robots stopped doing is *writing every footstep by hand*. The gait itself — the reflexive, contact-rich business of not falling — is increasingly learned, often inside a hybrid controller.

And the interesting part isn't the algorithm; it's the reward. You don't train a policy by describing a good gait. You describe what you *want* — stay upright, track the commanded velocity, move smoothly, don't cook the motors — and let the policy discover how, by trial and error across thousands of simulated robots at once. Which means the reward function is the real program, and like any program it has bugs. Reward forward speed too heavily and the simulator gleefully discovers a controlled fall that happens to move forward fast. Punish energy too hard and it learns the safest gait of all: standing perfectly still. Most of the craft is in shaping the reward until the behavior you get is the behavior you meant.

Then you *freeze* the network and ship it. This is the part that surprises people: at runtime there is no learning happening on the robot — just a forward pass through a fixed network, 50 times a second, with that fast PD loop doing the muscular work underneath. The training was expensive and happened once, offboard; the deployed policy is cheap and never changes its mind. (Research systems do adapt on hardware, but shipping products almost never touch the weights.)

![[blog-rl.png]]
*Two very different worlds. Left: training — thousands of parallel sims, a reward, an optimizer, running for hours to days. Right: deployment — one frozen network, one robot, a single forward pass every 20 ms.*

Which raises the obvious problem: the policy learned in a simulator, and the simulator is wrong. The simulated ankle has no cable stretch; the real one does. The simulated friction is a clean number; the real floor is not. Train against one perfect physics and the policy overfits to a world that doesn't exist. The fix is to make the simulator *unreliable on purpose* — randomize the mass, the friction, the sensor latency, shove the robot mid-stride — so the policy can't lean on any single quirk and instead learns something that survives a whole neighborhood of realities. You still can't skip the real machine: gains get tuned on hardware, actuators get characterized, trials go through rigs and tethers first. The simulator is precisely the thing you don't fully trust, which is why domain randomization exists at all.

There's a quieter way to win this fight, though, and it's easy to miss because it happens before any training. You can shrink the gap in the *hardware*. Berkeley's open-source humanoid was deliberately built for low simulation complexity — backdrivable actuators and clean joint dynamics that a simulator can treat as almost ideal — and the authors argue this let them transfer a minimal RL policy with only *light*, carefully targeted randomization: narrow for the identified hardware parameters, while still randomizing contact conditions like ground friction broadly. That's the deeper point: the sim-to-real gap isn't only a training problem you paper over with randomization. It's partly a design decision about how faithfully your robot can be simulated in the first place — the mirror image of a rule every roboticist learns the hard way: bad hardware caps what any software can do, and *good* hardware quietly hands the software an easier problem.

---

## How much of the robot do you actually own?

The neatest way to make all of this concrete is to put two robots side by side — not by spec sheet, but by a single question: **when the robot falls, whose problem is it?**

![[g1-official.jpg]]
*Unitree G1. Image: Unitree Robotics.*

The **Unitree G1** hands you the stack. The EDU variant gives you low-level access — you can command joints directly, run your own policy on an optional onboard GPU module, train it in Unitree's simulator, and deploy it. That's real power, and it comes with real liability: the moment you're writing the controller, the fall is yours to prevent. Everything above — the 23-to-43 joints, the LiDAR and depth camera, the dual encoders on every actuator — is supporting detail. The story is that the vendor boundary is drawn low, and you own most of what's beneath it.

![[spot-official.jpg]]
*Boston Dynamics Spot. Image: Boston Dynamics.*

**Boston Dynamics Spot** draws the boundary in the opposite place. You say "walk there," and Spot handles the balance, the stairs, the recovery from a slip — its production locomotion is a proprietary hybrid of reinforcement learning and model-predictive control, and you don't touch it. What you get is a contract: a clean high-level interface, and a reliable behavior behind it. There's a separately licensed joint-level control API that streams low-level commands at high rate (hundreds of Hz) for teams that qualify, but the default posture is unmistakable — **Boston Dynamics keeps the falling problem and sells you the walking.** From the outside both robots expose the same broad hierarchy, high-level intent over a fast joint loop; the difference is how far down the vendor lets you reach, and therefore who's responsible when physics wins.

The rest of the field spreads along that same axis. At the open end sit Unitree's larger H1 and research builds like the open-source Berkeley Humanoid. In the middle, Agility's Digit is already pulling real warehouse shifts behind a developer API — a humanoid Spot. And at the sealed extreme are Figure, Apptronik's Apollo, Boston Dynamics' electric Atlas, and Tesla's Optimus: stunning demos, no published specs, no SDK, no way in. The pattern is almost a law: the more a robot promises to just work, the less of it you're allowed to touch.

---

## The boxes are getting blurrier, not disappearing

The stack I've described is modular — estimator, planner, controller, each a box with an interface — and the frontier direction is to blur those boxes into a single learned policy that maps camera pixels and a language instruction more or less straight to action. Vision-language-action models like OpenVLA and NVIDIA's GR00T do exactly this: they produce sensorimotor actions, not just high-level plans, trained on a mixture of real robot demonstrations, cross-embodiment datasets, and (for the video part) human footage. It's genuinely promising and genuinely not solved — reliable, long-horizon, cross-robot autonomy is still ahead of us, and even an end-to-end policy still emits something a lower-level controller has to catch: a joint target, a chunk of end-effector deltas, a trajectory, at some rate, wrapped in the same safety layer as before.

The most mature version of this idea isn't a robot at all. Tesla said its FSD v12 replaced the hand-written city-streets driving stack with a network trained end-to-end on millions of fleet video clips — and Musk claimed it cut hundreds of thousands of lines of C++ dramatically in the process. But "end-to-end" means end-to-end *trained*, not one blob that swallowed everything: the learned policy still drives low-level actuation inside a larger vehicle-control and safety system. The one difference that matters for us is timing — a car that stutters for a moment keeps rolling; a walking humanoid falls. Same direction; crueler deadline on legs.

That's the thing to hold onto. Even if the top of the stack collapses into one model, the bottom doesn't go anywhere. Gravity still doesn't wait; something still has to run at 1 kHz; the learned brain still runs *on* a real-time controller, on a computer, on the machinery that keeps the robot upright while it thinks. The models get smarter. The clocks stay the same.

---

## The bigger bet: learning the simulator itself

We hit two walls back in the training section. The simulator is wrong, so policies overfit to a physics that doesn't exist. And robot data is scarce — there's no internet-sized corpus of robot actions the way there is of text and images. Those are really the same wall: not enough honest experience to learn from.

A **world model** is the audacious way around it. Instead of hand-writing a physics engine and forever patching its errors, you *learn* one — a network that takes the current state and a proposed action and predicts what happens next. Train it on real interaction and it captures the messy, contact-rich dynamics no hand-built engine ever nails. Train it on video — which the internet has in near-infinite supply — and you get visual and behavioral priors that scarce robot data can't supply on its own. That is the dream nearly every world-model startup is chasing, and why the money is pouring in: a learned, general model of reality you can train policies *inside* — one that could *shrink* the sim-to-real gap, because it's learned from real data instead of authored by hand. It's the instinct behind Google's Genie, NVIDIA's Cosmos, and driving world models like Wayve's GAIA: learn the world instead of authoring it. The same model does double duty, too — a robot can run it forward in its head, imagine a few candidate moves, and pick the one that doesn't end with it on the floor before committing. The honest asterisk: a learned simulator still has its own errors, its blind spots for actions it never saw, and its drift. It moves the gap; it doesn't abolish it — and you still need real robot-action data for the contact-rich parts.

It is a genuinely big idea, and it is not solved. The catch is compounding error: every imagined step is slightly wrong, the errors stack, and the farther the model dreams the less you should trust it — especially at contact, where the physics is hardest and matters most. Error grows with the horizon, and a rollout can look visually convincing while being physically wrong. But you can see the prize: one learned artifact that attacks the wrong simulator, the sim-to-real gap, and the data drought at the same time. That's a bet worth a lot of money.

---

The software is the part that fits in a blog post, which makes it easy to forget it's a fraction of the actual problem. A demo works once; a product works ten thousand times unattended, and most of that reliability is in the parts no diagram flatters — the actuators that set the ceiling on everything above them, the state estimator, the scarce robot data, the safety case, the unglamorous integration into a real building. You can't software your way past a bad motor.

That's the real lesson, and it's the one the field keeps relearning: **reliable robots don't come from a smarter model alone — they come from the stack around it.** The runtime that guarantees the fast loop's deadline, the safety layer that decides when to stop, the deployment and observability that let you see *why* it fell last Tuesday, the simulator it grew up in. The model is necessary; it is nowhere near sufficient. Whoever wants dependable robots has to own that whole timing-critical stack, not just the network at the top of it.

But if you only remember one thing, make it the shape of the machine: a fast loop at the core that cannot be late, slower and smarter loops stacked outside it, and a hard physical deadline waiting at every boundary. Forty-seven numbers in, twelve out, fifty times a second — and gravity does the rest.

---

## Further reading

- **[ROS 2 documentation](https://docs.ros.org/)** — nodes, topics, and the real-time story, straight from the source (and free).
- **[*Modern Robotics: Mechanics, Planning, and Control*](https://hades.mech.northwestern.edu/images/7/7f/MR.pdf)** (Lynch & Park) — the planner/controller foundations, with a free PDF and video lectures.
- **[Humanoid Locomotion and Manipulation: a survey](https://arxiv.org/abs/2501.02116)** — the model-based-vs-learning landscape, without hype.
- **[Berkeley Humanoid: A Research Platform for Learning-based Control](https://arxiv.org/abs/2407.21781)** — a lovely open example of learned locomotion, and a case study in shrinking the sim-to-real gap by design rather than by brute-force randomization.
- **[OpenVLA](https://arxiv.org/abs/2406.09246)** — a concrete, readable example of a vision-language-action policy and how it's actually trained.

## Image credits

- **Diagrams** — original, created for this post. © Jun Qian.
- **Unitree G1** — © Unitree Robotics ([unitree.com/g1](https://www.unitree.com/g1/)).
- **Boston Dynamics Spot** — © Boston Dynamics ([bostondynamics.com/products/spot](https://bostondynamics.com/products/spot/)).
