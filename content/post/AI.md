---
title: "Artificial Intelligence"
date: 2024-04-01T23:42:29-08:00
Description: ""
Tags: []
Categories: []
DisableComments: false
draft: false
---



**Intelligence works!!!**
<br>
I’m so excited to start studying AI! I’ve always been fascinated by how machines can learn and make decisions, and I finally got the chance to dive into this field. I can’t wait to explore topics like machine learning, neural networks, and etc. First time when I used Chatgpt (I know it belongs to Natural Language Processing and model, not covered in this course), I was so obessed with AI and machine learning. It is magical to see how a machine can understand and respond to human tasks, or even make decisions based on data. I have more motivation to learn more about AI compared to other courses, like physics or circuit (they are cool and useful though).
<br><br>

I'm so suprised that our course material was similar to UC Berkeley CS188. It is good I think <br>
I'm studying AI by my own, and I'm also planning to take more AI courses in the future. For example, Natural Language Model, AI transformor, etc.
<br>

**Important Topics Covered** :
<br>
**Search Algorithms**: Exploring both uninformed and informed search strategies, such as A* search, to solve complex problems efficiently.
<br>
**Game Playing**: Studying algorithms like minimax and alpha-beta pruning to make optimal decisions in adversarial settings.
<br>
**Markov Decision Processes (MDPs)**: Modeling decision-making scenarios where outcomes are partly random and partly under the control of a decision-maker.
<br>
**Reinforcement Learning**: Learning optimal behaviors through trial and error interactions with an environment, covering methods like Q-learning and policy iteration.
<br>
**Bayes' Nets and Classification**: Understanding probabilistic reasoning, including the representation and inference in Bayesian networks.(applicable in areas like speech and handwriting recognition.)
<br>
**Regression**: Regression predicts continuous values by modeling relationships between variables. Linear regression fits a line, while logistic regression uses probabilities for binary classification.
<br>
**Neural Networks**: Neural networks are interconnected layers of perceptrons that use activation functions to model complex, non-linear relationships. They are trained using backpropagation and are widely used for tasks like image recognition and language processing.
<br>
**Optimization Technique**s: These are methods like gradient descent used to minimize the loss function during neural network training, improving accuracy by adjusting weights efficiently.



<br>
** My some notes **
<br>

** Machine Learning (summarize neural network, Reinforcement..)**

<br>
<iframe 
  src="/images/pics/CMPT310/ML.pdf" 
  width="80%" 
  height="600" 
  style="border:none;">
</iframe>

** what is agent **
<iframe 
  src="/images/pics/CMPT310/agent.pdf" 
  width="80%" 
  height="600" 
  style="border:none;">
</iframe>

** Algorithm **
<iframe 
  src="/images/pics/CMPT310/algo.pdf" 
  width="80%" 
  height="600" 
  style="border:none;">
</iframe>

** Reinforcement Learning **
<iframe 
  src="/images/pics/CMPT310/Reforce.pdf" 
  width="80%" 
  height="600" 
  style="border:none;">
</iframe>

** Bayesian **
<iframe 
  src="/images/pics/CMPT310/pro.pdf" 
  width="80%" 
  height="600" 
  style="border:none;">
</iframe>

** Classification **
<iframe 
  src="/images/pics/CMPT310/classfication.pdf" 
  width="80%" 
  height="600" 
  style="border:none;">
</iframe>


<br><br>
We have projects based on Python.<br>
For example,
we tackle the problem of handwritten digit recognition—converting images of handwritten digits (0–9) into their corresponding numeric values. Specifically, we implement and compare three classifiers:<br>
Naïve Bayes – Uses probability estimates and assumes features are conditionally independent given the class.
<br>
Perceptron – Maintains weight vectors for each class and updates them based on classification errors.
<br>
MIRA (Margin-Infused Relaxed Algorithm) – Similar to Perceptron but adjusts weight vectors with a variable step size to achieve a better margin on each update.
<br><br>
Classifier Training & Tuning<br>
Naïve Bayes: Estimate the probability of each feature being on or off given a digit label. Apply Laplace smoothing and tune the smoothing parameter (k) to improve accuracy.
<br>
Perceptron: Train by iterating over all training examples and adjusting weights whenever the predicted label is incorrect. We can vary the number of training iterations to optimize performance.
<br>
MIRA: An online learner similar to Perceptron but uses a dynamic step size (τ) for weight updates, aiming to achieve a better margin. We also tune a parameter C to cap the update size and pick the best C based on validation accuracy.
<br>
We start with straightforward binary features (on/off pixels).
We then introduce additional features (e.g., counting loops or connected regions) to capture digit-specific patterns, which often boost classification accuracy further.
<br>
Evaluate each classifier on both a validation and test set to measure accuracy.
Compare results to see how tuning parameters (e.g., iterations for Perceptron, C for MIRA, smoothing k for Naïve Bayes) and enhanced features impact performance.
<br>

