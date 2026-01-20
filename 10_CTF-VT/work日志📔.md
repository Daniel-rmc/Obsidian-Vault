```bash
通过自己电脑代理服务器访问外网
export http_proxy="http://192.168.1.159:7890"
export https_proxy="http://192.168.1.159:7890"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"

export http_proxy="http://10.29.109.77:7897"
export https_proxy="http://10.29.109.77:7897"
export HTTP_PROXY="$http_proxy"
export HTTPS_PROXY="$https_proxy"

即使终端进程走了代理，但是docker守护的进程仍然不走代理，可以用下面命令临时指定：
sudo HTTP_PROXY="http://192.168.1.142:7897" HTTPS_PROXY="http://192.168.1.142:7897" docker build -t lerobot:latest .

清理ssh缓存：
ssh-keygen -R 192.168.1.101

激活容器lerobot 环境：
source lerobot_venv/bin/activate

在容器内激活：
source /root/lerobot_venv/bin/activate

进入容器
docker exec -it lerobot_rmc bash -c "source /root/lerobot_venv/bin/activate && exec bash"

升级安装lerobot的策略：
pip install -e ".[smolvla]" --no-deps
指定 --no-deps 可以忽略安装依赖，避免更新已经适配好的环境（尤其是pytorch版本，官方依赖中的不适用我们的显卡）

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
 
 
sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
 
sudo apt-get update
 
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.17.8-1
  sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

## 10.9:
问题：

- [x] cuda 12.9+lerobot + conda
- [x] docker 网络问题，终端进程代理后，docker 内部仍需要进行代理，同时即使是在 docker 外部运行 docker  build 之类的命令，也需要重新进行代理。
- [ ] 服务器无图形界面 gui ，无法可视化数据集；先不管。
- [x] 服务器网络问题：每次用自己的电脑代理一下



- [x] 针对 local_files_only 字段实效导致的训练 lerobot/diffusion_pusht 失败，由于 hub 上的版本已落后，lerobot 的版本更新啦。local_files_only 字段已被删除。

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1759998460518-1deb196a-cad2-49f7-9064-66fa459c43c4.png)

解决办法：[https://github.com/huggingface/lerobot/issues/864](https://github.com/huggingface/lerobot/issues/864) 

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1759998714813-2faecb01-1bc5-458d-8dcf-0d183fae24d7.png)

删除该行可解决。

- [x] lerobot 训练代码：
1. 训练时候需要指定 repo	_id：lerobot-train --config_path=lerobot/diffusion_pusht --policy.repo_id MinchiRuan/lerobot-pusht 
2. 只想在本地保存、不上传：lerobot-train --config_path=lerobot/diffusion_pusht --policy.repo_id local/lerobot-model

```bash
lerobot-train --config_path=lerobot/diffusion_pusht --policy.repo_id MinchiRuan/lerobot-pusht

lerobot-train --config_path=lerobot/diffusion_pusht --policy.repo_id local/lerobot-model
```



- [x] 显卡 RTX PRO 6000 Blackwell 和当前 pytorch 版本不兼容

1.cuda 12.9 + torch 最新的 nightly 版本 + 驱动 580.95 能够运行。



## 10.10:
- [x] 修复新装的系统，删除旧系统残留，挂载 home 到独立新分区。

```bash
sudo bash -c "
# 1️⃣ 备份当前 /home 数据
cp -a /home /home_backup && \
echo '备份完成，路径为 /home_backup' && \

# 2️⃣ 格式化 /dev/sda4 为 ext4（会清空 sda4 数据）
mkfs.ext4 /dev/sda4 && \
echo '/dev/sda4 格式化为 ext4 完成' && \

# 3️⃣ 创建临时挂载点并挂载 /dev/sda4
mkdir -p /mnt/home && \
mount /dev/sda4 /mnt/home && \
echo '临时挂载完成' && \

# 4️⃣ 复制 /home 数据到 /mnt/home
rsync -av /home/ /mnt/home/ && \
echo '/home 数据复制完成' && \

# 5️⃣ 获取 /dev/sda4 UUID
UUID=\$(blkid -s UUID -o value /dev/sda4) && \
echo '/dev/sda4 UUID: ' \$UUID && \

# 6️⃣ 添加 fstab 条目，实现开机自动挂载
grep -q \$UUID /etc/fstab || echo \"UUID=\$UUID /home ext4 defaults 0 2\" >> /etc/fstab && \
echo '/etc/fstab 更新完成' && \

# 7️⃣ 卸载临时挂载并挂载 /home
umount /mnt/home && \
mount /home && \
echo '/home 已挂载到 /dev/sda4' && \

# 8️⃣ 查看挂载情况
df -h /home
"





```

```bash

echo "=== 分区挂载概览 ===" && \
lsblk -f && \
echo && \
echo "=== 磁盘使用情况 ===" && \
df -hT && \
echo && \
echo "=== 当前挂载点详细信息 ===" && \
mount | column -t && \
echo && \
echo "=== /etc/fstab 配置 ===" && \
grep -v '^#' /etc/fstab | grep -v '^$'
```

- [x] 复现 lerobot 框架，确认在 RTX pro6000 显卡（显卡太新了，需要支持 sm_120, 只能使用最新版 torch nightly 版本，ubuntu24.0 系统下，兼容的环境配置。cuda12.9，pytorch2.9nightly 版本。）

遗留问题：目前没有影响到运行，可能没用到。

- [x] 又是 pytorch 2.7。与显卡不兼容，显卡太新，安装的 requirement 里的 torch 版本是 2.7 带 cuda12.6，不支持 sm_120，重新尝试更新 pytorch。 

```bash
pip install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/cu129
```

### 版本兼容性警告
```plain
lerobot 0.3.4 requires torch<2.8.0,>=2.2.1, but you have torch 2.9.0.dev20250901+cu129 which is incompatible.
lerobot 0.3.4 requires torchvision<0.23.0,>=0.21.0, but you have torchvision 0.24.0.dev20250901+cu129 which is incompatible.
```

+ 你现在的 **lerobot 0.3.4** 项目对 PyTorch 和 torchvision 的版本有限制。
+ 也就是说，虽然 PyTorch 可以使用你的 RTX PRO 6000 Blackwell GPU，但 **lerobot 的代码可能不兼容 2.9 nightly 版本**，可能会报错或 API 不匹配。



- [x] 成功运行 diffusion_pusht 训练样例。

lerobot-train --config_path=lerobot/diffusion_pusht --policy.repo_id MinchiRuan/lerobot-pusht-test

INFO 2025-10-10 14:27:23 ot_train.py:299 step:400 smpl:26K ep:206 epch:1.00 loss:0.078 grdn:2.610 lr:6.0e-05 updt_s:0.044 data_s:0.220

INFO 2025-10-10 14:28:16 ot_train.py:299 step:600 smpl:38K ep:308 epch:1.50 loss:0.071 grdn:1.967 lr:9.5e-05 updt_s:0.042 data_s:0.223

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1760077838117-f585fe7c-9f25-49e1-b3b7-d8e56a1195ab.png)



好用的命令行工具：oh my zsh

## 10.11 修复电脑：
重装电脑，确认必须要 ubuntu 24，然后 Nvidia 驱动 580.95.05，必须按照飞书里的文档命令行安装成功所有的版本。

- [x] 复现 lerobot 训练 diffusion 
- [x] 确认了各种环境的配置，并打包制作成了 lerobot_team 镜像。同时压缩了一个镜像压缩包lerobot_team_v1.tar 备份。

## 10.13:
挂载数据盘 到 /data；将 lerobot_team_v1.tar 上传 data 盘进行备份。

设置/data 目录 权限组 datausrs，并开启继承。

- [x] 数据盘 挂载，只让指定用户开放。已设置 datausers 组，

```bash
1️⃣ 创建挂载点并挂载数据盘
# 创建挂载点
sudo mkdir -p /data

# 挂载数据盘（假设数据盘为 /dev/sdb）
sudo mount /dev/sdb /data

# 查看挂载情况
df -h | grep /data

# 查看 UUID
sudo blkid /dev/sdb

# 添加到 /etc/fstab，实现开机自动挂载
sudo vim /etc/fstab

# 在 fstab 中添加一行：
# UUID=11d4973e-826c-4330-bc50-a7252e941159 /data ext4 defaults 0 2

# 测试 fstab 配置
sudo mount -a

2️⃣ 创建共享组
# 创建新组
sudo groupadd datausers

3️⃣ 将用户加入共享组
# 把已有用户加入 datausers
sudo usermod -aG datausers rmc
sudo usermod -aG datausers deeptouch
sudo usermod -aG datausers max
sudo usermod -aG datausers txy

# 验证组成员
getent group datausers


⚠️ 用户加入新组后需要重新登录 SSH 才能生效。

4️⃣ 修改 /data 目录权限
# 修改目录所有者和组
sudo chown -R root:datausers /data

# 设置目录权限，组可读写执行
sudo chmod -R 770 /data

# 设置 setgid，保证新建文件自动继承组
sudo chmod g+s /data

# 查看权限确认
ls -ld /data
```

- [x] 跳过上传模型到 huggingface 进行训练

```bash
# if self.policy.push_to_hub and not self.policy.repo_id:
#     raise ValueError(
#         "'policy.repo_id' argument missing. Please specify it to push the model to the hub."
#     )
if self.policy.push_to_hub and not self.policy.repo_id:
    import logging
    logging.warning("'policy.repo_id' missing → disabling push_to_hub automatically.")
    self.policy.push_to_hub = False
```

[Lerobot框架使用（含本地数据训练）_lerobot机械臂 本地训练huggingface-CSDN博客](https://blog.csdn.net/weixin_74181752/article/details/148314386)

- [x] 训练完的模型放在每个文件的 checkpoint 上，在 outputs 文件夹下

放在 outputs 文件夹下存放的 chekpoint 检查点，会自动加载模型的各种超参数配置等等。

- [x] 训练使用的数据使用本地，

dataset.root 指定数据集所在位置。

- [x] 跑本地数据和策略并保存模型文件到本地的流程：
- [x] 指定本地数据集所在目录： --dataset.root=./data \

记录 diffusion policy 训练的所需时间：

- [x] 跑100000 steps，总共计时： 462min 。
- [x] 跑 smalVLA，模型。

--policy.path=lerobot/smolvla_base \

--policy.type=(这个是从头训练，和上面的互斥，上面的使用预训练好的模型训练）

## 10.14:
- [x] 开源数据集下载，并进行开源算法进行调试：

[https://huggingface.co/datasets/agibot-world/AgiBotWorld-Alpha/tree/main](https://huggingface.co/datasets/agibot-world/AgiBotWorld-Alpha/tree/main)

- [x] 国内源：
- [ ] <font style="color:rgb(0, 0, 0);">pip install modelscope</font>

```bash
modelscope download --dataset agibot-world/AgiBotWorld-Alpha --local_dir /data/
```

[https://www.modelscope.cn/datasets/agibot-world/AgiBotWorld-Alpha](https://www.modelscope.cn/datasets/agibot-world/AgiBotWorld-Alpha)

下载总速度：8 个任务，4M/s，32M/s 最大速度，然后 10T 大概需要下载 86 个小时，3.7 天左右。

- [x] 确认一下 智元的代码数据格式转换是转成 V2.1 还是 V2.0。由于官方只给出了从 V2.1 转到 V3.0 的脚本。

智元开源数据库主页 readme 说是提供了转成 V2.0 的版本，但是我贴出源代码问了一下 gpt5.0，它分析源代码的结构，好像其实是转成 V2.1，需要试验一下，确认一下。

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1760430686319-e137d68c-2a9c-4dfb-931a-34ca28d19f82.png)

如果确认 Agibot 数据集是转 V2.0 的话，我查找 github 上的提交记录，发现有一个提交记录上提到了一个从 v20_to_v21 的代码，可以用这个代码去转换成 V2.1。

[https://github.com/huggingface/lerobot/pull/711](https://github.com/huggingface/lerobot/pull/711)

```bash
python lerobot/common/datasets/v21/convert_dataset_v20_to_v21.py \
    --repo-id=repo/id
```

流程可能是需要先转 V2.1，再转 V3.0：

```bash
python src/lerobot/datasets/v30/convert_dataset_v21_to_v30.py \
    --repo-id my_local/my_robot_dataset_v21 \
    --root /data/my_robot_dataset_v21 \
    --out-dir /data/my_robot_dataset_v30
```

- [ ] 给服务器配置网络（连接外网）：明天处理
- [x] 配置了服务器 tmux，让我的 ssh 断开后，仍然能够持续训练，并给容器也配置成了 dit 模式，后台运行不会突然中断。

```bash
docker system info | grep -i DetachKeys
rmc@deeptouch-NF5468-M7-A0-R0-00:~$ mkdir -p ~/.docker
cat > ~/.docker/config.json <<'EOF'
{
  "detachKeys": "ctrl-x,ctrl-x"
}
EOF
```

- [x] 从 huggingface 上下载数据集 svla_so100_pickplace

```bash
hf download lerobot/svla_so100_pickplace \
  --repo-type dataset \
  --revision v3.0 \
  --local-dir /data/svla_so100_pickplace

hf download lerobot/pi05_libero_finetuned \
--repo-type model \
--local-dir /workspace/lerobot/model_saved \
--max-workers 8
```

- [ ] 使用本地数据进行训练，报错 transformers 有错
- [x] 更新了两个小功能，已经 push 到 git 上了，根据当前验证配置好的环境 进行 requirement 的自动更新，方便后续迭代更新版本而不破坏现有环境。

```bash
# 删除 wandb 登录凭据
rm -f ~/.netrc
rm -rf ~/.config/wandb

# 删除 huggingface 登录凭据
rm -f ~/.huggingface/token
rm -rf ~/.cache/huggingface
```



pip install -e ".[smolvla]" --no-deps

```bash
export HF_ENDPOINT=https://hf-mirror.com

CUDA_VISIBLE_DEVICES=1 HF_ENDPOINT=https://hf-mirror.com \
python src/lerobot/scripts/lerobot_train.py \

```

```bash
HTTP_PROXY="http://192.168.1.142:7897" HTTPS_PROXY="http://192.168.1.142:7897"
CUDA_VISIBLE_DEVICES=1 \
lerobot-train \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=lerobot/svla_so100_pickplace \
  --dataset.root=/data/svla_so100_pickplace \
  --dataset.streaming=true \
  --policy.repo_id=local_smolvla_policy \
  --policy.push_to_hub=false \
  --policy.device=cuda \
  --wandb.enable=true \
  --batch_size=64 \
  --num_workers=4 \
  --steps=20000 \
  --output_dir=outputs/train/my_smolvla \
  --job_name=my_smolvla_training

lerobot-train \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=lerobot/svla_so101_pickplace \
  --dataset.streaming=true \
  --num_workers=0 \
  --batch_size=8 \
  --steps=20000 \
  --output_dir=outputs/train/my_smolvla_$(date +'%Y%m%d_%H%M%S') \
  --job_name=my_smolvla_training \
  --policy.device=cuda \
  --wandb.enable=true

  #注释掉lerobot_train.py 中的： # prefetch_factor=2,并将--num_workers=0设置为0，可以正常训练。
      dataloader = torch.utils.data.DataLoader(
        dataset,
        num_workers=cfg.num_workers,
        batch_size=cfg.batch_size,
        shuffle=shuffle and not cfg.dataset.streaming,
        sampler=sampler,
        pin_memory=device.type == "cuda",
        drop_last=False,
        # prefetch_factor=2,
    )

```

## 10.15
### debug:
<font style="color:rgba(6, 8, 31, 0.88);"> OpenCV 打不开数据集格式中的 AV1 视频：</font>

- [x] 更新了新的 ffmpeg，然后通过验证，提交成新的镜像 lerobot_team 啦，并打包成 tar 备份至/data 。
- [x] 容器共享内存 smh 默认为 64M，太小了，可能使得 dataloader 交换时候内存溢出，启动容器时候，指定交换内存为 16G 或者更大，这样可以大一些 batchsize。运行容器时，设置大的共享内存

```bash
docker run -dit \
  --name (你想创建的docker name)lerobot_pi0 \
  --gpus all \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v /home/“usrname”/lerobot:/workspace/lerobot \
  -v /data:/data \ #这一行需要挂载/data盘
  (images name)lerobot:latest
```



提了一个 issue 到 github 和 huggingface lerobot 社区啦：

[std::bad_alloc / DataLoader worker exited unexpectedly during training with smolvla_base · Issue #2209 · huggingface/lerobot](https://github.com/huggingface/lerobot/issues/2209)

<font style="color:rgb(31, 35, 40);">Set num_workers=0 and comment out the prefetch_factor=2 in lerobot_train.py:</font>  
<font style="color:rgb(31, 35, 40);">This prevents the error, and training starts successfully, but the training runs significantly slower.</font>

```bash
python3 src/lerobot/scripts/lerobot_train.py \  
--policy.path=lerobot/smolvla_base \
--dataset.repo_id=lerobot/svla_so101_pickplace \
--dataset.streaming=true \ 
--num_workers=4 \
--batch_size=64 \
--steps=20000 \
--output_dir=outputs/train/my_smolvla_$(date +'%Y%m%d_%H%M%S') \
--job_name=my_smolvla_training \
--policy.device=cuda \
--wandb.enable=true

```



但是特别特别慢，现在跑了 120 分钟 还没有开始出现 train 的日志。

- [x] 注意到一个 issue 里讨论：[The actions of SmolVLA. · Issue #2157 · huggingface/lerobot](https://github.com/huggingface/lerobot/issues/2157) SmolVLA 使用的是 end effector pose。
- [ ] [https://huggingface.co/datasets/HuggingFaceVLA/libero](https://huggingface.co/datasets/HuggingFaceVLA/libero) 这是模型微调使用的数据集，数据集 data
- [ ] [Fine-tuned π0 and π0.5 models fail to replicate reported success rates on LIBERO benchmark · Issue #2114 · huggingface/lerobot](https://github.com/huggingface/lerobot/issues/2114)

<font style="color:rgb(31, 35, 40);">Does SmolVLA output actions in joint space coordinates, or in end-effector pose (position + orientation + gripper state)? 回复：end effector pose :)</font>

## 10.16:
1. 等服务器跑起来，先把下载数据的任务挂起来，让它下着数据。
2. 找一下其他的模型，小的数据集，进行本地训练测试。

熟悉 lerobot 数据结构：

- [ ] 训练 pi-0 在svla_so101_pickplace 数据集上

```bash
python3 src/lerobot/scripts/lerobot_train.py \
    --dataset.repo_id=lerobot/svla_so101_pickplace \
    --policy.type=pi0 \
    --output_dir=outputs/pi0_training \
    --job_name=pi0_so101_pickplace \
    --policy.pretrained_path=lerobot/pi0_base \
    --policy.compile_model=true \
    --policy.gradient_checkpointing=true \
    --policy.dtype=bfloat16 \
    --steps=3000 \
    --policy.scheduler_decay_steps=3000 \
    --policy.device=cuda \
    --batch_size=32 \
    --wandb.enable=true
```

scp /data/lerobot_20251015.tar henry@192.168.1.69:/home/henry

```bash
torch-2.9.0.dev20250901+cu129
```

```bash
docker run -dit --gpus all \
  --name lerobot_pi0_run \
  --shm-size=32g \
  -v /data:/data \
  -v /workspace/lerobot:/lerobot \
  lerobot_pi0:latest

docker run -dit --gpus all \
  --name rmc_lerobot \
  --shm-size=32g \
  --network host \
  --restart unless-stopped \
  -v /data:/data \
  -v /home/rmc/workspace:/workspace \
  -v /ssd_cach:/ssd_cach \
  lerobot_pi05:latest
```



## todolist：
10.20-10.23: 调研 40 篇文献，总结 20 篇。

- [x] 看文章调研一下，单任务的指标汇总，找到一个好的 baseline 基于它开发,装配有关的大方向，长程一点的。

[VLA 文献调研](https://www.yuque.com/u22131060/bvddol/xif627zz0ged2gsz)

## <font style="color:rgb(0, 0, 0);">10.23 </font>
分享文献调研结果，深入挖掘以下五个问题。

## <font style="color:rgb(0, 0, 0);"> todo list：</font>
<font style="color:rgb(0, 0, 0);">1.SmolVla异步推理是怎么部署？是哪些部署在服务器端，哪些部署在机器人端？这之间是怎么通信的？</font>

[论文调研QA](https://www.yuque.com/u22131060/bvddol/hd2hhxvbwerc4xn4)

<font style="color:rgb(0, 0, 0);">2.octo 加入的本体感受是什么？为什么会更差？</font>

+ <font style="color:rgb(0, 0, 0);">proprioceptive information：关节角度、关节速度、末端执行器位姿、夹爪状态、力-扭矩读数（低维向量输入）</font>
+ <font style="color:rgb(0, 0, 0);">作者认为：这可能是由于本体感受信息和目标动作之间的因果混淆造成的。</font>**<font style="color:rgb(0, 0, 0);">因果混淆（Causal Confusion）：</font>**<font style="color:rgb(0, 0, 0);">本体状态（如当前关节角度）与未来动作存在强相关性，导致策略错误地将当前状态视为动作结果的因（而非动作执行的果）。</font>
+ <font style="color:rgb(0, 0, 0);">结合可能是</font><font style="color:rgb(0, 0, 0);">在</font>**<font style="color:rgb(0, 0, 0);">跨平台训练</font>**<font style="color:rgb(0, 0, 0);">的初期，直接混用未经妥善处理的异构本体感受数据会给训练带来负面的作用。</font>

<font style="color:rgb(0, 0, 0);">3.AgiBot 中间 latent 层是什么？如何训练的？损失是什么？</font>

<font style="color:rgb(0, 0, 0);">4.openVLA 4-bit 是为什么没有性能损失？</font>

<font style="color:rgb(0, 0, 0);">5.smolvla 提示词优化怎么弄的？</font>

- [ ] 训练几个基础模型，制作几个预训练模型 pi-0.5

pre-model:已完成的有 ：

    - [x] 1.diffusion_pushT
    - [ ] 2.act
    - [ ] 3.smovla
    - [ ] 4.pi-0

 从 act 到 pi-0，smolvla 训练都遇到以下问题：[std::bad_alloc / DataLoader worker exited unexpectedly during training with smolvla_base · Issue #2209 · huggingface/lerobot](https://github.com/huggingface/lerobot/issues/2209)

- [ ] 增加数据的摄像头数，如何修改 input 格式。
- [ ] 可以试试用其他数据集+diffusion。

训练策略： ACT， diffusion 推荐

- [x] 下载智元开源的数据集。
- [x] 熟悉一下 lerobot 的格式。
- [ ] 如何修改 action 标签。



## 10.27
1.计划从 Nvidia cuda，ubuntu24 开始建立镜像

```bash
accelerate launch --num_processes 2 \
    --mixed_precision bf16 \
    --main_process_port 29500 \
src/lerobot/scripts/lerobot_train.py \
    --dataset.repo_id=lerobot/svla_so101_pickplace \
    --dataset.root=/data/svla_so100_pickplace \
    --policy.type=pi05 \
    --output_dir=./outputs/pi05_training \
    --job_name=pi05_training \
    --policy.repo_id=rmc_pi05_1027 \
    --policy.pretrained_path=lerobot/pi05_libero_base \
    --policy.compile_model=true \
    --policy.gradient_checkpointing=true \
    --wandb.enable=true \
    --policy.dtype=bfloat16 \
    --steps=3000 \
    --policy.scheduler_decay_steps=1000 \
    --policy.device=cuda \
    --batch_size=128 \
    --policy.normalization_mapping='{"ACTION": "MEAN_STD", "STATE": "MEAN_STD", "VISUAL": "IDENTITY"}' \
    --policy.push_to_hub=False

```

2.尝试重新构建 ubuntu22.4，cuda12.8 的镜像，通过配置跟官方相同版本的进行

或者按照 issue#2305 的配置 

```bash
- lerobot version: 0.4.0
- Platform: Linux-6.14.0-29-generic-x86_64-with-glibc2.39
- Python version: 3.12.12
- Huggingface Hub version: 0.35.3
- Datasets version: 4.1.1
- Numpy version: 2.2.6
- PyTorch version: 2.7.0+cu128
- Is PyTorch built with CUDA support?: True
- Cuda version: 12.8
- GPU model: NVIDIA RTX PRO 6000 Blackwell Workstation Edition
- Using GPU in script?: <fill in>
```

3.从官网上下载模型数据。

pytorch 环境是： pip install torch==2.9.0+cu128 torchvision==0.24.0+cu128 torchaudio==2.9.0 --index-url [https://download.pytorch.org/whl/cu128](https://download.pytorch.org/whl/cu128)

## 10.28
### 解决了之前的 bug：
[std::bad_alloc / DataLoader worker exited unexpectedly during training with smolvla_base · Issue #2209 · huggingface/lerobot](https://github.com/huggingface/lerobot/issues/2209)

通过--dataset.video_backend pyav \ 指定视频 video_backend pyav 可以解决。

罪魁祸首 torchcodec 版本和 torch 版本不一致。

```bash
(lerobot) root@0981f4728167:/workspace/lerobot# pip install torchcodec
Requirement already satisfied: torchcodec in /opt/miniconda3/envs/lerobot/lib/python3.12/site-packages (0.5)
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
(lerobot) root@0981f4728167:/workspace/lerobot# python -c "import torch, torchcodec; print('torch:', torch.__version__); print('torchcodec:', torchcodec.__version__)"
terminate called after throwing an instance of 'std::bad_alloc'
  what():  std::bad_alloc
Aborted (core dumped)
(lerobot) root@0981f4728167:/workspace/lerobot# 
```

<font style="color:rgb(31, 35, 40);">It has been confirmed that the bug originates from an ABI instability caused by version incompatibility between TorchCodec and PyTorch, triggering underlying C++ code errors.</font>

<font style="color:rgb(31, 35, 40);">Solution:  
</font><font style="color:rgb(31, 35, 40);">Upgrade</font><font style="color:rgb(31, 35, 40);"> </font>**<font style="color:rgb(31, 35, 40);">TorchCodec to v0.8</font>**<font style="color:rgb(31, 35, 40);"> to ensure compatibility with</font><font style="color:rgb(31, 35, 40);"> </font>**<font style="color:rgb(31, 35, 40);">PyTorch 2.9</font>**<font style="color:rgb(31, 35, 40);">.  
</font><font style="color:rgb(31, 35, 40);">OR  
</font><font style="color:rgb(31, 35, 40);">Explicitly specify</font><font style="color:rgb(31, 35, 40);"> </font>`<font style="color:rgb(31, 35, 40);background-color:rgba(129, 139, 152, 0.12);">--dataset.video_backend pyav</font>`<font style="color:rgb(31, 35, 40);"> </font><font style="color:rgb(31, 35, 40);">to bypass TorchCodec for video decoding.  
</font><font style="color:rgb(31, 35, 40);">Recommendation:</font>

<font style="color:rgb(31, 35, 40);">We strongly advise using TorchCodec due to its </font>**<font style="color:rgb(31, 35, 40);">GPU-accelerated decoding capabilities</font>**<font style="color:rgb(31, 35, 40);">, which deliver significantly faster video processing compared to CPU-based solutions like PyAV.</font>



遇到 issue 2189 的问题。

[https://github.com/huggingface/lerobot/issues/2189](https://github.com/huggingface/lerobot/issues/2189)

通过指定    --policy.normalization_mapping='{"ACTION": "MEAN_STD", "STATE": "MEAN_STD", "VISUAL": "IDENTITY"}' 可以解决。

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1761616068716-677379cd-6476-46c7-9c60-817b969b8ba7.png)

### 测试训练好的模型：
使用 libero 对训练好的模型进行测试。

```plain
python src/lerobot/scripts/lerobot_train.py \
  --policy.path=lerobot/smolvla_base \
  --dataset.repo_id=lerobot/svla_so101_pickplace \
  --batch_size=64 \
  --steps=20000 \
  --output_dir=outputs/train/my_smolvla \
  --job_name=my_smolvla_training \
  --policy.device=cuda \
  --wandb.enable=true

```

```plain
python - <<'PY'
import sys, inspect
sys.path.append("src")

from lerobot.datasets.lerobot_dataset import LeRobotDataset

# 查看这个类的 __init__ 签名，帮我们判断可传哪些参数
sig = inspect.signature(LeRobotDataset.__init__)
print("🔍 LeRobotDataset.__init__ signature:", sig)

# 尝试使用最简构造
try:
    ds = LeRobotDataset("lerobot/svla_so101_pickplace")
except TypeError as e:
    print("⚠️ 直接传 string 报错:", e)
    ds = LeRobotDataset(repo_id="lerobot/svla_so101_pickplace")

print("✅ Dataset instance created.")
sample = ds[0]
print("Top-level keys:", list(sample.keys()))

if "observation" in sample:
    print("\nObservation keys:")
    for k in sample["observation"].keys():
        print(" -", k)
else:
    print("\nFlattened observation keys (observation.*):")
    for k in sample.keys():
        if k.startswith("observation."):
            print(" -", k)
PY
```

## 10.29
```bash
python - <<'PY'
import torch, torchcodec
print("torch:", torch.__version__)
print("torchcodec:", torchcodec.__version__)
PY
#如果上述正确输出了torch版本和torchcodec版本，则说明版本互相匹配啦。
#否则使用下面命令重新安装，torchcodec，去官网查看对应的torch版本
pip uninstall -y torchcodec
pip install torchcodec==0.8 -U
pip install -U torch torchvision torchaudio
```

### 更新了 docker images
更新两个 docker images：

1.lerobot_pi05:latest(ubuntu24 cuda12.9）

2.lerobot:ubuntu22_cuda12_8

这两个镜像 lerobot 环境都配好了，使用 conda activate lerobot 激活环境。

tips: 用镜像启动容器时候，需要设置--shm-size=32g，这样才支持训练时候加载大一些的 bathsize。

#### 镜像启动命令
```bash
docker run -dit \
  --name lerobot_rmc \
  --gpus all \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v ./workspace:/workspace \
  -v /data:/data \
  -v /ssd_cach:/ssd_cach \
  lerobot_pi05:latest
```

#### 设置 huggingface 镜像 和默认缓存路径
进去容器后，可以使用 vim ~/.bashrc 看看最后几行有没有设置 huggingface 默认缓存路径，应该是有下面这两行的，如果没有添加到～/.bashrc 里。

```bash
export HF_ENDPOINT=https://hf-mirror.com
export HF_HOME=/ssd_cach/hf_cache
```

这样就可以在国内加速访问 huggingface，拉取模型权重和数据集会快很多。然后设置默认的缓存路径，由于/data 盘是挂载进来的，每次运行数据是可以保存下来的。所有人把默认路径改为这个，任何一个人从 huggingface 上下载模型和数据，都可以先从缓存里检查，别人下载过的可以直接使用，你从 huggingface 上拉取了任何模型或者数据集别人要用的时候也可以快速读取，可以减少重复拉取下载，即使有镜像有的模型和数据集还挺大的，镜像网站也限速，拉取模型和数据集还挺耗费时间的。



我发现 HHD 磁盘 io 太慢啦，如果把 huggingface 缓存的模型权重和数据等放在 HHD 将大大限制速度，IO 瓶颈将限制训练速度。我增加了目录：/ssd_cach 



### 测试模型
```bash
python -m lerobot.scripts.eval_policy \
  --config_path=eval_config.json \
  --output_dir=outputs/eval/diffusion_pick
```

```bash
{
  "env": {
    "type": "gym_aloha/AlohaPick-v0",
    "obs_type": "pixels_agent_pos"
  },
  "policy": {
    "pretrained_path": "outputs/train/diffusion_pick/checkpoints/last/pretrained_model",
    "device": "cuda"
  },
  "eval": {
    "num_episodes": 20,
    "record_video": true
  }
}
```

```bash
# Available: libero_10, libero_100, libero_90, libero_goal, libero_object, libero_spatial
lerobot-eval \
  --policy.path="/workspace/lerobot/outputs/pi05_checkpoint010000/checkpoints/010000/pretrained_model" \
  --env.type=libero \
  --env.task=libero_10 \
  --eval.batch_size=8 \
  --eval.n_episodes=20 \
  --policy.device=cuda \
  --policy.use_amp=false
```

<font style="color:rgba(6, 8, 31, 0.88);">报错：</font>

```bash
Exception ignored in: <function MjRenderContext.__del__ at 0x7ad60ade39a0>
Traceback (most recent call last):
  File "/opt/miniconda/envs/lerobot/lib/python3.10/site-packages/robosuite/utils/binding_utils.py", line 199, in __del__
    self.gl_ctx.free()
  File "/opt/miniconda/envs/lerobot/lib/python3.10/site-packages/robosuite/renderers/context/egl_context.py", line 150, in free
    EGL.eglDestroyContext(EGL_DISPLAY, self._context)
  File "src/errorchecker.pyx", line 59, in OpenGL_accelerate.errorchecker._ErrorChecker.glCheckError
OpenGL.raw.EGL._errors.EGLError: EGLError(
        err = EGL_NOT_INITIALIZED,
        baseOperation = eglDestroyContext,
        cArguments = (
                <OpenGL._opaque.EGLDisplay_pointer object at 0x7ad5f2ff7940>,
                <OpenGL._opaque.EGLContext_pointer object at 0x7ad56040cac0>,
        ),
        result = 0
)
```

<font style="color:rgba(6, 8, 31, 0.88);">EGL 在无显示环境（Docker、服务器）下最容易出问题。  
</font><font style="color:rgba(6, 8, 31, 0.88);">使用 osmesa 纯 CPU 渲染最稳定。</font>

<font style="color:rgba(6, 8, 31, 0.88);">执行：</font>

```bash
export PYOPENGL_PLATFORM=osmesa
export MUJOCO_GL=osmesa
lerobot-eval \
  --policy.type=pi05 \
  --policy.pretrained_path="/workspace/outputs/pi05_checkpoint010000/checkpoints/010000/pretrained_model" \
  --env.type=libero \
  --env.task=libero_10 \
  --eval.batch_size=4 \
  --eval.n_episodes=8 \
  --policy.device=cuda \
  --policy.use_amp=false

# GPU渲染
export MUJOCO_GL=egl
export PYOPENGL_PLATFORM=egl

lerobot-eval \
  --policy.path=./outputs/pi05_checkpoint010000/checkpoints/last/pretrained_model \
  --env.type=libero \
  --env.task=libero_10 \
  --eval.batch_size=1 \
  --eval.n_episodes=1 \
  --policy.device=cuda \
  --policy.use_amp=false \
  --env.obs_type=pixels_agent_pos6 \
  --rename_map='{"observation.images.top":"observation.images.image","observation.images.wrist":"observation.images.image2"}' \
  > /workspace/eval_debug.log 2>&1

export MUJOCO_GL=egl
export PYOPENGL_PLATFORM=egl
lerobot-eval \
  --env.type=libero \
  --env.task=libero_10 \
  --policy.path=/workspace/lerobot/model_save/pi05_libero_finetuned \
  --policy.device=cuda \
  --eval.batch_size=1 \
  --eval.n_episodes=5 \
  --env.max_parallel_tasks=1 \
  > /workspace/eval_debug.log 2>&1

```

<font style="color:rgba(6, 8, 31, 0.88);">⚙️</font><font style="color:rgba(6, 8, 31, 0.88);"> 变化说明：</font>

+ <font style="color:rgba(6, 8, 31, 0.88);">渲染后端：切换为 osmesa</font>
+ <font style="color:rgba(6, 8, 31, 0.88);">CPU-only 渲染，比较慢但不易挂掉</font>
+ <font style="color:rgba(6, 8, 31, 0.88);">缩小 batch 防止 OOM</font>
+ <font style="color:rgba(6, 8, 31, 0.88);">显式 </font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">--eval.save_video=True</font>`<font style="color:rgba(6, 8, 31, 0.88);"> 强制导出视频</font>

<font style="color:rgba(6, 8, 31, 0.88);"></font>

### <font style="color:rgba(6, 8, 31, 0.88);">10.30</font>
```bash
export MUJOCO_GL=egl
export PYOPENGL_PLATFORM=egl

  lerobot-eval \
  --env.type=libero \
  --env.task=libero_spatial,libero_object,libero_goal,libero_10 \
  --eval.batch_size=1 \
  --eval.n_episodes=10 \
  --policy.path=/workspace/lerobot/model_save/pi05_libero_finetuned \
  --policy.n_action_steps=10 \
  --policy.device=cuda \
  --env.max_parallel_tasks=1 \
  --rename_map='{"observation.images.empty_camera_0": "observation.images.image"}' \
  > /workspace/eval_debug_3.log 2>&1
```

<font style="color:rgba(6, 8, 31, 0.88);"></font>

## <font style="color:rgba(6, 8, 31, 0.88);">测试结果：</font>
### <font style="color:rgb(0, 0, 0);">分组总览</font>
| **<font style="color:rgb(0, 0, 0);">组别</font>** | **<font style="color:rgb(0, 0, 0);">回合数</font>** | **<font style="color:rgb(0, 0, 0);">成功率(%)</font>** |
| --- | ---: | ---: |
| **<font style="color:rgb(0, 0, 0);">libero_spatial</font>** | <font style="color:rgb(0, 0, 0);">100</font> | <font style="color:rgb(0, 0, 0);">98.0</font> |
| **<font style="color:rgb(0, 0, 0);">libero_object</font>** | <font style="color:rgb(0, 0, 0);">100</font> | <font style="color:rgb(0, 0, 0);">100.0</font> |
| **<font style="color:rgb(0, 0, 0);">libero_goal</font>** | <font style="color:rgb(0, 0, 0);">100</font> | <font style="color:rgb(0, 0, 0);">96.0</font> |
| **<font style="color:rgb(0, 0, 0);">libero_10</font>** | <font style="color:rgb(0, 0, 0);">100</font> | <font style="color:rgb(0, 0, 0);">96.0</font> |
| **<font style="color:rgb(0, 0, 0);">Overall</font>** | <font style="color:rgb(0, 0, 0);">400</font> | <font style="color:rgb(0, 0, 0);">97.5</font> |


### <font style="color:rgb(0, 0, 0);">各任务成功率（每任务10回合）</font>
| <font style="color:rgb(0, 0, 0);">组别</font> | <font style="color:rgb(0, 0, 0);">任务ID</font> | <font style="color:rgb(0, 0, 0);">成功率(%)</font> |
| --- | ---: | ---: |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">0</font> | <font style="color:rgb(0, 0, 0);">90</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">1</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">2</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">3</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">4</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">5</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">6</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">7</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">8</font> | <font style="color:rgb(0, 0, 0);">90</font> |
| <font style="color:rgb(0, 0, 0);">libero_spatial</font> | <font style="color:rgb(0, 0, 0);">9</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">0</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">1</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">2</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">3</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">4</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">5</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">6</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">7</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">8</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_object</font> | <font style="color:rgb(0, 0, 0);">9</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">0</font> | <font style="color:rgb(0, 0, 0);">90</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">1</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">2</font> | <font style="color:rgb(0, 0, 0);">90</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">3</font> | <font style="color:rgb(0, 0, 0);">90</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">4</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">5</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">6</font> | <font style="color:rgb(0, 0, 0);">90</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">7</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">8</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_goal</font> | <font style="color:rgb(0, 0, 0);">9</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">0</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">1</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">2</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">3</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">4</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">5</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">6</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">7</font> | <font style="color:rgb(0, 0, 0);">100</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">8</font> | <font style="color:rgb(0, 0, 0);">70</font> |
| <font style="color:rgb(0, 0, 0);">libero_10</font> | <font style="color:rgb(0, 0, 0);">9</font> | <font style="color:rgb(0, 0, 0);">90</font> |


<font style="color:rgb(0, 0, 0);">备注：成功率(%) = 成功回合数 / 10 × 100。</font>





## 11.1:
1. 确认模型的：动作空间，关节空间，独立还是相对的

注意到一个 issue 里讨论：[The actions of SmolVLA. · Issue #2157 · huggingface/lerobot](https://github.com/huggingface/lerobot/issues/2157) SmolVLA 使用的是 end effector pose。然后，这个是我们现在pi05_libero_finetuned模型，使用的是数据集：[https://huggingface.co/datasets/HuggingFaceVLA/libero](https://huggingface.co/datasets/HuggingFaceVLA/libero) 微调的，他的动作描述是这样：

![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1762049670862-65be6d35-5a6a-4004-b20a-05a6297569a8.jpeg)

论文描述如下：关节角度，我查询各种资料也显示应该是关节空间，相对位置表示。

![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1762049689394-976b99c9-e040-4b5d-b8e7-1af1bbefcf55.jpeg)

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762049780742-a18cbb2e-8d8f-4639-bde7-0f770e310913.png)

另外文章中也说用于训练的数据集混合了很多数据，为了进行区分动作空间训练时用关键词 Prompt 进行区分，应该对不同的动作空间表示都是具有泛化的，但是我倾向于用 libero 的仿真测试看看交互接口，和 libero 保持一致，毕竟这个测试过。

## 11.2:
通过 libero的测试，看看debug下能看到和仿真交互的接口输出的action是怎么表示的

+ State: 8维
    - Pose axis-angle 6 abosulute ----> left_arm, pose_axis_angle 0-6
    - Gripper pose 1维拓展到2维重复的 ---> gripper close 拓展到两维    [0] [0 0]
+ 确认 Action：7维
    - 相对的pose ，axis angle 6维
    - Gripper pose 1维
+ <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">动作的 7 维是“相对/增量”命令（[-1,1] 归一化的位姿增量 + 夹爪），由 OSC_POSE 控制器映射到关节级执行。</font>
+ <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">observations（pixels 模式，双相机）</font>
    - <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">pixels.image: (256, 256, 3) uint8 ∈ [0,255]</font>
    - <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">pixels.image2: (256, 256, 3) uint8 ∈ [0,255]</font>
+ <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">observations（pixels_agent_pos 模式，双相机）</font>
    - <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">上面两张图 + agent_pos: (8,) float64（3 位置 + 3 姿态(axis-angle) + 2 夹爪关节）</font>
+ <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">actions</font>
    - <font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">(7,) float32 ∈ [-1, 1]（EE 位姿增量 + 夹爪）</font>

### 任务描述：数据采集需要对任务进行一个描述：
![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762090799350-88b66f1d-efff-4e21-8bb4-d5205ebb9ef6.png)

<font style="color:rgb(0, 0, 0);">Below are the instructions for all rounds in the sequence. These can be used directly for data collection or VLA model training.</font>

### <font style="color:rgb(0, 0, 0);">简单的描述</font>
#### <font style="color:rgb(0, 0, 0);">Round 1: A1 → B1 → C1 → D1 → E1 (Initial state customized by participants)</font>
+ **<font style="color:rgb(0, 0, 0);">A1</font>**<font style="color:rgb(0, 0, 0);">: Place the sphere into the circle hole in the first row.</font>
+ **<font style="color:rgb(0, 0, 0);">B1</font>**<font style="color:rgb(0, 0, 0);">: Place the triangular pyramid into the triangle hole in the first row.</font>
+ **<font style="color:rgb(0, 0, 0);">C1</font>**<font style="color:rgb(0, 0, 0);">: Place the wedge into the triangle hole in the first row.</font>
+ **<font style="color:rgb(0, 0, 0);">D1</font>**<font style="color:rgb(0, 0, 0);">: Place the cuboid into the square hole in the first row.</font>
+ **<font style="color:rgb(0, 0, 0);">E1</font>**<font style="color:rgb(0, 0, 0);">: Place the L-shaped block into the L-shaped hole in the first row.</font>

#### <font style="color:rgb(0, 0, 0);">Round 2: A2 → B2 → C2 → D2 → E2 (Initial state can be manually restored)</font>
+ **<font style="color:rgb(0, 0, 0);">A2</font>**<font style="color:rgb(0, 0, 0);">: Place the sphere into the circle hole in the second row.</font>
+ **<font style="color:rgb(0, 0, 0);">B2</font>**<font style="color:rgb(0, 0, 0);">: Place the triangular pyramid into the triangle hole in the second row.</font>
+ **<font style="color:rgb(0, 0, 0);">C2</font>**<font style="color:rgb(0, 0, 0);">: Place the wedge into the triangle hole in the second row.</font>
+ **<font style="color:rgb(0, 0, 0);">D2</font>**<font style="color:rgb(0, 0, 0);">: Place the cuboid into the square hole in the second row.</font>
+ **<font style="color:rgb(0, 0, 0);">E2</font>**<font style="color:rgb(0, 0, 0);">: Place the L-shaped block into the L-shaped hole in the second row.</font>

#### <font style="color:rgb(0, 0, 0);">Round 3: A3 → B3 → C3 → D3 → E3 (Initial state can be manually restored)</font>
+ **<font style="color:rgb(0, 0, 0);">A3</font>**<font style="color:rgb(0, 0, 0);">: Place the sphere into the circle hole in the third row.</font>
+ **<font style="color:rgb(0, 0, 0);">B3</font>**<font style="color:rgb(0, 0, 0);">: Place the triangular pyramid into the triangle hole in the third row.</font>
+ **<font style="color:rgb(0, 0, 0);">C3</font>**<font style="color:rgb(0, 0, 0);">: Place the wedge into the triangle hole in the third row.</font>
+ **<font style="color:rgb(0, 0, 0);">D3</font>**<font style="color:rgb(0, 0, 0);">: Place the cuboid into the square hole in the third row.</font>
+ **<font style="color:rgb(0, 0, 0);">E3</font>**<font style="color:rgb(0, 0, 0);">: Place the L-shaped block into the L-shaped hole in the third row.</font>

#### <font style="color:rgb(0, 0, 0);">Round 4: A4 → B4 → C4 → D4 → E4 (Initial state can be manually restored)</font>
+ **<font style="color:rgb(0, 0, 0);">A4</font>**<font style="color:rgb(0, 0, 0);">: Place the sphere into the circle hole in the fourth row.</font>
+ **<font style="color:rgb(0, 0, 0);">B4</font>**<font style="color:rgb(0, 0, 0);">: Place the triangular pyramid into the triangle hole in the fourth row.</font>
+ **<font style="color:rgb(0, 0, 0);">C4</font>**<font style="color:rgb(0, 0, 0);">: Place the wedge into the triangle hole in the fourth row.</font>
+ **<font style="color:rgb(0, 0, 0);">D4</font>**<font style="color:rgb(0, 0, 0);">: Place the cuboid into the square hole in the fourth row.</font>
+ **<font style="color:rgb(0, 0, 0);">E4</font>**<font style="color:rgb(0, 0, 0);">: Place the L-shaped block into the L-shaped hole in the fourth row.</font>

1.真机进行推理，docker 部署，然后需要注意哪些问题？

### what make a good dataset：
![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762091037445-f3720f64-e657-4f98-9200-98c57e118705.png)

## 11.3:
1. 训练 10000steps 420min，7 个小时。
2. 在 5090 机器的 docker 上，确认能够获取机器人的一些 video 设备的摄像头和数据通信端口。

```bash
deeptouch@deeptouch-ai:~$ lsusb
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 003 Device 002: ID 0b05:19af ASUSTek Computer, Inc. AURA LED Controller
Bus 003 Device 004: ID 05e3:0608 Genesys Logic, Inc. Hub
Bus 003 Device 017: ID 05e3:0610 Genesys Logic, Inc. Hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 004 Device 004: ID 05e3:0626 Genesys Logic, Inc. Hub
Bus 004 Device 005: ID 8086:0b07 Intel Corp. RealSense D435
Bus 004 Device 006: ID 8086:0b07 Intel Corp. RealSense D435
deeptouch@deeptouch-ai:~$ 
```

```bash
python - <<'PY'
import pyrealsense2 as rs

serial = "243722073411"  # 换成你要测试的那台序列号，另一台请后面再测
cfg = rs.config()
cfg.enable_device(serial)
cfg.enable_stream(rs.stream.color, 640, 480, rs.format.rgb8, 30)

pipe = rs.pipeline()
try:
    profile = pipe.start(cfg)
    print("Started pipeline for device:", serial)
    frames = pipe.wait_for_frames(5000)
    color = frames.get_color_frame()
    print("Got color frame:", bool(color), (color.get_width(), color.get_height()) if color else None)
finally:
    pipe.stop()
    print("Pipeline stopped.")
PY
```

```bash
python - <<'PY'
from lerobot.cameras.realsense import RealSenseCamera, RealSenseCameraConfig

serial = "335522070753"  # 换成要测的那台
cfg = RealSenseCameraConfig(serial_number_or_name=serial, width=640, height=480, fps=30)
cam = RealSenseCamera(cfg)
try:
    cam.connect(warmup=False)
    img = cam.read()
    print("Frame shape:", getattr(img, "shape", None))
finally:
    if cam.is_connected:
        cam.disconnect()
        print("Disconnected.")
PY


```

```bash
# 先停掉旧容器（如果存在）
docker rm -f lerobot_rmc || true
docker run -dit \
  --name lerobot_rmc \
  --gpus all \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v /home/rmc/lerobot:/workspace/lerobot \
  -v /data:/data \
  -v /ssd_cach:/ssd_cach \
  lerobot_pi05:latest

/workspace/lerobot/outputs/pi05_checkpoint_liberobase_realmanfintuned_20251103_012148
  
  rsync -ah --info=progress2 --partial --inplace --whole-file \
  -e "ssh -T -o Compression=no -o IPQoS=throughput -c aes128-gcm@openssh.com -o StrictHostKeyChecking=accept-new" \
  /home/rmc/lerobot/outputs/pi05_checkpoint_liberofintuned_realmanfintuned_20251103_094701/ \
  deeptouch@192.168.1.159:/data/outputs/pi05_checkpoint_liberofintuned_realmanfintuned/

rsync -ah --info=progress2 --partial --inplace --whole-file \
  -e "ssh -T -o Compression=no -o IPQoS=throughput -c aes128-gcm@openssh.com -o StrictHostKeyChecking=accept-new" \
  /workspace/lerobot/outputs/pi05_checkpoint_liberobase_realmanfintuned_20251103_012148/ \
  deeptouch@192.168.1.159:/data/outputs/pi05_checkpoint_liberobase_realmanfintuned/

  
```

3. 真机调通，输出的动作序列：

![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1762169232188-bff15a7d-8cea-4e98-bfb0-249f8f3dfcae.jpeg)



## 11.4
```bash
# 先停掉旧容器（如果存在）
docker rm -f lerobot_rmc || true

# 假设宿主机出现了 /dev/video0 和 /dev/video1；若只有一个就保留 video0
docker run -dit \
  --name lerobot_rmc \
  --gpus all \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v /home/rmc/lerobot:/workspace/lerobot \
  -v /data:/data \
  -v /ssd_cach:/ssd_cach \
  \
  # 摄像头设备透传（按需调整）
  --device=/dev/video0 \
  --device=/dev/video1 \
  --group-add=video \
  -v /run/udev:/run/udev:ro \
  --device-cgroup-rule='c 81:* rmw' \
  \
  # 如果有串口设备（按需添加）
  --device=/dev/ttyUSB0 \
  --device=/dev/ttyACM0 \
  --group-add=dialout \
  --device-cgroup-rule='c 188:* rmw' \
  --device-cgroup-rule='c 166:* rmw' \
  \
  # 对某些深度相机（如 RealSense），可能还需要 USB 总线访问：
  # -v /dev/bus/usb:/dev/bus/usb \
  \
  lerobot_pi05:latest
```

1.通过仿真器的输入和输出，确认 action 的单位，和模型输入的内容以及输出的内容。

[https://huggingface.co/datasets/HuggingFaceVLA/libero](https://huggingface.co/datasets/HuggingFaceVLA/libero)

libero 仿真器，给的 libero 数据是：

#17

observation.images.image

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762249805395-3ed332fc-d2c8-4eb3-937c-54f74a298781.png)

observation.images.image2

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762249816416-8d9db5a9-1717-4647-8db3-cb343839c8db.png)

observation：[-0.055803101509809494,-0.05191003158688545,0.6638696193695068,3.1608452796936035,-0.007363406475633383,-0.09749018400907516,0.03938949480652809,-0.039458028972148895]

action：

[0,-0.7205356955528259,-0.13660714030265808,0.061071429401636124,0.006428571417927742,0.017142856493592262,-1]

timestamp：1.6

frame_index：16

episode_index：0

index：16

task_index：0

```bash
git config --global user.name "Daniel_rmc"
git config --global user.email "daniel_rmc@shdeeptouch.cn"
```

仿真器如何接收 action，仿真执行的动作和 action 输出是怎么连接的。

```bash
mkdir /home/rmc/.vscode-server/extensions/.1937843c-e302-4801-a69c-4883ca710746/python_files/lib/jedilsp/jedi/third_party/typeshed/stdlib/2and3/logging
```

<font style="color:rgb(97, 97, 97);background-color:rgb(243, 243, 243);">mkdir /home/rmc/.vscode-server/extensions/.0a39ce25-07c1-4a8b-ac66-c44ebc8a8d2d/python_files/lib/jedilsp/jedi/third_party/typeshed/stdlib/2and3/logging</font>

## 11.15
- [x] 1.通过debug模式，了解仿真器输入state和输出action，以及输出action后到控制机器人是如何工作的。
- [x] 2.image 180 度是怎么回事？

为什么要旋转 180°

LiberoEnv._format_raw_obs 里对每张 raw image 使用了 image[::-1, ::-1]（等价于同时水平 + 垂直翻转，即旋转 180°）。原因（工程层面的常见性与此处的动机）：Off-screen OpenGL 渲染和仿真相机的坐标系与常规图像坐标（左上角为原点）常有差异，常见至少需要一次垂直翻转（上下颠倒）。

在 robosuite / mujoco 相机管线中，还可能存在水平镜像（左右反）的问题（取决于相机姿态和渲染读取方式），因此这里直接做了“180°旋转”，一次性把上下和左右都校正到与训练数据一致的朝向。

这一步的核心目的是“**让当前仿真图像的朝向与训练数据/既有模型期望的朝向保持一致**”，避免模型学到的左/右、上/下语义与实际画面相反，从而导致控制反向或目标理解错误。

后续 preprocess_observation 只做通道顺序与数值归一化，不会改变几何朝向。

如果去掉这一步，模型可能表现明显变差（左右/上下语义错位）。

libero 数据集给的 image：![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762345257387-e09c3889-45c4-4881-a09f-ba510b02d6c0.png)![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1762345267125-1d545176-7a2e-4fda-97ed-adbff6b0e5e0.png)



3.归一化和反归一化，（模型内外，仿真交互），缩放系数（0.05，0.05，0.05，0.5，0.5，0.5）。

数据归一化是只根据我们自采数据集进行归一化，还是会保留之前预训练模型的归一化参数。

数据归一化可以用数据采集提供，也可以加载预训练的，但是推荐使用数据自己的，meta.<font style="color:rgb(0, 0, 0);">stats.json 文件里计算了数据的统计量。</font>

以及，这里面的缩放系数，是默认设置的还是配置文件给的。

## 11.6
- [x] 1.确认夹爪的开合表示，-1，+1 哪个代表开哪个代表关，我们的机器人的夹爪是+1 代表闭合。
- [x] 2.模型应该从基础的 pi05 开始调，而不是在 libero 微调后的模型上开始调。

## 11.7
### 1. 组会纪要
方老师： 1.介绍类脑-触觉，国家重大项目。 研究方向包括：触觉的，VTLA 的，多模态等。2.重点讨论两个操作平台，灵巧手讨论。

涛哥：1.peg in the hole 任务下的框架是 lerobot，base 模型是 pi05； 2.新采集了 350 条数据。3.模型训练 完成了。 4.视触觉 用千觉 sdk 从图像提取了力。

方老师：先试试 pi05 的 baseline 怎么样，然后考虑如何加入触觉，提到触觉 VLA 双层架构。

提到了下周的进度：完成真机部署。

ATI 传感器（适用多维的力/扭矩 传感器）



## 11.10
1.连接真机，先检测摄像头的连接状态，然后挂载摄像头设备透传到 docker，

```bash
# 先停掉旧容器（如果存在）
docker rm -f lerobot_rmc || true

# 假设宿主机出现了 /dev/video0 和 /dev/video1；若只有一个就保留 video0
docker run -dit \
  --name lerobot_rmc \
  --gpus all \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v /home/rmc/lerobot:/workspace/lerobot \
  -v /data:/data \
  -v /ssd_cach:/ssd_cach \
  \
  # 摄像头设备透传（按需调整）
  --device=/dev/video0 \
  --device=/dev/video1 \
  --group-add=video \
  -v /run/udev:/run/udev:ro \
  --device-cgroup-rule='c 81:* rmw' \
  \
  # 如果有串口设备（按需添加）
  --device=/dev/ttyUSB0 \
  --device=/dev/ttyACM0 \
  --group-add=dialout \
  --device-cgroup-rule='c 188:* rmw' \
  --device-cgroup-rule='c 166:* rmw' \
  \
  # 对某些深度相机（如 RealSense），可能还需要 USB 总线访问：
  # -v /dev/bus/usb:/dev/bus/usb \
  \
  lerobot_pi05:latest
```

2. 检测从摄像头到 action。
3. 从 action 到执行动作。

```bash
# 停旧容器（容器不存在则忽略错误）
docker rm -f lerobot_rmc 2>/dev/null || true

# 启动新容器（使用 video2 与 video4 两路摄像头）
docker run -dit \
  --name lerobot_rmc \
  --gpus all \
  --network host \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v /home/deeptouch/workspace/lerobot:/workspace/lerobot \
  -v /data:/data \
  -v /ssd_cach:/ssd_cach \
  --device=/dev/video2 \
  --device=/dev/video4 \
  --network=host \
  --group-add=video \
  -v /run/udev:/run/udev:ro \
  --device-cgroup-rule='c 81:* rmw' \
  lerobot_pi05:latest



  # 进入容器后先conda activate lerobot 激活环境，然后记得安装Robotic_Arm,flask
  pip install Robotic_Arm
  apt-get install v4l-utils
  pip install flask

  # 检测摄像头dev
  python -m lerobot_robot_realman.examples.monitor_cameras --probe


python -m lerobot_robot_realman.examples.run_realman_smolvla \
  --model-dir /workspace/lerobot/model_save/smolvla_training_20251116_200135/smolvla_training_20251116_200135/checkpoints/008000/pretrained_model \
  --arm left_arm --action-mode osc_delta --device cuda \
  --cam1 /dev/video10 --cam2 /dev/video16 --cam3 /dev/video4 --no-init


  
```

```bash
# 1. 确认宿主或容器内是否能看到 UDP 包（替换端口为实际配置）
apt-get update && sudo apt-get install -y tcpdump
tcpdump -nn -i any udp port 8008 -c 10

# 2. 查看监听进程（如果底层库会创建监听）
ss -ulpn | grep 8008
ss -ulpn | grep 8009

# 3. 确认容器内本机 IP（看是否与 target_ip 在同一网段）
ip addr show

# 4. 临时把 target_ip 改成实际本机可达 IP 再运行，比较是否开始收到数据
# 5. 增加调试：在 _on_robot_state_update 顶部 print(received_arm_ip)
```

重新训练一个： smolval 训练需要映射好摄像头。

```bash
cd /workspace/lerobot
PYTHONPATH=src:. python -m lerobot_robot_realman.examples.run_realman_pi05 \
  --model-dir /workspace/lerobot/model_save/pi05_training_20251106_203227/checkpoints/010000/pretrained_model \
  --arm left_arm \
  --action-mode osc_delta \
  --cam1 2 \
  --cam2 4 \
  --device cuda

python Realman/test_send_action.py --arm right_arm --loop（用来验证和恢复初始位置）
```

现在成功动起来啦双臂。效果不是很好

可能的原因：

1. 摄像头视角问题，双臂初始状态摄像头和采集数据时候不是很一致；相机加载的好像不对。

已排查出：video12 和 video18 是视觉 rgb 摄像头。

2. 模型训练效果问题（没有主视角摄像头，导致训练效果不好，训练 loss 最低为 0.2，理想情况为 0.00X）
3. 部署的异步推理框架待检查和完善，由于发送动作和执行动作之间有延迟，需要进行异步推理。



## 11.11
实际部署的效果：

1.z 轴反向运动，而且 z 轴输出过大，可以输入一帧训练数据集，看看 action。

解决：已经找到原因，采集图片后需要按照训练时候数据集预处理时进行相同的图像归一化处理，以及在加载 postprocesser 时候需要使用指定的预训练的模型文件下的配置。（保持和训练数据集相同的反归一化）

```bash
python lerobot/tools/inspect_pi05_batch.py \
  --model /workspace/lerobot/model_save/pi05_training_20251106_203227/checkpoints/010000/pretrained_model \
  --repo-id my_local_transform_data \
  --root /workspace/lerobot/datasets/transform_data \
  --index 12 \
  --device cuda \
  --print-shapes
```

```bash
rsync -avP /home/rmc/lerobot/outputs/act_training_20251114_151101 deeptouch@192.168.1.159:/home/deeptouch/workspace/lerobot/model_save/act_training_20251114_151101
```

```bash
python -m lerobot_robot_realman.examples.run_realman_pi05 \
  --model-dir /workspace/lerobot/model_save/pi05_training_20251110_215725/checkpoints/002000/pretrained_model \
  --arm left_arm --action-mode osc_delta --device cuda \
  --cam1 /dev/video18 --cam2 /dev/video24 --no-init

python -m lerobot_robot_realman.examples.run_realman_pi0 \
  --model-dir /workspace/lerobot/model_save/pi0_training_20251111_203510/checkpoints/002000/pretrained_model \
  --arm left_arm --action-mode osc_delta --device cuda \
  --cam1 /dev/video18 --cam2 /dev/video24 --no-init
  
python -m lerobot_robot_realman.examples.run_realman_smolvla \
    --model-dir /workspace/lerobot/model_save/smolvla_training_20251111_204540/smolvla_training_20251111_204540/checkpoints/010000/pretrained_model \
    --arm left_arm --action-mode osc_delta --device cuda \
    --cam1 /dev/video18 --cam2 /dev/video24 --no-init
```

现象：机器人手臂可以向圆柱体靠近，但是不能正确找到夹起它的位置，左手腕部相机的视角影响特别大（（腕部相机视角可以旋转）。

采集数据时候的视角如下：推理时候尽量保持相同的视角

![](https://cdn.nlark.com/yuque/0/2025/webp/22559595/1762860339248-e45e7afb-427d-4c3a-9874-f8224b37dd45.webp)

## 11.12
to do list： 

1.部署一下 smolval 的官方异步推理部署，试试效果。

## 11.14
```bash
docker stop lerobot_rmc || true
docker rm lerobot_rmc || true

docker run -dit \
  --name lerobot_rmc \
  --gpus all \
  --runtime=nvidia \
  --ipc=host \
  --shm-size=32g \
  --restart=unless-stopped \
  -v /home/rmc/lerobot:/workspace/lerobot \
  -v /data:/data \
  -v /ssd_cach:/ssd_cach \
  lerobot_pi05:latest

  
```

1. 实现了一下简单 dp 策略的小模型：ACT 小模型试试效果。

ACT 已开始训练，发现训练瓶颈在于 io 瓶颈和图片增强部分 image transform 部分，显卡的训练和推理时间占比很小，90%以上时间显卡处于等待状态。

日志中每 200 步约耗时 16–17 分钟：第 0 →200 步：10:58:13 → 11:14:57 ≈ 1004 秒，单步 ≈ 5.0 s

第 200 →400 步：11:14:57 → 11:30:56 ≈ 959 秒，单步 ≈ 4.8 s

训练循环里记录的指标：update_s ≈ 0.26–0.32 s（模型前向+反向+优化器步）；data_s ≈ 4.5–4.7 s（DataLoader+预处理+增强）说明 90% 时间消耗在取 batch 与数据预处理，模型本身并不慢，GPU 绝大多数时间处于等待状态。

2. 优化了训练速度：
    1. 将数据集从/data 盘复制到 /lerobot/datasets 文件目录下，从 hhd 到 ssd 可以增加数据读取速度。
    2. 主要优化空间在数据侧 data_s：关闭图像增强 + 并发持久化 + 更高预取，通常能下降 15–30%（多摄像头场景越多、收益越明显），因为 update_s 已经很小啦，说明显卡的训练推理很快啦。

通过增大 num_works，以及上述的各种方法，将 data_s 从之前的十倍于 update_s 降低到 2 倍于 update_s。整体速度提高了五倍。训练时长从原本 13h 变成了 1.5h。（针对 act）

由于数据的不连续和 task 字段错误等 bug，下午没有训练上新增 top_view 的数据。下午在 debug。

3. 检查到了数据的错误：当前数据集中 tasks 为 none，数据的时间戳不连续，最大不对齐超过 0.03s。
4. 以及数据的 observation.task 字段错误，无法正常训练，编写了 debug 该问题的调试脚本，已经解决了该问题，训练起来啦。

截止晚上已经解决了 bug，将 smolval 对应 top_view 的数据训练起来啦。



## 11.17
##### 测试增加 top_view 视角后的摄像头：
能够实现提高百分 20 左右的夹取成功率；（0-20%）；根据这几次测试的结果，总结了现有高概率复现的问题：

1. 从纯腕部摄像头 2d 画面，经常在夹爪靠近物体的的时候误判物体处于正确夹取范围，实际上可能此时夹爪高于物体上端几毫米或者几厘米。（垂向误差）
2. 夹爪经常碰倒圆柱体，右边的指头经常贴着圆柱体（向下压圆柱体，然后弄倒）。（物体不在可夹抓范围）；

（水平横向误差）

1 和 2 引出问题：如何使得夹爪能够正确判断是否夹住。

3. 物体即使在可夹抓范围，因为圆柱体很小，底部摩擦力小，夹爪夹得位置高，也会夹倒圆柱体，当圆柱体不在夹爪的中间位置，特别偏向某一指头时，由于夹爪是对称对夹，所以当夹爪闭合时，容易在完全闭合前，弄倒圆柱体。（对称夹爪，当物体不在对称轴附近时，容易被单边指夹碰倒圆柱体。）

3 引出问题：如何让夹爪夹得稳。尤其在物体放置平面低摩擦系数时，如何不碰倒物体。关于夹爪和物体接触面的摩擦系数暂不考虑可以通过材料增加摩擦系数和通过增加夹爪力度增大正压力。



对于这三个问题，我有以下几个思考：

1：对于垂直误差问题：原因：因为目前的夹爪安装姿态和腕部相机的安装姿态，导致腕部相机的视野是垂直于夹爪闭合平面的，而夹爪的夹取动作采集数据集中的动作轨迹也是利用夹爪从垂直角度“套上”圆柱体，腕部摄像头只能判断圆柱体在水平面投影在不在夹爪的中央，难以判断垂直高度上，夹爪操作平面是否与圆柱体相交。

可解决的方案： 1.增加水平面上的观察视角，比如增加头部摄像头，通过头部摄像头观察水平面，物体是否在夹爪的上下宽度范围里。（实际验证，提高了夹取成功率）

2：水平误差和垂直误差共同决定。 由于无法判断垂直方向是否和圆柱体相交，可能错过了正确的水平位置对齐时机，也就是水平方向的对齐必须在夹爪高于物体时完成，而不能当夹爪已经接触到了物体或者低于物体时再去水平摆动夹爪。 待解决办法：可以通过改进数据采集方案：通过先缓慢靠近物体，当确认物体在正确待夹取位置后（腕部相机观察到圆柱体处于夹爪的两指中间，头部相机观察到夹爪和圆柱体相交，先暂停短时间进行动作确认，然后进行夹爪闭合。

3：对于非对成夹爪导致的容易单边碰到圆柱体的问题，可有两个解决办法：1.优化采集数据质量，垂直方向上：尽量让夹爪处于中间位置夹取圆柱体，让物体长度超出夹爪即使物体滑动和碰倒由于高出的长度容易被另一边夹爪夹住。水平方向上：让物体对称轴重合于夹爪对称轴，也就是让物体处于夹爪中央位置 （对数据采集要求高） 2. 设计非对称夹爪，两只夹指自由开闭。（action 维度增加一维，将 gripper 从 bool（0，1）判断开闭，变为夹爪绝对位置（gripper1，griiper2），对数据采集要求低，灵活适应各种任务。



以上的问题，也可以通过改变夹爪安装姿态得到改善，将夹爪从垂直于手臂关节安装，改成平行于手臂关节安装，也就是将夹爪作为手臂的延长线，这样腕部相机在当前安装位置，可以和夹爪的闭合平面平行，倾斜腕部相机可以得到水平和垂直夹爪闭合平面的视野。同时还是需要结合 top_view 相机对夹爪的夹取进行一个确定。（第三视角观察物体和夹爪的位置）

但无论何种安装夹爪的方式，水平或垂直于手臂，2d 摄像头总会有一个方向难以确认。

1. 像 UMI 一样增加反光（凸/凹面镜待确认，应该是凸面镜视野大就像急转弯的地方），让腕部相机看到另一个方向的视野，此时就相当于 3d 视野啦。

2. 通过增加触觉模态，来得到高可靠接触确认。

3.通过采集数据集，采集正样本，负样本，进行学习。

（面对问题时，可以从下列三个方向上进行思考：1.从硬件上思考，2.从软件上（算法）思考 3.从高质量数据集的构建上进行思考），思考是自下向上的。目标任务为中心的。



另外关于触觉信息的增强方面的工作思考。

1. 可以做一个有关触觉理解的： <font style="color:rgb(62, 62, 62);">Tactile-QA, 类似图片理解，video QA，通过采集 Tactile 的数据集，形成 TLA，着重关注 Tactile 方面的理解，比如：纹理、材料感知、还有滑动感知等。</font>

##### 2. 异步推理
```bash
# 终端A：服务端
python -m lerobot.async_inference.policy_server --host 0.0.0.0 --port 8080 --fps 30 --inference_latency 0.05

# 终端B：客户端（已将相机名改为 camera1/2/3）
python -m lerobot_robot_realman.examples.run_async_realman_client \
  --server 127.0.0.1:8080 \
  --policy-type smolvla \
  --pretrained /workspace/lerobot/model_save/smolvla_training_20251116_200135/smolvla_training_20251116_200135/checkpoints/008000/pretrained_model \
  --device cuda \
  --actions-per-chunk 30 \
  --chunk-threshold 0.6 \
  --fps 20 \
  --arm left_arm \
  --left-ip 192.168.0.18 --right-ip 192.168.0.19 --rmc-port 8080 \
  --action-mode osc_delta \
  --cam /dev/video10:camera1 --cam /dev/video16:camera2 --cam /dev/video4:camera3
```

成功部署了，但是异步推理反而更差了，虽然动作很流畅，但是机械臂总是在向一个特定的方向进行找寻物体，同时不管圆柱体在哪儿，它都向着操作台上面去找圆柱体。



## 11.18
to do list：

1.确认一下量化模式的区别的影响，使用 MEAN_STD 和 q10，q99 这样的分段统计量化，有没有区别。

2.解决 worning 的问题。



## 11.19
1.确认 ACT，Smolvla，diffusion policy，Pi05 的 loss 。

ACT：Acion Chunking Transformer

1.取条件输入（图像，语言，本体状态） 与 监督目标（动作序列），并按照配置做归一化和填充到固定长度 TxD。

smolvla：



Pi05：

2. 确认，action目前是相对的 <font style="color:rgb(31, 35, 40);">end-effector pose (position + orientation + gripper state)</font> 来进行训练，状态state是当前的绝对位姿么

是否能用绝对 end-effector pose 来进行训练。机器人控制能否直接移动到绝对位置上，这部分写在机器人控制层，这样能够直接接收模型输出预测到 action 来 执行动作。



3. 实现 diffusion 训练，并加上历史帧，现在的时间上下文为 2。由于 diffusion 的 data 处理和时间上下文，data_s 时间比 smolvla 还长，通过测试不同的如下参数：挑选了一组比较合适的 NUM_WORKERS=64, LERO_PREFETCH_FACTOR=16（更多预取），以及选定了 batchsize=32，比 64 更快，同时 loss 收敛得更好。从默认参数训练的时间 20 小时缩小到 3 小时左右，提速 6 倍左右。
4. 规划设计了新的数据采集规范，并固定了 top 摄像头的位置，亚克力板的位置固定左上角有一个双面胶。

![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1763546621765-47238461-04e8-4b8d-8f78-7d4e318ee795.jpeg)![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1763546620816-f3ab9683-a62b-4585-9daa-cdbc1536d65f.jpeg?x-oss-process=image/auto-orient,1)![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1763546623544-d90618a2-86ba-43af-84e9-6ee460356a35.jpeg)![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1763546622905-660240fb-b953-4ea5-9ef0-421c3b469de7.jpeg?x-oss-process=image/auto-orient,1)

5. 在 act 上增加历史帧。



## 11.20
```bash
(base) rmc@deeptouch-NF5468-M7-A0-R0-00:~$ docker images
REPOSITORY     TAG                        IMAGE ID       CREATED        SIZE
lerobot        ubuntu22_cuda12_8          2066b2eaae1e   3 weeks ago    32.5GB
lerobot_pi05   latest                     a005db34894a   3 weeks ago    47.3GB
robot_data     v1                         6c611298937a   3 weeks ago    11.3GB
lerobot_team   latest                     d72d802eb88f   5 weeks ago    29.6GB
nvidia/cuda    12.9.1-devel-ubuntu24.04   eecafe98c3e1   4 months ago   10.2GB
nvidia/cuda    12.9.1-devel-ubuntu22.04   c76e8f8e54eb   4 months ago   10.2GB
nvidia/cuda    12.8.1-devel-ubuntu22.04   892341855b05   8 months ago   9.34GB
(base) rmc@deeptouch-NF5468-M7-A0-R0-00:~$ # 列出所有镜像并逐个保存
for image in $(docker images --format "{{.Repository}}:{{.Tag}}"); do
    filename="/data/docker_images/${image//\//_}.tar"  # 将斜杠替换为下划线，以便于文件命名
    docker save -o "$filename" "$image"
    echo "Backed up $image to $filename"
done
Backed up lerobot:ubuntu22_cuda12_8 to /data/docker_images/lerobot:ubuntu22_cuda12_8.tar
Backed up lerobot_pi05:latest to /data/docker_images/lerobot_pi05:latest.tar
Backed up robot_data:v1 to /data/docker_images/robot_data:v1.tar
Backed up lerobot_team:latest to /data/docker_images/lerobot_team:latest.tar
Backed up nvidia/cuda:12.9.1-devel-ubuntu24.04 to /data/docker_images/nvidia_cuda:12.9.1-devel-ubuntu24.04.tar
Backed up nvidia/cuda:12.9.1-devel-ubuntu22.04 to /data/docker_images/nvidia_cuda:12.9.1-devel-ubuntu22.04.tar
Backed up nvidia/cuda:12.8.1-devel-ubuntu22.04 to /data/docker_images/nvidia_cuda:12.8.1-devel-ubuntu22.04.tar
(base) rmc@deeptouch-NF5468-M7-A0-R0-00:~$ ls /data/docker_images/
lerobot_20251015.tar     lerobot_team:latest.tar  lerobot:ubuntu22_cuda12_8.tar             nvidia_cuda:12.9.1-devel-ubuntu22.04.tar  robot_data:v1.tar
lerobot_pi05:latest.tar  lerobot_team_v1.tar      nvidia_cuda:12.8.1-devel-ubuntu22.04.tar  nvidia_cuda:12.9.1-devel-ubuntu24.04.tar
(base) rmc@deeptouch-NF5468-M7-A0-R0-00:~$
```

```bash
(base) rmc@deeptouch-NF5468-M7-A0-R0-00:~$ sudo cp -r /ssd_cach /data/
[sudo] password for rmc: 
(base) rmc@deeptouch-NF5468-M7-A0-R0-00:~$
```

## 11.21
制作组会ppt：

![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1763716836375-1291be50-c695-4156-a635-b8ffdca347fd.png)

 

训练的步长：需要长一点，训练到： 6w steps。

下周：设计 lora 微调，冻结参数微调。

训练的技巧：先使用大数据集训练，再使用小数据集训练 这个的大小数据集是什么意思？

先用 2000，再用 100 条，

VLA-Areno。



# 11.24
选择模型 checkpoints 如下：

pi05:28000 loss:0.27463

smolvla:4200 loss:0.059724

上述效果不好（可能是归一化方式不一致）

- [x] 测试模型训练效果：
- [ ] 采集数据，转换完成后，开启训练。

# 11.25
异常情况：

1. diffusion 更改后，loss 降下了很多，但是半夜训练 被 numworkers 挤掉啦。

上午 to do：

1. resume diffusion 的训练

```bash
lerobot-train --config_path="/workspace/lerobot/outputs/diffusion_training_20251124_205151/checkpoints/014000/pretrained_model/train_config.json" --resume=true
```

2. 测试 diffusion 和其他的稳定的模型。
3. 检查图像 image 保持相同尺寸。推理和训练

to do list

4. RLinfra 看看强化学习的框架，是在线交互还是离线交互。 
5. 训练使用关节空间的模型，比较和笛卡尔空间。



请帮我查看一下这是为什么，这些信息是在说什么？

我想 resume 后放在新的目录;

把我写一个 resume.sh,并把 train_smolvla.sh 恢复成之前的从头训练的脚本

# 11.26
to do list：

1. 检查图像 image 保持相同尺寸。推理和训练
2. 测试新规划后的 top-view 视角的任务成功率。

解决相机推理时候和数采时捕获相机画面不一致的问题。![](https://cdn.nlark.com/yuque/0/2025/png/22559595/1764149106852-0b7b1ed4-c9a7-4337-adb3-4aead7b9809f.png)

直接使用 cammera manager 会出现下面的错误，同时 cammera manager 也不是完整的跟数采相同的格式，数采的格式是通过recording_config.yaml，lerobot_formmater ，data_recod 应该还需要下面的 udevam 共同完成的，主要是没有设定一个输出 image 的 api 接口，不方便调用。

我根据 数采这些代码，看了一下里面逻辑，在这里重新写了获取图像的内容，同时 保持尺寸变换的逻辑，优先 720 1280  再通过 cv.resize icv2.INTER_AREA 到480 640



# 11.28
将推理的 batch 图像编码和 resize 跟数采的进行啦对齐。



# 12.3
计划 to do：

**工作：**

**上午+中午：**

**整理代码工作**

- [x] 整理到一个文件夹，将各个类换成原始类的继承
- [ ] 重新整理之前的训练脚本和代码，包含训练脚本，act，smolvla，diffusion，pi05，pi0。
- [x] 整理真机部署代码和规范，包含真机图像检测；机械臂位置初始化（键盘输入 action，send action 到机械臂进行执行动作；真机部署 smolvla，act，pi05，diffusion；异步推理 server 和 client 框架适配已经训练的各种策略。）

**长期发展**

5. 设计 如何多视角 2d 构建 3d 点云，然后融入 diffusion 中，进行初步的多视角融合。

**学业：**

下午 or 晚上

    - [x] 将开题报告的具体研究内容部分，拆分成 4000 字+2000 字，现在是将技术路线写在了一起，单独将技术路线独立出来汇总形成第三个板块的内容（2000 字）；将前面的具体研究内容把三个子任务之间的联系加强一下。
    - [ ] （可以制作一个子任务关系图），同时仔细审核深化一下具体实现目标和效果，解决的科学问题。



# 12.4
1. 训练 groot 模型，
2. 测试数据集划分，训练 epoch 200（act 算法，每 10epoch 保存一次模型，同时保存 loss 最低的 best 模型）

目前数据集 30fps，200 个样本，每个大概在 6-10s  之间，一共 53820 个 frame，一共大概 1794s，，半个小时。

# 12.8
```bash
lerobot-train \
  --dataset.repo_id=pepijn223/bimanual-so100-handover-cube \
  --output_dir=./outputs/xvla_bimanual \
  --job_name=xvla_so101_training \
  --policy.path="lerobot/xvla-base" \
  --policy.repo_id="YOUR_USERNAME/xvla-biso101" \
  --steps=3000 \
  --policy.device=cuda \
  --policy.action_mode=so101_bimanual \
  --policy.freeze_vision_encoder=True \
  --policy.freeze_language_encoder=True \
  --policy.train_policy_transformer=True \
  --policy.train_soft_prompts=True
```

# 12.9
1. 配置 xvla。
2. 配置没有 state 的训练方法。
3. 调研多视角融合算法。

成功率：

| | diffusion | act | smolvla | pi05 |
| --- | --- | :---: | :---: | :---: |
| 100 episode | 0% | | | |
| 200 episode | 0% | | | |
| 400 episode |  | | | |

# 12.16
1. 看 diffusion 里的 cnn 图像特征。
2. SERL，HIL-SERL，conrft，GR-RL，RL-100

# 12.7-12.21 ：
1. 开题答辩 ppt，开题报告撰写。

后续工作日志📔见：

[[Realman + lerobot 适配工作]]
