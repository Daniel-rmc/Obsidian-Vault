git check out icra 失败时候：由于gitclone 时候指定了 --depth=1：git clone --depth=1 https://github.com/LejuRobotics/kuavo_data_challenge.git 
导致 git fecth all 看不到icra分支，只有main分支。
使用如下命令可以解决：
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
环境配置好后，commit成了新的镜像：docker commit 97b6c26dec65 my_kdc_icra_image

docker run -it \
  --name kdc_icra \
  --gpus all \
  --net host \
  --privileged \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -e DISPLAY=$DISPLAY \
  -v /home/rmc/workspace:/workspace \
  my_kdc_icra_image:latest \
  /bin/bash

`pip install torch==2.9.0 torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128`

进入镜像后 配置hf mirror

`huggingface-cli download kuavo_data_challenge_icra \
    --repo-type dataset \
    --include "sim/TASK1-ToySorting/*" \
    --local-dir ./TASK1_Data \
    --resume-download`
1.14 
download 了 所有数据集；
对任务一进行转换成lerobot数据。

python kuavo_data/CvtRosbag2Lerobot.py \
  --config-path=../configs/data/ \
  --config-name=KuavoRosbag2Lerobot.yaml \
  rosbag.rosbag_dir=/workspace/datasets/task1_first400 \
  rosbag.lerobot_dir=/workspace/datasets/lerobot_data/task1

单机多卡加速训练：
accelerate launch \
--config_file configs/accelerate/accelerate_config.yaml \
kuavo_train/train_policy_with_accelerate.py  \
--config-path=../configs/policy \
--config-name=act_config.yaml

python kuavo_train/train_policy.py \
--config-path=../configs/policy/ \
--config-name=act_config.yaml