---
title: "What I Learned In The Lab From Training Our Three-Finger Hand"
date: 2026-05-01T21:15:00-06:00
Description: "A personal lab retrospective on training our three-finger dexterous hand with Isaac GR00T, and how I had to work through bad data, unreliable sensing, and disappointing early results before the system improved."
Tags: ["Isaac GR00T", "Robotics", "Dexterous Hand", "VLA", "Imitation Learning", "TensorRT", "Embodiment"]
Categories: ["Robotics"]
DisableComments: false
draft: false
---

The first time I watched our policy run on the hand, I felt mostly disappointed.

We had already done the part that sounds impressive on paper. We had an in-house **three-finger, human-like dexterous hand**. We had built a teleoperation pipeline to collect videos of manipulation tasks. We had a second dexterous-hand sensing system recording the angle of each joint. We had converted the data into the format Isaac GR00T expected, fine-tuned the model, and waited for the satisfying moment when the robot would finally do something intelligent.

Instead, the result looked confused. The fingers were inconsistent. Motions that should have been smooth looked hesitant or noisy. The model was not learning the skill we thought we had demonstrated.

As an intern, that was a hard moment. I wanted the project to look further along than it really was. Instead, the hand made it obvious that we were still dealing with a fragile pipeline. In hindsight, that was probably the most useful thing that could have happened to me.

## GR00T Forced Me To Be Honest About The Project

What drew me to Isaac GR00T in the first place was that it does not treat embodiment as an afterthought. A dexterous hand is not just a gripper with extra joints, and the repo is built around that idea. It expects me to define what the robot actually sees, what state it actually has, and what actions it is actually capable of producing.

That fit our situation well, because our hand was not simple enough to fake. I could not collapse it into one open-close scalar and call it done. I had to describe a real system: scene video, wrist video, arm state, hand state, and a high-dimensional action space that reflected how the fingers actually moved.

Using GR00T's `NEW_EMBODIMENT` route mattered to me for a less technical reason too: it forced me to stop pretending the project was closer to finished than it really was. Once I started defining the hand honestly, I could also see the weaknesses more honestly.

## The Hard Part Was Everything Around The Model

At the beginning, I wanted to believe that once I had demonstrations and a strong open-source model, the rest would be incremental. That turned out to be naive.

The actual work was building a data pipeline that deserved to be learned from.

Our setup had two different streams that both mattered:

- teleoperation data and video showing the task
- joint-angle recordings from a separate dexterous-hand sensing system

That sounds reasonable. In practice, it created a fragile pipeline. The project stopped feeling like a clean machine-learning problem and started feeling like a real lab problem. Was the teleoperation smooth enough? Were the sensors stable? Were the joint angles aligned with the video frames? Were some demonstrations technically recorded but still bad enough to poison the dataset?

That is where most of my time really went.

Isaac GR00T was useful here because its dataset structure is strict. Once I started building around its LeRobot-style format and its `meta/modality.json` schema, there was less room to be vague. I had to specify exactly which part of the state vector meant what, which action channels belonged to which joints, and how the observations were synchronized. That discipline was good for the project, even when it was annoying.

Looking back, the repo did not solve our hand for me. What it gave me was a framework that made it harder to hide behind vague progress.

## The First Fine-Tuning Run Was Humbling

When I ran supervised fine-tuning the first time, I expected something imperfect but promising. What I got was worse than that.

The model's behavior was not just underwhelming. It was a sign that something more basic was wrong. The policy did not fail in a subtle, research-paper way. It failed in the blunt way hardware projects often fail: it reflected the messiness of the pipeline that produced it.

That was frustrating, because the natural instinct is to blame the model. It is easy to say the network is not strong enough, or the architecture is wrong, or the open-source checkpoint does not transfer well enough. But the more time I spent with the data, the harder it became to keep telling myself that story.

The real problems were closer to the bench:

- demonstration quality was inconsistent
- sensor readings were noisier than I wanted
- some timing alignment between video and finger state was not good enough
- some episodes were "recorded" but not actually clean training examples

That was a difficult point in the project, mostly because it forced me to admit that we had built a pipeline that looked more mature than it actually was. The model was not the only thing being evaluated. Our whole collection stack was being evaluated too, and early on, it did not pass.

## Things Started Improving When I Worked On The Boring Problems

The turning point was not a clever modeling trick. It was deciding to fix the boring parts properly.

We changed the data pipeline. We changed the sensing hardware. We became stricter about what counted as a usable demonstration. We paid more attention to synchronization, calibration, and consistency instead of assuming the model would average out the noise.

That was not glamorous work. A lot of it was repetitive, slow, and honestly a little thankless in the moment. But it changed the project.

Once the data got cleaner and the sensor side became more reliable, the behavior of the same GR00T-based fine-tuning setup improved in a way that was immediately visible. The hand looked less confused. Finger motion became more coherent. The model started reflecting the task structure we had been trying to teach it all along.

That was one of the most satisfying parts of the whole experience, because the improvement was not abstract. It felt earned. We were not watching a lucky run. We were watching a system get better because I had spent enough time fixing the foundation under it.

## GR00T Became The Stable Part Of The Project

I still think the choice of model mattered. Starting from `nvidia/GR00T-N1.7-3B` gave me a strong policy backbone, and the custom-embodiment path was exactly what I needed for a hand that did not fit the stock robot examples. I used the repo's supervised fine-tuning workflow, kept the action horizon in the short chunked style GR00T expects, and leaned on its evaluation and policy interface.

But the more time I spent with the project, the more I saw GR00T as the stable center of the stack, not the entire stack.

The open-source part gave me:

- a strong starting policy
- a clean way to define our custom hand as a new embodiment
- a disciplined dataset format
- training and evaluation tools
- a deployment path through a policy API and server-client setup

What it did not give me, and what I had to fight for myself, was the part that actually made the project work in the lab:

- good teleoperation habits
- reliable sensing
- synchronization I could trust
- standards for rejecting bad demonstrations
- safety and control logic around the learned policy

That split became clearer to me over time. The open-source model was not failing me. It was exposing where our own pipeline was weak.

## I Learned The Difference Between "Interesting" And "Working"

One thing I did not appreciate at the beginning was how different it feels to have a system that is merely interesting versus one that is actually starting to work.

An interesting system gives you demos, plots, and reasons to keep going. A working system starts becoming predictable. It does not surprise you for the wrong reasons as often. When it fails, the failure is narrower. When it succeeds, the success feels repeatable instead of accidental.

That is where I felt the project change.

At first, the hand plus GR00T combination was intellectually exciting but operationally shaky. After we improved the data quality and sensing hardware, it became something more serious. Not finished, not perfect, but believable. I could finally see the shape of a real learning pipeline instead of a fragile prototype held together by optimism.

## What I Took Away From The Project

Before this project, I might have described dexterous-hand learning as a model problem. After going through it, I do not think that is true.

For me, the real lesson was that a dexterous-hand project succeeds or fails on whether the whole pipeline deserves the sophistication of the model sitting on top of it. If the demonstrations are weak, the sensing is noisy, or the synchronization is sloppy, then a powerful model just learns those problems more efficiently.

That is why this project stayed with me. The satisfying part was not simply that Isaac GR00T eventually gave us a better policy. The satisfying part was that I had to grow the rest of the system until the model had a fair chance to succeed.

In the end, I did not feel like I had "plugged an open-source model into a robot." I felt like I had gone through the much more useful experience of learning what it takes to make a dexterous hand trainable at all.

That was a harder lesson than I expected. It was also a better one. As an intern, that mattered to me more than a polished demo. I got to see what real progress looks like when it is slow, frustrating, and tied to dozens of small fixes that no one would notice from the outside.
