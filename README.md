# E2Calib: 事件相机标定

<p align="center">
   <img src="http://rpg.ifi.uzh.ch/img/papers/CVPRW21_Muglikar.png" height="300"/>
</p>
该仓库包含了如论文[Muglikar et al. CVPRW'21](http://rpg.ifi.uzh.ch/docs/CVPRW21_Muglikar.pdf)描述的通过事件数据进行灰度视频恢复的实现，以用于进行相机（和外部传感器）标定。

如果您在学术研究中使用了该代码，请引用下列工作：

[Manasi Muglikar](https://manasi94.github.io/), [Mathias Gehrig](https://magehrig.github.io/), [Daniel Gehrig](https://danielgehrig18.github.io/), [Davide Scaramuzza](http://rpg.ifi.uzh.ch/people_scaramuzza.html), "How to Calibrate Your Event Camera", Computer Vision and Pattern Recognition Workshops (CVPRW), 2021

```bibtex
@InProceedings{Muglikar2021CVPR,
  author = {Manasi Muglikar and Mathias Gehrig and Daniel Gehrig and Davide Scaramuzza},
  title = {How to Calibrate Your Event Camera},
  booktitle = {{IEEE} Conf. Comput. Vis. Pattern Recog. Workshops (CVPRW)},
  month = {June},
  year = {2021}
}
```

**原仓库中支持了三种事件文件的格式，这里仅介绍ROS包格式（[dvs\_msgs](https://github.com/uzh-rpg/rpg_dvs_ros/tree/master/dvs_msgs)）的依赖安装和使用。想了解另外两种格式的可以参考[原仓库](https://github.com/uzh-rpg/e2calib)**

## 安装

安装过程总共分为两步（默认已安装了ROS）：

1. 因为兼容性的问题，需要在原始环境（不能是虚拟环境）中安装**转换代码**的依赖包。
2. 在conda环境中安装**图像恢复代码**的依赖包。

### 转换代码的依赖

```bash
pip3 install --no-cache-dir -r requirements.txt # 请在克隆的本仓库路径下执行
pip3 install dataclasses # 如果你的python版本小于3.7
pip3 install --extra-index-url https://rospypi.github.io/simple/ rospy rosbag
```

### 图像恢复的依赖

首先需要安装anaconda，可参考[链接](https://blog.csdn.net/qq_39779233/article/details/127199957)。

然后执行下述，cuda请根据显卡驱动版本进行安装，对应关系可参考[链接](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#id4)。

```bash
cuda_version=10.1

conda create -y -n e2calib python=3.7 # 创建e2calib的conda虚拟环境
conda activate e2calib
conda install -y -c anaconda numpy scipy
conda install -y -c conda-forge h5py opencv tqdm
conda install -y -c pytorch pytorch torchvision cudatoolkit=$cuda_version

pip install python/ # 请在克隆的本仓库路径下执行，这一步是为了安装e2vid
```

图像恢复代码通过[E2VID](http://rpg.ifi.uzh.ch/docs/TPAMI19_Rebecq.pdf)将h5格式文件中的事件恢复成灰度图像。

为了将结果输入[kalibr](https://github.com/ethz-asl/kalibr) （多相机和IMU标定包），还需要将恢复的图像转换为rosbag，因此还需要在**原始环境**中安装以下依赖：

```bash
pip3 install tqdm opencv-python opencv-contrib-python
pip3 install --extra-index-url https://rospypi.github.io/simple/ sensor-msgs
```

### kalibr安装

请参考kalibr[官方安装手册](https://github.com/ethz-asl/kalibr/wiki/installation)

## 标定

标定分为四步：

1. 将rosbag转换为常规hdf5格式的文件。
2. 将(1)生成的h5文件以固定频率实现图像恢复，这一步需要之前构建的`e2calib`环境。
3. 将恢复图像和IMU信息合成rosbag
4. 使用`kalibr`对(3)生成的rosbag进行标定

注意：除步骤(2)之外，其他python环境使用的都是原始环境。

### 将rosbag转换为h5

```bash
cd python/ # 请在克隆的本仓库路径下执行
python3 convert.py ${rosbag名(含路径)} -o ${h5文件名(含路径)} -t ${事件在rosbag中的topic}
```

### 图像恢复

```bash
cd python/ # 请在克隆的本仓库路径下执行
python3 offline_reconstruction.py  --h5file ${h5文件名(含路径)} --freq_hz ${保存图像的频率} --upsample_rate ${上采样率} --height ${图像高度} --width ${图像长度}
```

上采样率：举个例子，若`freq_hz`设置为5，`upsample_rate`设置为4，则生成图像频率为5hz，生成图像的窗长为 $\frac{1}{5\times4}s=0.05s$。

建议将`freq_hz`设置为5，`upsample_rate`设置为4。

图片会被默认保存至`${e2calib}/python/frames/e2calib`路径

### 合成rosbag

由于可能需要进行外参标定，所以建议使用kalibr的bagcreater，以下以两个相机+一个IMU举个例子：

保存路径设置为

```
+-- dataset-dir
    +-- cam0
    │   +-- 1385030208726607500.png
    │   +--      ...
    │   \-- 1385030212176607500.png
    +-- cam1
    │   +-- 1385030208726607500.png
    │   +--      ...
    │   \-- 1385030212176607500.png
    \-- imu0.csv
```

imu0.csv文件应使用以下格式：(timestamps=[ns], omega=[rad/s], alpha=[m/s^2])

```
timestamp,omega_x,omega_y,omega_z,alpha_x,alpha_y,alpha_z
1385030208736607488,0.5,-0.2,-0.1,8.1,-1.9,-3.3
 ...
1386030208736607488,0.5,-0.1,-0.1,8.1,-1.9,-3.3
```

通过以下命令合成rosbag（请先source kalibr工作空间的环境变量）

```bash
rosrun kalibr kalibr_bagcreater --folder dataset-dir/. --output-bag ${rosbag名}
```

按照上述得到的rosbag话题名为：

- /cam0/image_raw
- /cam1/image_raw
- /imu0

### 标定

请参考kalibr的[wiki](https://github.com/ethz-asl/kalibr/wiki)。
