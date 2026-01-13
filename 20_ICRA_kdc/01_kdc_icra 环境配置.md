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
  -v /home/user/你的旧挂载目录:/container/旧目录 \
  -v /home/user/想要新增的目录:/root/new_data \
  my_kdc_icra_image:latest \
  /bin/bash

