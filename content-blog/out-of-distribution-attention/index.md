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
/*  margin-top: initial;*/
  margin-top: 0;
  margin-bottom: 0.5em;
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

p.tight-to-figure {
  margin-bottom: 0;
}

figure.hidden + .tight-to-figure {
  margin-top: 1em;
}

details > summary::marker {
  font-style: initial;
}
details > summary ~ * {
  margin-left: 1.7em;
}
details > summary {
  cursor: pointer;
  list-style-type: "[+] ";
  user-select: none;
}
details[open] > summary {
  list-style-type: "[−] ";
  margin-bottom: 0;
}
details.margin-bottom {
  margin-bottom: 1em;
}

.toggle {
  color: lightcoral;
  user-select: none;
  cursor: pointer;
}
.toggle:hover {
  text-decoration: underline;
  text-decoration-style: dotted;
}
.toggle input {
  margin-right: 0.5em;
}
.hidden {
  display: none;
}

h4 {
  font-weight: lighter;
}
</style>

Does a diffusion model require retraining to generate images smaller or larger than those in its training set?

It seems so, judging by <a href="https://en.wikipedia.org/wiki/Stable_Diffusion">stable-diffusion</a> 1.5:

<figure class="table-fig">
  <table class="no-border-bottom">
    <thead>
      <tr>
        <th>200x200</th>
        <th>256x256</th>
        <th>512x512 *</th>
        <th>768x768</th>
      </tr>
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
    512² = Optimal. Smaller = deep-fried. Larger = duplication of body parts
  </figcaption>
</figure>

<details>
  <summary><em>Background: how does stable-diffusion generate images?</em></summary>
  <p>
    Stable-diffusion is a <a href="https://arxiv.org/abs/2112.10752">latent diffusion</a> model. Where a typical diffusion model removes noise from noised <em>pixels</em>: latent diffusion models remove noise from noised <em>latents</em> — a <em>learned downsample</em> of pixels. This reduces the dimensionality of the denoising problem.
  </p>
  <p>
    To produce a 512x512 image:
  </p>
  <ul>
    <li>We start by generating 64x64, 4-channel noise (noised latents).</li>
    <li>The Unet (and our ODE solver) iteratively remove portions of the noise from these noised latents, until we have fully-denoised latents.</li>
    <li>We pass our latents to the <abbr title="Variational Autoencoder">VAE</abbr> decoder, which upsamples them to a 512x512 RGB image.</li>
  </ul>
</details>
<p>
  Stable-diffusion 1.5 was <a href="https://huggingface.co/runwayml/stable-diffusion-v1-5">trained primarily* to generate 512x512 images</a>.<br>
  <small>*SD1.5 is descended from SD1.1, which was trained to generate 256² images.</small>
</p>

<p class="tight-to-figure">
  <strong>768²</strong> images exhibit "double-body" or other deformations:
</p>

<!-- <small><label class="toggle"><input type="checkbox" id="approx-768">Show approx decodes</label>, to see that problem is <strong>not</strong> caused by the <abbr title="Variational Autoencoder">VAE</abbr>.</small> -->

<figure class="table-fig">
  <table class="no-border-bottom">
    <tbody>
      <tr>
        <!-- <td>
          Decoder:<br>
          <label><input type="radio" name="768-decoder" value="VAE" checked>VAE</label><br>
          <label><input type="radio" name="768-decoder" value="approx">Approx</label>
        </td> -->
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
  <!-- <figcaption>
    768x768 samples: most exhibit "double-body" or other deformation.<br>
    <div class="subcaption">
      <label class="toggle"><input type="checkbox" id="approx-768">Show approx decodes</label>, to see that problem is <strong>not</strong> caused by the <abbr title="Variational Autoencoder">VAE</abbr>.
    </div>
  </figcaption> -->
</figure>

<p class="tight-to-figure">
  <strong>512²</strong> images (<label class="toggle"><input type="checkbox" id="baseline-512">Show</label>) are our baseline; bodies generally coherent, colour choices reasonable.
</p>

<figure class="table-fig hidden" id="baseline-512-target">
  <table class="no-border-bottom">
    <tbody>
      <tr>
        <td>{{<linked-img src="./images/512/00193.1157730004.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00181.86322125.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00187.580263270.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00189.715317074.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00190.715317074.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00195.1289965640.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00197.1385218415.sd1.5.regular512.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/512/00200.1542102181.sd1.5.regular512.png" width="75px" >}}</td>
      </tr>
    </tbody>
  </table>
  <!-- <figcaption>
    512x512 samples: bodies generally coherent (go easy on the hands!)<br>
  </figcaption> -->
</figure>

<p class="tight-to-figure">
  <strong>256²</strong> images lack detail and have odd colours, but have reasonable composition:
</p>

<!-- <small><label class="toggle"><input type="checkbox" id="approx-256">Show approx decodes</label>, to see that problem is <strong>not</strong> caused by the <abbr title="Variational Autoencoder">VAE</abbr>.</small> -->

<figure class="table-fig">
  <table class="no-border-bottom">
    <tbody>
      <tr>
        <!-- <td>
          Decoder:<br>
          <label><input type="radio" name="256-decoder" value="VAE" checked>VAE</label><br>
          <label><input type="radio" name="256-decoder" value="approx">Approx</label>
        </td> -->
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
  <!-- <figcaption>
    256x256 samples: correctly-composed, but detail-impaired.
    <div class="subcaption">
      <label class="toggle"><input type="checkbox" id="approx-256">Show approx decodes</label>, to see that problem is <strong>not</strong> caused by the <abbr title="Variational Autoencoder">VAE</abbr>.
    </div> 
  </figcaption>-->
</figure>

<p>
  What makes stable-diffusion perform poorly at non-512² image sizes?
</p>
<!-- Is it sensitive to image dimensions, sequence length, or distance between features? -->

<h3><abbr title="Variational Autoencoder">VAE</abbr> decoder?</h3>

<!-- <details>
  <summary><em>Background: what is the role of the <abbr title="Variational Autoencoder">VAE</abbr> decoder in latent diffusion?</em></summary>
  <p>
    Stable-diffusion is a <a href="https://arxiv.org/abs/2112.10752">latent diffusion</a> model. Where a typical diffusion model removes noise from noised <em>pixels</em>: latent diffusion models remove noise from noised <em>latents</em> — a <em>learned downsample</em> of pixels. This reduces the dimensionality of the denoising problem.
  </p>
  <p>
    To produce a 512x512 image:
  </p>
  <ul>
    <li>We start by generating 64x64, 4-channel noise (noised latents).</li>
    <li>The Unet (and our ODE solver) iteratively remove portions of the noise from these noised latents, until we have fully-denoised latents.</li>
    <li>We pass our latents to the <abbr title="Variational Autoencoder">VAE</abbr> decoder, which upsamples them to a 512x512 RGB image.</li>
  </ul>
</details> -->
<p>
  Perhaps deformities and detail loss occur during when the <abbr title="Variational Autoencoder">VAE</abbr> decoder upsamples the image? Part of its job is to invent new detail; maybe it does this poorly for image sizes outside of its training distribution?
</p>
<details class="margin-bottom">
  <summary>Well, 256² images might actually be in-distribution for SD1.5's <abbr title="Variational Autoencoder">VAE</abbr>.</summary>
  <ul>
    <li>
      SD1.1 was trained on <a href="https://huggingface.co/runwayml/stable-diffusion-v1-5">256²</a> images
      <ul>
        <li><small>it's unclear whether this refers to the Unet or the <abbr title="Variational Autoencoder">VAE</abbr>.</small></li>
      </ul>
    </li>
    <li>
      SD's 1.4-era <a href="https://huggingface.co/stabilityai/sd-vae-ft-mse">improved <abbr title="Variational Autoencoder">VAE</abbr>s</a> were <em>evaluated</em> against 256² images
      <ul>
        <li><small>it's unclear what size images they were <em>trained</em> on, or whether SD1.5's <abbr title="Variational Autoencoder">VAE</abbr> is derived from them.</small></li>
      </ul>
    </li>
  </ul>
</details>
<p>
  Let's eliminate the <abbr title="Variational Autoencoder">VAE</abbr> altogether. Is there another way to decode the latents, which we trust to not be sensitive to sequence length?
</p>

<h4>Decoding latents with an approximate decoder</h4>
<details class="margin-bottom">
  <summary>I distilled an <a href="https://birchlabs.co.uk/machine-learning#vae-distillation">approximate decoder</a> from real <abbr title="Variational Autoencoder">VAE</abbr> decoding results (latent+image pairs). The model architecture is just 3 dense layers.</summary>

```python
from torch.nn import Module, Linear, SiLU
from torch import FloatTensor

class Decoder(Module):
  def __init__(self, inner_dim = 12) -> None:
    super().__init__()
    self.in_proj = Linear(4, inner_dim)
    self.nonlin0 = SiLU()
    self.hidden_layer = Linear(inner_dim, inner_dim)
    self.nonlin1 = SiLU()
    self.out_proj = Linear(inner_dim, 3)
  
  def forward(self, sample: FloatTensor) -> FloatTensor:
    sample = self.in_proj(sample)
    sample = self.nonlin0(sample)
    sample = self.hidden(sample)
    sample = self.nonlin1(sample)
    sample = self.out_proj(sample)
    return sample
```
</details>

<details class="margin-bottom">
  <summary>This decoder is not sensitive to sequence length.</summary>
  <ul>
    <li>
      <strong>Only</strong> does colour-space conversion (latent channels to RGB)
      <ul>
        <li><small>does <strong>not</strong> upsample / create new details</small></li>
      </ul>
    </li>
    <li>Decodes pixels <em>independently</em> of each other</li>
  </ul>
</details>

<p>
  <small>We use 8x nearest-neighbour upsample to match <abbr title="Variational Autoencoder">VAE</abbr> output size without interpolating details.</small>
</p>

<p class="tight-to-figure">
We observe that the composition mistakes in 768² images and the reduced detail in 256² images are still evident when decoded with this simpler decoder:
</p>

<figure class="table-fig">
  <table class="no-border-bottom">
    <tbody>
      <tr>
        <th>Decoder</th>
        <th colspan="8">Samples (768²)</th>
      </tr>
      <tr>
        <td>VAE</td>
        <td>{{<linked-img src="./images/768/vae/00368.86322125.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00369.340323845.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00370.340323845.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00371.436376137.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00372.436376137.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00373.580263270.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00374.580263270.sd1.5.regular768.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/vae/00383.715317074.sd1.5.regular768.png" width="75px" >}}</td>
      </tr>
      <tr>
        <td>Approx</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00368.86322125.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00369.340323845.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00370.340323845.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00371.436376137.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00372.436376137.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00373.580263270.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00374.580263270.sd1.5.regular768.approx.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/768/approx_beeg/00383.715317074.sd1.5.regular768.approx.png" width="75px" >}}</td>
      </tr>
      <tr>
        <th>Decoder</th>
        <th colspan="8">Samples (256²)</th>
      </tr>
      <tr>
        <td>VAE</td>
        <td>{{<linked-img src="./images/256/vae/00307.436376137.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00308.436376137.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00310.580263270.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00313.830333947.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00316.1157730004.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00320.1385218415.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00321.1542102181.sd1.5.regular256.png" width="75px" >}}</td>
        <td>{{<linked-img src="./images/256/vae/00334.2106704619.sd1.5.regular256.png" width="75px" >}}</td>
      </tr>
      <tr>
        <td>Approx</td>
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
</figure>

<p>So, the problem does <strong>not</strong> lie with the <abbr title="Variational Autoencoder">VAE</abbr>.</p>

<h3>Convolutions?</h3>

<p>Can convolutions be responsible for long-distance composition decisions?</p>

<p>A 3x3 convolution kernel doesn't cover a large area, but its effective area increases as we descend the Unet (and resnet blocks downsample the latents).</p>

<p>
We undergo an 8x downsample by the time we reach the bottom. 64x64 latents become 8x8. At this size: a 3x3 convolution kernel can actually include quite a lot of image.
</p>

<figure class="table-fig">
  <table class="no-border-bottom">
    <thead>
      <tr>
        <th>Pixel size</th>
        <th/>
        <th>512x512</th>
        <th>768x768</th>
      </tr>
      <tr>
        <th rowspan="3">Latent size</th>
        <th>Unet depth</th>
        <th colspan="2"/>
      </tr>
      <tr>
        <th>top</th>
        <th>64x64</th>
        <th>96x96</th>
      </tr>
      <tr>
        <th>bottom</th>
        <th>8x8</th>
        <th>12x12</th>
      </tr>
      <!-- <tr>
        <th>3x3 conv covers (% of canvas)</th>
        <th>14</th>
        <th>6</th>
      </tr> -->
    </thead>
    <tbody>
      <tr>
        <td colspan="2"/>
        <td>{{<linked-img src="./images/latents-approx-conv.miko.512.png" width="175px" alt="largest area covered by convolution kernel for this 512x512 close portrait, is about the size of a face." >}}</td>
        <td>{{<linked-img src="./images/latents-approx-conv.miko.768.png" width="175px" alt="largest area covered by convolution kernel for this 768x768 less-zoomed-in portrait is about the size of a head." >}}</td>
      </tr>
    </tbody>
  </table>
  <figcaption>
    Area covered by 3x3 convolution kernel at deepest part of Unet
  </figcaption>
</figure>

<p>
  The convolution kernel doesn't include the whole image; it extracts features from a local neighbourhood. This enables <em>local</em> coherence (continuing the outline of a head), and a chance at long-distance coherence via transitivity (a head is followed by a neck is followed by a torso). But overall lacks the range to enforce "a head cannot be here because we already drew one elsewhere".
</p>
<p>
  The difference in distance (width or height) under the convolution kernel between our 512 and 768 image is only 50%. Stable-diffusion can draw subjects close or far from the camera, so perhaps it can tolerate a difference of this much.
</p>
<p>
  It's not enough to explain why <em>small</em> images get deep-fried. If the goal were "fit a body part into the kernel", then that's <em>easier</em> for small images.
</p>
<p>
  Not ruling out convolutions as a possible culprit, but do not see a training-free solution.<br>
  As for bigger images: deeper Unet?
</p>

<script>
  const checkboxIDs = ['baseline-512'];//, 'approx-768', 'approx-256'];
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