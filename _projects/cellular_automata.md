---
layout: page
title: Cellular Automata
description:  Object-oriented Python implementations of cellular automata from scratch.
img:
redirect:
importance: 3
category: ""
related_publications: false
---

<!-- GitHub Repository -->
<p>
  <a href="https://github.com/mattrflew/cellular-automata" target="_blank" rel="noopener">
    <i class="fa-brands fa-github fa-2x"></i>
    &nbsp; View Code on GitHub
  </a>
</p>

<!-- Overview -->

Cellular automata are discrete mathematical models in which each cell in a grid evolves according to a simple set of local rules. Despite their simplicity, these rules can generate surprisingly complex behaviour. The [Wikipedia page](https://en.wikipedia.org/wiki/Cellular_automaton) is a fun read with interesting examples.

This project is a collection of some cellular automata simulations I implemented from scratch in Python using an object-oriented approach. While there are some genuine applications of cellular automata, this was just a fun exercise to make some cool patterns. Refresh the page to reload the gifs, or see them directly on GitHub.

# Conway's Game of Life
Implementation of the classic [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life). Cells live, die, and reproduce based on the state of their neighbours and produces endless fascinating patterns.

<div class="text-center">
  {% include figure.liquid
     path="assets/projects/cellular_automata/game_of_life.gif"
     width="50%"
     class="img-fluid rounded z-depth-1"
     zoomable=true %}
</div>


# Waves

A simulation where each cell distributes its intensity stochastically to neighbouring cells with wrap-around boundaries, producing ripple like patterns.

<div class="text-center">
  {% include figure.liquid
     path="assets/projects/cellular_automata/waves.gif"
     width="65%"
     class="img-fluid rounded z-depth-1"
     zoomable=true %}
</div>

# Falling Sand

A simple particle simulation where local rules approximate gravity, allowing sand to fall and pile up.
<div class="text-center">
  {% include figure.liquid
     path="assets/projects/cellular_automata/falling_sand.gif"
     width="65%"
     class="img-fluid rounded z-depth-1"
     zoomable=true %}
</div>
