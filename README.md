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

## Minio
Download the binary from [Minio](https://min.io/download#/windows) and put it somewhere you're okay with having large files. For example:
```cmd
set MINIO_ACCESS_KEY=minio
set MINIO_SECRET_KEY=miniokey
minio.exe server D:\Data
```

You will need to create a bucket (with the '+' sign at lower right corner of the web interface) named bucket through the web GUI that minio provides, just open http://127.0.0.1:9000 in your browser.

Then copy the folder custom_files from **deepracer** into your new bucket as that's where the defaults expect them to be.

## Redis
Redis server can be started with docker:
```cmd
docker run --name redis --rm -p 6379:6379 -it redis
```

## Coach on Windows
[Coach](https://nervanasystems.github.io/coach/) is package that does the training work behind SageMaker. So, after I figured out how to train with coach directly, I think it's probably easier to get started with coach directly. Althrough the docs of coach says it's only been tested on Ubuntu, it actually also works on Windows almost out of the box. 

Let's create a virtual envrionment for coach first. (I'll use Anaconda here, but pip should also work.) **Please Note:** the highest version of Python that supported by both coach and coach compatible TensorFlow is 3.7.9. So, make sure you specify the Python version here to avoid getting incompatible Python installed. 
```cmd
conda create -n coach python=3.7.9
conda activate coach
```

Now, install TensorFlow (The highest version compatible with coach is 1.14.0)
```cmd
conda install tensorflow-gpu=1.14.0
```

Then, coach can be installed with ```pip```. (It't not available with Anaconda.)
```cmd
pip3 install rl_coach
conda install boto3
conda install fsspec
```

## Modify the trainer for Windows
If you had ever trained deepracer locally on Ubuntu, you may notice the ```sourcedir.tar.gz``` file shared between SageMaker and RoboMaker containers. It is the same RL source code that used by both sides. 
- For SageMaker, it takes the states from RoboMaker to train with the RL algorithm. This is the part that's very computational intensive and relies on GPUs the accelerate the process. Since docker on Windows has no access to GPUs so far, we need to run this part locally on Windows. I'll explain how to do this step by step in the sections below. 
- For RoboMaker (the one with Gazebo), it takes the trained RL algorithm from SageMaker and run simulations for the specified iterations. This part doesn't need GPU, so this part is the same with Chris's instruction. 

The following parts are based on the code in ```sourcedir.tar.gz```. I think it comes from the ```rl_coach``` folder in the ```deepracer``` repo by Chris. The ```sourcedir.tar.gz``` is the original file I take from the minio bucket. The code in source folder is what I modified to make it runable on Windows locally. (You don't need to do the follwoing subsections again if you'd like to use my version. But if you ran into issues, these instructions may be helpful for debugging.)

### Change they way s3 client build keys for minio
In ```s3_client.py```, ```s3_boto_data_store.py``` and ```training_worker.py```, Python's ```os.path.normpath``` was used to build the keys for unload/download files to/from minio server. However, it will use ```\\``` to replace ```/``` in the keys, which in incompatible with minio's way of path to files. So, I simply skipped ```os.path.normpath``` and build the key strings direstly with ```/```. (See commit [877325c](https://github.com/PhoenixDai/deepracer-windows/commit/877325c8a7389b210c7b9fe0f2c2b55bf009de67) and [aea76ec](https://github.com/PhoenixDai/deepracer-windows/commit/aea76ec2875caed9ee9596fc21d8113768998af9) for the changes I made.) Please feel free to advise if you have better ways to handle this issue. 

### Get hyperparameter from file pointed by SM_TRAINING_ENV
I didn't spend much time on figuring out how hyperparameters were passed from SageMaker to rl_coach. I think it would be more straight forward to load from the file pointed by ```SM_TRAINING_ENV```. (See commit [0751522](https://github.com/PhoenixDai/deepracer-windows/commit/0751522717a48ba080211143802316469a925a0f) for details.)

### Lower the data requesting frequency
It seems the network speed is lower in Windows than Ubuntu. So the data requesting frequency in ```deepracer_memory.py``` need to be set to a lower value. So far, ```time.sleep(1000*POLL_TIME)``` works very well. (See commit [3209933](https://github.com/PhoenixDai/deepracer-windows/commit/3209933c58dc082d9d584adabd4cab502a58b57f) for details.)

## Let's Go
### One more thing
Now we are ready (almost) to train deepracer locally. The only thinkg left is to create some folders that the programs need to use. 
- In the ```bucket``` folder, create a sub-folder ```custome_files```, and place the ```model_metadata.json``` and ```reward.py``` file of your choice over there. If you don't have one, you can get the default ones from ```deepracer``` repo. 
- In the ```bucket``` folder, create a sub-folder ```rl-deepracer-sagemaker```. In ```rl-deepracer-sagemaker```, create sub-folder ```ip``` and ```model```.
- In the ```bucket``` folder, create a sub-folder ```rl-deepracer-pretrained```. In ```rl-deepracer-pretrained```, create a sub-folder ```model```. Place a pretrained model here. If you don't have one, you can use the one [rl-deepracer-pretrained.zip](https://www.dropbox.com/s/f1vr9hetin6650g/rl-deepracer-pretrained.zip?dl=0) I shared. (If no pretrained model is provided, the RoboMaker won't start. It's strange but it's not a big issue for me, so I didn't spend time to debug it.)
- Create a folder for RoboMaker data. I'll just use ```D:\\data\robo```. It will be used to share files between your PC and RoboMaker container. 
- In the ```robo``` foler, create a sub-folder ```checkpoint```.
- Create a temporary data folder. I'll use ```D:\\data\run```. It will be used by the trainer program.

### Start the trainer
Open a Windows command line window and set some environment variables:
```cmd
set SAGEMAKER_TRAINING_MODULE=sagemaker_tensorflow_container.training:main
set AWS_ACCESS_KEY_ID=minio
set AWS_SECRET_ACCESS_KEY=miniokey
set AWS_REGION=us-east-1
set TRAINING_JOB_NAME=rl-deepracer-sagemaker
set S3_ENDPOINT_URL=http://localhost:9000
set NODE_TYPE=SAGEMAKER_TRAINING_WORKER
set SM_TRAINING_ENV=D://data/hyperparameters.json
set ALGO_MODEL_DIR=D://data/run/model
```
Then, go to ```D:\\data\run```. (I'll just use the one I'm using as an example. Same for the folder being used below.) Now, you can run the command below to start your trainer:
```cmd
python D:\\source\training_worker.py --RLCOACH_PRESET deepracer --aws_region us-east-1 --model_metadata_s3_key s3://bucket/custom_files/model_metadata.json --pretrained_s3_bucket bucket --pretrained_s3_prefix rl-deepracer-pretrained --s3_bucket bucket --s3_prefix rl-deepracer-sagemaker
```

### Start the simulator (RoboMaker)
This part should be the easiest. Just open another Windows command line window, go to the folder of your local copy of ```deepracer```, update the environment variables in ```robomaker.env``` accordingly and run:
```cmd
docker run --rm --name dr --env-file ./robomaker.env -p 8080:5900 --cpus "6" -v D://deepracer/simulation/aws-robomaker-sample-application-deepracer/simulation_ws/src:/app/robomaker-deepracer/simulation_ws/src -v D://data/robo/checkpoint:/root/.ros/ -it crr0004/deepracer_robomaker:console "./run.sh build distributed_training.launch"
```

## Enjoy!