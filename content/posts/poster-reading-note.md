+++
title = "Reading Notes of the Paper POSTER"
date = 2024-03-13
draft = true

[taxonomies]
tags = ["deep-learning"]
+++



# POSTER

## List of Datasets to Download

1. RAF-DB

2. FERPlus

3. AffectNet 7 class & 8 class

## Points

- There are existing works that focus separately on inter-class similarity, intra-class discrepancy and scale sensitivity.

- POSTER aims to solve the 3 problems as a whole.

- techniques

    - 2 stream

        - image

            contains global features like cheeks, forehead, and tears drop that landmarks don't involve

        - landmark

            reduce the effect from the image background and focus on the *salient* region

    - pyramid
    - cross-fusion
    - transformer

Motivation for designing a transformer-based cross-fusion block: let the 2 streams guide each other. the design alleviates **inter-class similarity** and **intra-class discrepancy**.

While the use of a pyramid is to reduce the effect of **scale sensitivity**.

## Related Works

1. **Deep Learning** in FER. There are processes that have been made. From Region Attention Network(RAN) to ( idk what this is in detail) Deep Attentive Center Loss which can estimate the attention weight for the features for enhancing discrimination. And more.
2. **Facial Landmarks** in FER. Used in Face recognition, tracking, and emotion recognition. Since the DL technique has been employed for facial landmark detection tasks, and many accurate detectors have been proposed. Researchers now can focus more on dealing with the landmark itself as a feature. Hence networks that take both images and landmarks as input were proposed. (3 works are mentioned, all ignore the correlations of facial landmarks and image features.)
3. **Vision Transformer**. ViT, CrossViT

## Methodology

1. preprocesser(i call it)

    - Facial landmark detector

        Cunjian Chen. PyTorch Face Landmark: A fast and accurate facial landmark detector, 2021

    - Image backbone

        Jiankang Deng, Jia Guo, Niannan Xue, and Stefanos Zafeiriou. Arcface: Additive angular margin loss for deep face recognition. In *Proceedings of the IEEE/CVF conference on computer vision and pattern recognition*, pages 4690–4699, 2019

2. cross-fusion transformer encoder

    - MSA
        - QKV encoding
        - double-head?
        - do the attention
    - norm and add and MLP and add

3. pyramid

    **By Conv1d in different kernel sizes and strides**

## Remaining Questions

- What is the projection in `AttentionBlock` for? `self.proj = nn.Linear(dim, dim)`
- Why is there no pos_embed in for `x_lm` in the ViT? Is it just because the landmark already contained position information?

# POSTERV2

## Difference

1. **Drop the image-to-landmark branch**. That is, it performs transformer only on `x_img`.
2. **Drop Cross Fusion**. Use global `q` from landmarks instead of local ones. Meanwhile do local attention too.
3. **Multi-scale by ir and lm**.

## Questions

- After obtaining global attention features `o1`, `o2,` and `o3`, why use different ways to extract `q`, `k,` and `v`? (2*Conv2d, Conv2d, and none) well I guess it's to make the `o1`, `o2,` and `o3` to the same shape.
