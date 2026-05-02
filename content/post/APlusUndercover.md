---
title: "Software Engineering: Building Arcade Game"
date: 2025-04-29T10:35:00-07:00
Description: "A retrospective on our Software Engineering Java game project, from early design documents to pathfinding, testing, and refactoring."
Tags: ["Software Engineering", "Java", "Game Development", "Testing", "CMPT276"]
Categories: ["Software Engineering"]
DisableComments: false
draft: false
---

In Software Engineering course, my team and I built **A+ Undercover**, a small 2D Java game with a very unserious premise and a very real software engineering story behind it. The player is a student trying to survive an exam hall: collect smartphones to get answers, avoid TAs, dodge fake answer sheets, grab bonus professor answers when they appear, and make it to the exit before time runs out.

What makes the project interesting to me now is not just the game itself. It is the way the repository preserves the entire arc of the project: the original design documents, the halfway uncertainty, the architectural reset, the heavy testing push, and the late-stage refactoring that made the final version feel much more disciplined than the first plan.

## The Idea Started Simple

The earliest design documents describe a straightforward arcade maze game on a **20x20 grid**. The rules were already there: phones were worth points, fake answer sheets were penalties, TAs were moving enemies, and the goal was to submit the exam without getting caught. The original use cases assumed **arrow-key controls**, a visible pause button, and difficulty changes through the **number of TAs** on the map.

That first version reads like many student project plans: clear enough to get moving, but still too optimistic about how smoothly the pieces would fit together.

## The Mid-Project Reset Was The Real Turning Point

The Phase 2 report is the most honest document in the whole folder. The team explicitly says that the first implementation did not behave the way they expected, even though it was following the early UML fairly closely. Instead of forcing the bad structure to survive, they reset and rebuilt around a more practical 2D Java game architecture, borrowing ideas from RyiSnow's Java game tutorial and then extending them with their own UI and gameplay changes.

That decision shows up everywhere in the codebase. The final game no longer feels like a direct translation of the UML. It feels like a project that learned what it actually needed:

- a central `GameBoard` to manage the loop, rendering, and state transitions
- a dedicated `KeyHandler` for input across title, play, option, win, and game-over states
- a `TileManager` for loading and drawing the map efficiently
- `Entity`, `Player`, and `Enemy_TA` classes that separate shared movement logic from player- and enemy-specific behavior
- a `CollisionChecker` that became its own subsystem instead of being scattered across unrelated classes
- a `PathFinder` and `Node` pair to support TA pursuit with A* search

That is a much better architecture than the one the project started with, and the repository makes the transition visible instead of hiding it.

## What The Final Game Actually Ships

By the end, **A+ Undercover** had become more than a classroom prototype.

The player moves through a **50x50 tile world**, centered on screen while the map scrolls underneath. Three smartphones are required for the win condition. Fake answer sheets subtract **30 points**. A professor answer bonus adds **8 points** and disappears over time. TAs chase the player using pathfinding rather than random motion. The score is capped at **100**, the timer is set to **two minutes**, and the win condition requires both enough phones and reaching the door tile.

A few design decisions also became sharper during development:

- difficulty no longer changes the number of TAs; it changes their **speed**
- controls moved from arrow keys to **WASD**
- pausing moved from an on-screen button to the **Escape** key
- objects ended up being placed on valid walkable tiles rather than fully fixed positions

Those changes matter because they show the team adjusting the game for what was actually playable, not what looked clean on a requirements sheet.

## My Favorite Part Is The Engineering Underneath

The most interesting technical detail in the project is the TA behavior. The `PathFinder` class implements A* over a tile grid, calculating `gCost`, `hCost`, and `fCost` values and building a path for enemies to follow. That choice gives the TAs a purposeful, exam-proctor feel. They do not just wander. They hunt.

The surrounding systems are also more thoughtful than I expected from a short student project:

- `AssetSetter` randomizes reward placement on valid empty tiles
- `TileManager` only draws tiles inside the visible window
- `UI` handles multiple screens and keeps the game visually coherent
- `CollisionChecker` was later cleaned up to rely on a `Direction` enum and helper rectangles rather than repeated string-heavy logic
- duplicate behavior between `Player` and `Entity` was pulled upward during refactoring

The code is not perfect, but it shows a team moving from "make it work" toward "make it maintainable."

## The Testing Phase Was Not Cosmetic

The final report documents a testing push that was much more serious than I expected. The repository contains **209 JUnit tests**, and the Phase 3 report lists very high coverage across packages:

- `app`: 97% line coverage, 91% branch coverage
- `entity`: 98% line coverage, 88% branch coverage
- `tile`: 98% line coverage, 90% branch coverage
- `object`: 99% line coverage, 88% branch coverage
- `ai`: 98% line coverage, 93% branch coverage

More importantly, the tests are not all shallow getter/setter checks. They probe pathfinding tie-breaking, collision edge cases, UI rendering through `BufferedImage` graphics contexts, score caps, win conditions, invalid directions, and fake test doubles for controlled scenarios. The commit history in late March also makes the focus obvious: test folders appear, Javadocs expand, duplicate methods are removed, image-loading helpers are consolidated, and magic-string direction handling gets cleaned up.

There is even a separate code-review document that calls out concrete smells like duplicate `draw()` and `setup()` methods, magic numbers in the UI, inconsistent bonus scoring, and overgrown screen-rendering methods. That kind of review work usually gets skipped in small student games, so I was glad to see it preserved here.

This is the part that made the project feel like software engineering rather than just game programming.

## The Repo Also Preserves The Messy Reality

One reason I like reading this folder is that it does not pretend the work was linear. The halfway report openly says the team was still struggling to make the main loop and integration work. The Phase 2 report admits they had to overhaul the first version. The later commits show code smell cleanup around naming, constants, and collision logic. Even now, running the test suite locally is a reminder that GUI-heavy Java projects are hard to automate cleanly across environments.

That is not a weakness in the story. It is the story.

Good student projects are rarely polished from day one. They get better because the team learns when to throw away a weak idea, when to refactor, and when to stop treating documentation as sacred if the implementation has taught you something more useful.

## What I Took Away From It

Looking back, **A+ Undercover** is a small project with a surprisingly complete engineering arc:

- start with a simple idea and rough requirements
- discover that the first architecture is not strong enough
- rebuild around clearer responsibilities
- add features only after the loop and state management are stable
- test aggressively
- refactor the pieces that are obviously fighting you

If I continued this project, I would probably focus next on packaging, stronger separation between Swing startup and test code, and cleaner repository boundaries between source, generated artifacts, and documentation. But as a CMPT 276 project, it already succeeds at something more important: it shows a team learning how real software evolves under pressure.

That is why I ended up appreciating this folder so much. The game is funny, the theme is memorable, and the code tells the truth about how the project got there.

**{Related Courses}**: CMPT 276 Software Engineering, Object-Oriented Programming, Software Testing
