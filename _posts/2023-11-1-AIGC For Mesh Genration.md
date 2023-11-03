---
layout: post
title: 模型的AIGC相关库
tags:
published: false
---

# [**Magic3D: High-Resolution Text-to-3D Content Creation**](https://research.nvidia.com/labs/dir/magic3d/)**Text to 3D Model**

​                ✓ **Object Level**

​                ● DreamFusion: [DreamFusion: Text-to-3D using 2D Diffusion](https://dreamfusion3d.github.io/)

​                ● Magic3D: [Magic3D: High-Resolution Text-to-3D Content Creation](https://research.nvidia.com/labs/dir/magic3d/)

​                ● Fantasia3D（有源码）: [Fantasia3D: Disentangling Geometry and Appearance for High-quality Text-to-3D Content Creation](https://fantasia3d.github.io/)

​                ● ProlificDreamer: [ProlificDreamer: High-Fidelity and Diverse Text-to-3D Generation with Variational Score Distillation](https://ml.cs.tsinghua.edu.cn/prolificdreamer/)

​                ● Shape-E（有源码）: https://github.com/openai/shap-e

​                ● Latent-NeRF（有源码）: [GitHub - eladrich/latent-nerf: Official Implementation for "Latent-NeRF for Shape-Guided Generation of 3D Shapes and Textures"](https://github.com/eladrich/latent-nerf)

​                ● Text2Mesh: [Text2Mesh Text-Driven Neural Stylization for Meshes | Oscar Michel1, Roi Bar-On1,2, Richard Liu*1, Sagie Benaim2, Rana Hanocka1](https://threedle.github.io/text2mesh/)

​                ● Zero123: [Zero-1-to-3: Zero-shot One Image to 3D Object](https://zero123.cs.columbia.edu/)

​                ● One-2-3-45（有源码）: [One-2-3-45](https://one-2-3-45.github.io/)

​                ● [GitHub - OpenRobotLab/PointLLM（有源码）: [arXiv 2023\] PointLLM: Empowering Large Language Models to Understand Point Clouds](https://github.com/OpenRobotLab/PointLLM)

​                ● （有源码）[MVDream: Multi-view Diffusion for 3D Generation](https://mv-dream.github.io/index.html)

​                ● （有源码）[SyncDreamer: Generating Multiview-consistent Images from a Single-view Image](https://liuyuan-pal.github.io/SyncDreamer/)



​                ✓ **Scene Level**

​                ● SceneScape: [SceneScape: Text-Driven Consistent Scene Generation](https://scenescape.github.io/)

​                ● Text2Room: [Text2Room: Extracting Textured 3D Meshes from 2D Text-to-Image Models](https://lukashoel.github.io/text-to-room/)

​                ● Text2NeRF: [Text2NeRF: Text-Driven 3D Scene Generation with Neural Radiance Fields](https://eckertzhang.github.io/Text2NeRF.github.io/)



# **3D Implicit Representation**

[3D Gaussian Splatting for Real-Time Radiance Field Rendering](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/)

[MobileNeRF](https://mobile-nerf.github.io/)



# **Generate Mesh**

已经研究过：

​                ● Real Fusion(CVPR 2023)

​                ○ [GitHub - lukemelas/realfusion](https://github.com/lukemelas/realfusion)	

​                ● Texture(CVPR 2023): Text生成贴图，考虑模型的外形信息，质量尚可

​                ○ https://github.com/TEXTurePaper/TEXTurePaper

​                ● Pointe: Text生成点云。Openai, ChatGPT的公司做的，当时测的时候生成速度挺快，但是只能生成5w个点的点云

​                ○ https://github.com/openai/point-e

​                ● Nerf2Mesh(CVPR 2023): 通过Nerf+多图，生成Diffuse+Specular Texture，尚未测试

​                ○ [GitHub - ashawkey/nerf2mesh: [ICCV2023\] Delicate Textured Mesh Recovery from NeRF via Adaptive Surface Refinement](https://github.com/ashawkey/nerf2mesh)

​                ● 3D Highlighter(CVPR 2023): 标注模型的不同区域，如灯的腿、蜡烛的帽子等奇怪的区域等。但是测试后效果一般，不太稳定

​                ○ [GitHub - threedle/3DHighlighter: Localizing Regions on 3D Shapes via Text Descriptions](https://github.com/threedle/3DHighlighter)

​                ● Stable DreamFusion：融合多个Generate Mesh的开源项目的一个库，效果不错，支持TextToMesh \ TextureToMesh

​                ○ [GitHub - ashawkey/stable-dreamfusion: Text-to-3D & Image-to-3D & Mesh Exportation with NeRF + Diffusion.](https://github.com/ashawkey/stable-dreamfusion)



将来研究方向：

主要包括两个方向：图/视频/文字生成模型、模型生成模型

​                ● ProlificDreamer:清华团队推出的TextToMesh，看宣传效果不错，但是代码尚未开源

​                ○ [GitHub - thu-ml/prolificdreamer: ProlificDreamer: High-Fidelity and Diverse Text-to-3D Generation with Variational Score Distillation](https://github.com/thu-ml/prolificdreamer)

​                ● ThreeStudio：包含当前最好的多个GenerateMesh的集合库

​                ○ [GitHub - threestudio-project/threestudio: A unified framework for 3D content generation.](https://github.com/threestudio-project/threestudio)

​                ● Neuralangelo: 英伟达推出的视频or多图生成模型的库，效果看起来不错，尚待测试

​                ○ [GitHub - NVlabs/neuralangelo: Official implementation of "Neuralangelo: High-Fidelity Neural Surface Reconstruction" (CVPR 2023)](https://github.com/NVlabs/neuralangelo)

​                ● Sin3DM：MeshToMesh，看起来像PCG的模型扩展，尚待测试

​                ○ [Sin3DM](https://sin3dm.github.io/)



# **贴图生成**

​                ● Text2Tex

[GitHub - daveredrum/Text2Tex: [ICCV 2023\] Text2Tex: Text-driven Texture Synthesis via Diffusion Models](https://github.com/daveredrum/Text2Tex)

​                ● TEXTure

[GitHub - TEXTurePaper/TEXTurePaper: Official Implementation for "TEXTure: Text-Guided Texturing of 3D Shapes"](https://github.com/TEXTurePaper/TEXTurePaper)

# **材质生成**

​                ● Adobe 3D Sampler - Image to Material

​                ● Materialize

[Bounding Box Software - Materialize](https://www.boundingboxsoftware.com/materialize/getkey.php)

​                ● https://github.com/xilongzhou/Material_prior

​                ● [GitHub - mit-gfx/diffmat: PyTorch-based differentiable material graph library for procedural material capture](https://github.com/mit-gfx/diffmat)

​                ● [Physically Based - The PBR values database](https://physicallybased.info/)