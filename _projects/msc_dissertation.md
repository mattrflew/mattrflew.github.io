---
layout: page
title: Kalman Tracking for Super-Resolution Ultrasound Imaging
description: A Kalman-filter-based microbubble tracking pipeline developed for my MSc dissertation at the University of Edinburgh.
img:
redirect:
importance: 1
category: ""
related_publications: false
---

<!-- Project Description -->
This project formed the basis of my MSc dissertation in Computational Applied Mathematics at the University of Edinburgh. I developed and evaluated a **Kalman Filter–based microbubble tracking pipeline** for **Super-Resolution Ultrasound Imaging (SRUI)** using the [ULTRA-SR](https://ultra-sr.com/) challenge dataset.


<!-- PDF Download -->
<p>
  <a href="/assets/projects/msc_dissertation/DissertationMatthewFlewelling.pdf.pdf" target="_blank" rel="noopener">
    <i class="fa-solid fa-file-pdf fa-2x"></i>
    &nbsp; Download Dissertation (PDF)
  </a>
</p>



<!-- Tracking Possibilities Image -->
<div class="row mt-3">
  <div class="col-sm-6">
    {% include figure.liquid
       path="assets/projects/msc_dissertation/track_vis_ver3.png"
       class="img-fluid rounded z-depth-1"
       caption="Example tracking possibility 1" %}
  </div>

  <div class="col-sm-6">
    {% include figure.liquid
       path="assets/projects/msc_dissertation/track_vis_ver4.png"
       class="img-fluid rounded z-depth-1"
       caption="Example tracking possibility 2" %}
  </div>
</div>

<div class="caption">
  Visualisation of two different tracking possibilities applied to the same set of detected microbubbles across four consecutive frames. Filled circles mark detections and dashed lines represent tracks, highlighting the ambiguity in linking detections over time.
</div>
