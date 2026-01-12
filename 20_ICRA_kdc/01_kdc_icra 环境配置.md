git check out icra 失败时候：由于gitclone 时候指定了 --depth=1：git clone --depth=1 https://github.com/LejuRobotics/kuavo_data_challenge.git 
导致 git fecth all 看不到icra分支，只有main分支。
使用如下命令可以解决：
git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"