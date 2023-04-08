---
title: "Generating Out-of-Distribution Images with Stable-Diffusion"
date: 2023-04-08T00:00:00Z
lastmod: 2023-04-08T00:00:00Z
draft: True
description: "Fiddling with attention formulation to output in-distribution attention probabilities"
author: ["Alex Birch"]
contributors: ["EleutherAI"]
categories: ["Announcement"]
---

<style>
figure {
  display: inline-block;
}
figure.table-fig {
  margin-top: initial;
  margin-bottom: initial;
}
table.no-border-bottom:not(.highlighttable, .highlight table, .gist .highlight) td {
  border-bottom: initial;
}
figure.table-fig figcaption {
  margin-top: initial;
  text-align: center;
  font-style: italic;
  font-weight: lighter;
}
</style>

Is it necessary to retrain a diffusion model, to generate images smaller or larger than those in its training set?

On inspection, it seems so:

<figure class="table-fig">
  <table class="no-border-bottom">
    <thead>
      <th>200x200</th>
      <th>256x256</th>
      <th>512x512</th>
      <th>768x768</th>
    </thead>
    <tbody>
      <tr>
        <td><img src="./images/00295.715317074.sd1.5.regular200.png" width="175px" alt="Completely destroyed image; just garish stripes" /></td>
        <td><img src="./images/00285.715317074.sd1.5.regular256.png" width="175px" alt="Substantially damaged image; garish, over-exposed. A low-detail person stands in front of a hill." /></td>
        <td><img src="./images/00265.715317074.sd1.5.regular512.png" width="175px" alt="Detailed 3D render of a vaporwave shrine maiden." /></td>
        <td><img src="./images/00275.715317074.sd1.5.regular768.png" width="175px" alt="Two shrine maidens melding into one." /></td>
      </tr>
    </tbody>
  </table>
  <figcaption>Stable-diffusion 1.5. Optimal = 512. Smaller = deep-fried. Larger = duplication of body parts</figcaption>
</figure>

Stable-diffusion 1.x (and stable-diffusion 2.x-base) were trained on 512x512 images, encoded as 64x64 latents.

What specifically makes it perform poorly at different image sizes?