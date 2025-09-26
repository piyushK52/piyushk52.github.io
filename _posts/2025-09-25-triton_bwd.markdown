---
layout: post
title:  "Triton Fused Backward Layer Norm"
date:   2025-09-25 20:44:32 +0530
categories: jekyll update
---

This post will go over some of the overlooked details of fused backward layer norm implementation in triton. Here I will mostly go into the maths and visualization, for the complete code you can check the original Triton tutorial - [Layer Normalization](https://triton-lang.org/main/getting-started/tutorials/05-layer-norm.html)

First thing to cover is the derivation of Vector Jacobian Product of x, given by the formula below:
<br>
<div style="text-align: center;">
  <img src="/asset/images/vjp_x.png" alt="VJP X" style="max-height: 400px; max-width: 100%; margin: 0 auto;">
  <figcaption>Vector Jacobian Product of X</figcaption>
</div>
<br>

Here are some basic differentials that we will ultimately plug into the final formula.
<br>
<div style="text-align: center;">
  <img src="/asset/images/vjp_derivation_1.png" alt="VJP X derivation" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
  <figcaption>VJP differentials</figcaption>
</div>
<br>

Plugging all the above differentials into the main formula below we get our VJP of x.
<br>
<div style="text-align: center;">
  <img src="/asset/images/vjp_derivation_2.png" alt="VJP X derivation" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
  <figcaption>VJP formula</figcaption>
</div>
<br>

We can also see the straightforward derivation of the VJPs of w and b above.
<br>
<div style="text-align: center;">
  <img src="/asset/images/vjp_wb.png" alt="VJP WB" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
  <!-- <figcaption>VJP WB</figcaption> -->
</div>
<br>

Now, since these each input in a batch will affect the gradient calculation of W and B, we will need to sum up the calculated dw and db matrix along the batch dimension. For summing the inputs along the batch we parallelize the sum in small groups, this partial sum (parallelized) is finally reduced again to get the output.
<br>
<div style="text-align: center;">
  <img src="/asset/images/w_reduction.png" alt="Sum reduction" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
  <figcaption>Sum reduction (with dummy values)</figcaption>
</div>
<br>

The reduction from dW to dW_partial is simple where we assign one program to each row but for reducing dW_partial to dW_final we do it in groups of BLOCK_SIZE_M. This is mainly due to the fact that unlike stage 1 (dw -> dw_partial) which was somewhat compute intensive with the calculation of dX as well in this stage we are purely doing reduction.
<br>
<div style="text-align: center;">
  <img src="/asset/images/dw_final_reduction.png" alt="Final reduction" style="max-height: 600px; max-width: 100%; margin: 0 auto;">
  <figcaption>Final reduction (Stage 2)</figcaption>
</div>
<br>

BLOCK_SIZE_N is basically N in the tutorial code but it can be better found through auto-tuning. To adapt this code for diffusion transformers last three dimensions should be taken into consideration for normalization (B, C, H, W) -> (B, C * H * W).