# SageAttention

This repository provides the official implementation of SageAttention and SageAttention2.

**SageAttention: Accurate 8-Bit Attention for Plug-and-play Inference Acceleration**  
Paper: https://arxiv.org/abs/2410.02367  
Jintao Zhang, Jia Wei, Haofeng Huang, Pengle Zhang, Jun Zhu, Jianfei Chen

**SageAttention2 Technical Report: Accurate 4-Bit Attention for Plug-and-play Inference Acceleration**  
Paper: https://arxiv.org/abs/2411.10958  
Jintao Zhang, Haofeng Huang, Pengle Zhang, Jia Wei, Jun Zhu, Jianfei Chen

![Local Image](./resource/intro.png)
*SageAttention on CogvideoX-2B (RTX4090)*

![Local Image](./resource/intro2.png)
*SageAttention2 on Llama3.1-8B*

## Beta Version of SageAttention2
This is a beta release of SageAttention2. We welcome any feedback on accuracy, performance issues, bugs, feature requests, or suggestions. Please feel free to open an issue or launch a pull request!

Current Features:
+ INT8 quantization for $QK^\top$
+ FP8 quantization for $PV$
+ FP32 buffer for $PV$ accumulator to enhance the accuracy of the low-precision (FP22) accumulator. 

For stable version, please use [SageAttention-1](https://github.com/thu-ml/SageAttention/tree/sageattention-1).

## Project Updates
- **News** 2024-11-21: SageAttention 2.0.0 beta is released! Now SageAttention has measured speedup on L20, L40, A100, A800 and A6000 other than RTX3090 and RTX4090.
- **News** 2024-11-12: Support for `sageattn_varlen` is available now.
- **News** 2024-11-11: Support for different sequence length between `q` and `k,v`,  `(batch_size, head_num, seq_len, head_dim)` or `(batch_size, seq_len, head_num, head_dim)` input shapes, and `group-query attention` is available now.


## Base environment
+ `python>=3.9`   
+ `torch>=2.3.0`  
+ `triton>=3.0.0` 
+ `CUDA>=12.4` if you want to use fp8 else `CUDA>=12.0`

We recommend to install: (the kernel will be faster a little)  
+ `python>=3.11`  
+ `torch>=2.4.0`  
+ `triton-nightly`
+ `CUDA=12.6`


## Installation

For the stable version or Triton-only version, refer to [SageAttention-1](https://github.com/thu-ml/SageAttention/tree/sageattention-1) and install using pip:
```
pip install sageattention==1.0.6
```

To use SageAttention 2.0.0, please compile from source:
```
git clone https://github.com/thu-ml/SageAttention.git
cd sageattention 
pip install -e . # or python setup.py install
```


> **Note:** Currently, SageAttention is optimized for excellent performance on RTX4090, RTX3090, L20, and L40 GPUs. On A100, A800, and A6000 GPUs, performance is best with a `head_dim=128`, while `head_dim=64` is less optimal. Similarly, performance on the Hopper architecture is currently not optimal. We are actively working to enhance performance in these configurations.


## How to Use
```python
from sageattention import sageattn
attn_output = sageattn(q, k, v, tensor_layout="HND", is_causal=False)
```
+ `q, k, v` are **FP16/BF16** dtype with the shape `(batch_size, head_num, seq_len, head_dim)` using default `tensor_layout="HND"`. For shape `(batch_size, seq_len, head_num, head_dim)`, set `tensor_layout="NHD"`. 
+ `is_causal` determines the use of a causal mask.

### Available APIs:
+ `sageattn`: Automatically selects the optimal kernel based on the GPU to achieve a good performance-accuracy trade-off.
+ `sageattn_qk_int8_pv_fp16_triton`: INT8 quantization for $QK^\top$ and FP16 for $PV$ with FP16 accumulator using Triton backend.
+ `sageattn_qk_int8_pv_fp16_cuda`: INT8 quantization for $QK^\top$ and FP16 for $PV$ using CUDA backend.
+ `sageattn_qk_int8_pv_fp8_cuda`: INT8 quantization for $QK^\top$ and FP8 for $PV$ using CUDA backend.
+ `sageattn_varlen`: INT8 quantization for $QK^\top$ and FP16 for $PV$ with FP16 accumulator using Triton backend. Support for varying sequence lengths within the same batch.

For optimal speed and accuracy performance on custom devices and models, we strongly recommend referring to the [this file](./sageattention/core.py) for detailed guidance.

> **Note:**
Support for `head_dim` values of `64`, `96`, and `128` is currently available. Extended support for other `head_dim` is under development.
Support for different sequence length between `q` and `k,v` and `group-query attention` is available.


## **Plug-and-play Example**

**We can replace `scaled_dot_product_attention` easily.**  
We will take [Cogvideo](https://huggingface.co/THUDM/CogVideoX-2b) as an example:

**Just add the following codes and run!**
```python
from sageattention import sageattn
import torch.nn.functional as F

F.scaled_dot_product_attention = sageattn
```

Specifically,

```bash
cd example
python sageattn_cogvideo.py
```

**You can get a lossless video in** `./example` **faster than by using** `python original_cogvideo.py`

> **Note:** Not all models use `F.scaled_dot_product_attention`, so maybe you should replace the original Attention by modifying the `Attention Class` of the target model.


## Performance
### Speed of Kernels

*`8+8` means the kernel with INT8 quantization for $QK^\top$ and FP8 quantization for $PV$. `8+16` uses FP16 for $PV$.*
![Local Image](./resource/A100_hd128.png)

![Local Image](./resource/A800_hd128.png)

![Local Image](./resource/4090_hd64.png)

![Local Image](./resource/4090_hd128.png)

![Local Image](./resource/L20_hd64.png)

![Local Image](./resource/L20_hd128.png)

![Local Image](./resource/A6000_hd128.png)

![Local Image](./resource/3090_hd64.png)

![Local Image](./resource/3090_hd128.png)


> **Note:** The TOPS results refer only to the Attention Kernel, excluding the quantization and smoothing. we use FP16 accumulator for FP16 $PV$, and FP32 accumulator for FP8 $PV$.

### End-to-end Performance
<!-- ![Local Image](./resource/real_speedup.png) -->

*The table below shows the end-to-end performance across various models using [SageAttention-1](https://github.com/thu-ml/SageAttention/tree/sageattention-1). For more evaluation, please refer to our papers.*
![Local Image](./resource/end-to-end_performance.png)


## Citation
If you use this code or find our work valuable, please cite:
```
@misc{zhang2024sageattention,
      title={SageAttention: Accurate 8-Bit Attention for Plug-and-play Inference Acceleration}, 
      author={Jintao Zhang and Jia wei and Haofeng Huang and Pengle Zhang and Jun Zhu and Jianfei Chen},
      year={2024},
      eprint={2410.02367},
      archivePrefix={arXiv},
      primaryClass={cs.LG},
      url={https://arxiv.org/abs/2410.02367}, 
}

@misc{zhang2024sageattention2,
      title={SageAttention2 Technical Report: Accurate 4 Bit Attention for Plug-and-play Inference Acceleration}, 
      author={Jintao Zhang and Haofeng Huang and Pengle Zhang and Jia Wei and Jun Zhu and Jianfei Chen},
      year={2024},
      eprint={2411.10958},
      archivePrefix={arXiv},
      primaryClass={cs.LG},
      url={https://arxiv.org/abs/2411.10958}, 
}
```
