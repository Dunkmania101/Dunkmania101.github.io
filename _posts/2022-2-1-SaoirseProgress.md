---
layout: post
title: Saoirse - My sandbox game project
subtitle: The state of my Saoirse game's development.
categories: Saoirse
tags:
    - Saoirse
---

Hello again! This is post 5 on the development progress of my Saoirse (pronounced "Seer-Sha") game.

Here's a quick summary of what I've done since the last post:

    1. I made the rendering system use Pyglet's batching system for improved performance.

    2. I updated to Pyglet 2.0 for futureproofing and simplicity in rotating models.

    3. I continued work on making all objects provide a json-comptible representation of themselves in any state for saving to disk.

    4. Probably the biggest change was that I removed the BaseCategorizedRegistry class as it was adding complexity and wasn't really needed because the Identifiers support unlimited levels of subdivision, not just two.

I also finally got around to putting this project in a Git repository!
It can be found [here](https://github.com/Dunkmania101/Saoirse) and mirrored [here](https://gitlab.com/dunkmania101/Saoirse).
Since it's being kept in version control now, I won't be posting the giant code snippets in these posts any more unless there's something specific I want to showcase.

