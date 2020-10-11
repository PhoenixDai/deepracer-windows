# Deepracer Windows
This repo provides some instructions on training deepracer locally on Windows. It's based on the amazing work the deepracer community has done, especially the [deepracer](https://github.com/aws-deepracer-community/deepracer) by Chirs and [deepracer-for-dummies](https://github.com/aws-deepracer-community/deepracer-for-dummies).

## Prerequisites
- Windows 10 PC with Nvidia GPU 
  - I only have Nvidia GPUs. AMD GPUs should also work in a similar way if you can set up TensorFlow properly for your AMD GPU. 
  - If you in Windows insider preview channel with build 20150+, you can follow the instructions to enable [GPU support for WSL](https://docs.microsoft.com/en-us/windows/win32/direct3d12/gpu-cuda-in-wsl).
  - If you are not planning to use GPU, you probably don't need to follow my instructions as CPU training on Windows can be done in the same way of [deepracer-for-dummies](https://github.com/aws-deepracer-community/deepracer-for-dummies).
- [Visual C++ build tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)
  - It seems only the ```annoy``` package, a dependency of coach, need to get compiled when installing. If you can find a pre-compiled binary somewhere, you probably don't need to install the compiler. 
- [Docker on Windows](https://www.docker.com/products/docker-desktop)
- Anaconda is optional but it could save you a lot of time by installing CUDA, etc. automatically.
- Get a local copy of the [deepracer](https://github.com/aws-deepracer-community/deepracer) repo.