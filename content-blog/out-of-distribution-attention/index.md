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
/*  margin-bottom: 2em;*/
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
.subcaption {
  display: block;
  margin-top: 0.4em;
  line-height: 1.75em;
  font-size: 80%;
}
details > p {
  color: lightgrey;
  padding-left: 1em;
}
details > summary {
  cursor: pointer;
}
.approx-toggle {
  color: lightcoral;
}
.approx-toggle input {
  margin-right: 0.5em;
}
.hidden {
  display: none;
}
</style>

Does a diffusion model require retraining to generate images smaller or larger than those in its training set?

On inspection, it seems so:

<figure class="table-fig">
  <table class="no-border-bottom">
    <thead>
      <th>200x200</th>
      <th>256x256</th>
      <th>512x512 *</th>
      <th>768x768</th>
    </thead>
    <tbody>
      <tr>
        <td>{{<linked-img src="./images/00295.715317074.sd1.5.regular200.png" width="175px" alt="Completely destroyed image; just garish stripes">}}</td>
        <td>{{<linked-img src="./images/00285.715317074.sd1.5.regular256.png" width="175px" alt="Substantially damaged image; garish, over-exposed. A low-detail person stands in front of a hill." >}}</td>
        <td>{{<linked-img src="./images/00265.715317074.sd1.5.regular512.png" width="175px" alt="Detailed 3D render of a vaporwave shrine maiden." >}}</td>
        <td>{{<linked-img src="./images/00275.715317074.sd1.5.regular768.png" width="175px" alt="Two shrine maidens melding into one." >}}</td>
      </tr>
    </tbody>
  </table>
  <figcaption>
    Optimal = 512. Smaller = deep-fried. Larger = duplication of body parts
    <div class="subcaption">Images generated with stable-diffusion 1.5</div>
  </figcaption>
</figure>

<p>
<details>
  <summary>Stable-diffusion 1.x was trained on 512x512 images, encoded as 64x64 latents.</summary>
  <p>
    Stable-diffusion is a <em>latent diffusion</em> model.<br>
    It removes noise from noised <em>latents</em>, not pixels. We start with 100% noise, and iteratively remove noise until we have completely denoised latents.<br>
    A <abbr title="Variational Autoencoder">VAE</abbr> is then used to decode these latents back into pixels.<br>
    <small>Latents are (moreorless) a learned resample of pixels. Stable-diffusion downsamples the image area by 8x in each dimension, but resamples RGB onto 4 channels.</small>
  </p>
</details>
</p>

<strong>Larger</strong>-than-512² images exhibit "double-body" or other deformations:

<figure class="table-fig">
  <table class="no-border-bottom">
    <tbody>
      <tr>
        <td>{{<linked-img src="./images/768/vae/00368.86322125.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00369.340323845.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00370.340323845.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00371.436376137.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00372.436376137.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00373.580263270.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00374.580263270.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00383.715317074.sd1.5.regular768.png" width="75px" >}}</td>
      </tr>
      <tr id="approx-768-target" class="hidden">
        <td>{{<linked-img src="./images/768/approx_beeg/00368.86322125.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00369.340323845.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00370.340323845.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00371.436376137.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00372.436376137.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00373.580263270.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00374.580263270.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00383.715317074.sd1.5.regular768.approx.png" width="75px" >}}</td>
      </tr>
    </tbody>
  </table>
  <figcaption>
    768x768 samples: most exhibit "double-body" or other deformation.<br>
    <div class="subcaption">
      <label class="approx-toggle"><input type="checkbox" id="approx-768">Show approx decodes</label>, to see that problem is <strong>not</strong> caused by the <abbr title="Variational Autoencoder">VAE</abbr>.
    </div>
  </figcaption>
</figure>

<strong>Smaller</strong>-than-512² images seem to be detail-impaired but don't make the same composition mistakes:

<figure class="table-fig">
  <table class="no-border-bottom">
    <tbody>
      <tr>
        <td>{{<linked-img src="./images/256/vae/00307.436376137.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00308.436376137.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00310.580263270.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00313.830333947.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00316.1157730004.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00320.1385218415.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00321.1542102181.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00334.2106704619.sd1.5.regular256.png" width="75px" >}}</td>
      </tr>
      <tr id="approx-256-target" class="hidden">
        <td>{{<linked-img src="./images/256/approx_beeg/00339.436376137.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00340.436376137.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00342.580263270.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00345.830333947.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00348.1157730004.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00352.1385218415.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00353.1542102181.sd1.5.regular256approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/approx_beeg/00366.2106704619.sd1.5.regular256approx.png" width="75px" >}}</td>
      </tr>
    </tbody>
  </table>
  <figcaption>
    256x256 samples: correctly-composed, but detail-impaired.
    <div class="subcaption">
      <label class="approx-toggle"><input type="checkbox" id="approx-256">Show approx decodes</label>, to see that problem is <strong>not</strong> caused by the <abbr title="Variational Autoencoder">VAE</abbr>.
    </div>
  </figcaption>
</figure>

What makes it perform poorly at different image sizes?  
Is there any part of the diffusion model which is sensitive to image dimensions, sequence length, or distance between features?

<h3>Convolutions?</h3>

Can convolutions be responsible for long-distance composition decisions?

Each resnet block in the Unet downsamples our latents; we undergo an 8x downsample by the time we reach the bottom. 64x64 latents become 8x8. At this size: a 3x3 convolution kernel can actually include quite a lot of image.

<figure class="table-fig">
  <table class="no-border-bottom">
    <thead>
      <th>512x512</th>
      <th>768x768</th>
    </thead>
    <tbody>
      <tr>
        <td>{{<linked-img src="./images/latents-approx-conv.miko.512.png" width="175px" alt="largest area covered by convolution kernel for this 512x512 close portrait, is about the size of a face." >}}</td>
        <td>{{<linked-img src="./images/latents-approx-conv.miko.768.png" width="175px" alt="largest area covered by convolution kernel for this 768x768 less-zoomed-in portrait is about the size of a head." >}}</td>
      </tr>
    </tbody>
  </table>
  <figcaption>
    Latent area covered by 3x3 convolution kernel at deepest part of Unet
    <div class="subcaption">
      Latents decoded to pixels using an <a href="https://twitter.com/Birchlabs/status/1638680623110864898">approximate decoder</a> distilled from SD1.5's <abbr title="Variational Autoencoder">VAE</abbr>,<br>whose color-space conversion is not sensitive to sequence length.
    </div>
  </figcaption>
</figure>

The convolution kernel doesn't include the whole image; it extracts features from a local neighbourhood. A possible intuition is that this helps with completing forms coherently (continuing the outline of a head), but that long-distance decisions (a head cannot be here because we already drew one elsewhere) are only possible transitively (a head is followed by a neck is followed by a torso).  

The difference in distance (width or height) under the convolution kernel between our 512 and 768 image is only 50%. Stable-diffusion can draw subjects close or far from the camera, so perhaps it can tolerate a difference of this much.

It's not enough to explain why small images get deep-fried. If the goal were "fit a body part into the kernel", then that's <em>easier</em> for small images.


<script>
  const checkboxIDs = ['approx-768', 'approx-256'];
  for (const checkboxID of checkboxIDs) {
    const targetID = `${checkboxID}-target`;
    document.getElementById(checkboxID).addEventListener('change', (event) => {
      const classList = document.getElementById(targetID).classList;
      if (event.target.checked) {
        classList.remove('hidden');
      } else {
        classList.add('hidden');
      }
    });
  }
</script>