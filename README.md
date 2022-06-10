# JWST数据处理笔记

## 相关网站：

JWST[文档网站](https://jwst-docs.stsci.edu/),包含望远镜各式各样的信息

JWST[数据网站](https://www.stsci.edu/jwst/science-execution/data-analysis-toolbox)里包括处理到可以做科学的pipeline、后续的数据分析工具，以及望远镜数据处理培训网站[JWebbinars](https://www.stsci.edu/jwst/science-execution/jwebbinars), 也可以直接去JWST的[youtube网页](https://www.youtube.com/c/JWSTObserver)看个痛快

JWST[观测项目网站](https://www.stsci.edu/jwst/science-execution/approved-programs) 具体到深场巡天的项目被聚合在[这里](http://www.iap.fr/jwst-edls/fields.html)

JWST主要用[mirage](https://github.com/spacetelescope/mirage)来生成[仿真数据](https://www.stsci.edu/jwst/science-planning/proposal-planning-toolbox/simulated-data) , mirage这个软件整合了仪器特性，如果要自己运行的话需要大约三百G硬盘容量来下载仪器数据

Dan Coe 老师整理了个[从仿真到数据处理网站](https://www.dancoe.space/jwst/simulations)，预计内容会越来越丰富，他也写了个[mirage的例子](https://github.com/dancoe/mirage)

JWST[袖珍手册.pdf](https://www.stsci.edu/files/live/sites/www/files/home/jwst/instrumentation/_documents/jwst-pocket-guide.pdf)

## JWST的pipeline

我这里用来运行[pipeline](https://github.com/spacetelescope/jwst)的数据主要有：EGS天区巡天项目[CEERS](https://ceers.github.io/releases.html)的[仿真数据](https://web.corral.tacc.utexas.edu/ceersdata/)和[jwebbinars](https://www.stsci.edu/jwst/science-execution/jwebbinars)第三课的资料

两套处理流程略有不同，可以都下下来看看数据感受一下

### Pipeline安装

我用了[pipeline程序网站](https://github.com/spacetelescope/jwst) 的 [Install the DMS Operational Build](https://github.com/spacetelescope/jwst#installing-a-dms-operational-build) 的安装方案：用conda安装jwstdp的环境，然后用这个环境里的python安装jwst，asdf，requests，astropy等等

pipeline运行的时候会从服务器上下载[Calibration Reference Data System (CRDS)](https://jwst-crds.stsci.edu/)定标文件到crds_cache文件夹，所以需要在运行pipeline前指定crds_cache文件夹位置和定标文件网站到环境变量CRDS_PATH和CRDS_SERVER_URL，也可以在python里这么着：

 os.environ['CRDS_PATH'] = '~/Jobs/Astro-Code/crds_cache

 os.environ['CRDS_SERVER_URL'] = 'https://jwst-crds.stsci.edu'

这样pipeline就配置好了，可以处理数据了：

### 原始数据结构

韦伯望远镜的探测器是红外阵列，都和CMOS一样单个像素直接读出，其中NIRSpec的探测器上有能用的微快门来控制像素读出，很有特点，我这里只看完了[NIRCam](https://jwst-docs.stsci.edu/jwst-near-infrared-camera)的数据，也只测试了NIRCam的pipelie

原始[数据的后缀](https://jwst-pipeline.readthedocs.io/en/latest/jwst/introduction.html#reference-files)是uncal，一般是多层的fits，记录的从上一次复位开始到当前读出时刻的探测器计数，之后探测器复位，完成一个integration周期：

![https://jwst-docs.stsci.edu/files/97979489/97979493/1/1596073268833/Figure2.png](https://jwst-docs.stsci.edu/files/97979489/97979493/1/1596073268833/Figure2.png)

从复位开始，读出三次，每次积累四个曝光单元，每次读出的数据是从复位开始到读出时刻的总计数，用ds9可以看出来原始数据这个多层的fits的[计数会越来越亮](https://jwst-docs.stsci.edu/understanding-exposure-times)，也可以分成十二次读出等各种读出模式，[图片出处](https://jwst-docs.stsci.edu/jwst-near-infrared-spectrograph/nirspec-instrumentation/nirspec-detectors/nirspec-detector-readout)

这个fits里读出的是每个读出时刻某个像素积累的计数，如果某个读出时刻某个像素收到宇宙线的话，从读出时刻，像素计数的图上很容易看到读数有个突变，如果积分到一半儿时间的时候像素饱和，也可以用还没饱和的那些计数和积累时间来[估计源的流量)[https://www.cosmos.esa.int/documents/739790/3315704/ESA_JWST_Master_Class_Detectors_Assignment.pdf)

![数据结构](https://jwst-docs.stsci.edu/files/115769825/115769829/1/1619663725399/Data_cube.png)

原始数据名字后半部分里有_nrca1到5，编号和探测器的位置关系：

![编号和探测器的位置关系](https://jwst-docs.stsci.edu/files/97978207/97978216/1/1596073158761/NIRCam+detectors+FOV.png)

前四个探测器对应是短波近红外相机读出模式，五号探测器对应长波红外相机，视场大一些，分辨率也低一些

### 数据处理pipeline运行

![数据流](https://jwst-docs.stsci.edu/files/115769825/115769826/1/1619663725333/JWST_calibration_flow-fixed.png)

韦伯数据处理pipeline指的是从原始数据到science ready数据，分[三个步骤](https://jwst-docs.stsci.edu/jwst-science-calibration-pipeline-overview/stages-of-jwst-data-processing)，得到的数据产品也分为三个级别，对原始数据做了仪器响应修正得到的是[一级数据](https://jwst-docs.stsci.edu/jwst-science-calibration-pipeline-overview/stages-of-jwst-data-processing/calwebb_detector1)，再做定标后得到的是[二级数据](https://jwst-docs.stsci.edu/jwst-science-calibration-pipeline-overview/stages-of-jwst-data-processing/calwebb_image2)，把二级数据叠加起来得到的是[三级数据](https://jwst-docs.stsci.edu/jwst-science-calibration-pipeline-overview/stages-of-jwst-data-processing/calwebb_image3)，其中二级数据分为abc三个品种，简单的说处理到二级数据几乎就可以做科研了

![三个步骤](https://jwst-docs.stsci.edu/files/97980350/97980351/1/1596073343762/JWST_pipeline_structure.png)

韦伯的pipeline运行模式可以在python里也可以在终端里命令行，我这里用python里的模式，程序运行方法是先做一个对象，在用这个对象run：

detector1 = calwebb_detector1.Detector1Pipeline()

run_output = detector1.run(‘uncal.fits’)

image2 = calwebb_image2.Image2Pipeline()

image2.run(‘rate.fits’)

image3 = calwebb_image3.Image3Pipeline()

image3.run(‘asn_file’)

每个run之前可以给对象加参数

数据处理第一步是解决仪器响应，要把分析像素上读数和时间的关系，把坏点，饱和像素，增益，暗场等等的全都考虑进去，这一步需要用CRDS网站的设备信息，如果本地没有的话，程序会尝试从网站下载，大约3个G，处理几波数据后这个crds_cache文件夹会变得很大

![第一步](https://jwst-docs.stsci.edu/files/97980352/97980353/2/1613499228405/CALWEBB_DETECTOR1.png)

第一步做完后，得到的是rate文件

![stage1](https://github.com/chengchengcode/jwst-pipeline-note/blob/main/figures/stage1.png)

接下来是第二步，修正平场，wcs，流量定标之类的：

![第二步](https://jwst-docs.stsci.edu/files/97980355/97980356/1/1596073344709/CALWEBB_IMAGE2.png)

效果是这样的:

![stage2](https://github.com/chengchengcode/jwst-pipeline-note/blob/main/figures/stage2.png)

在CEERS项目的例子里，平场里手动加了新的结构，所以单独做了修正

第三步，把几次观测的图像合并到一起，并且会做著名的[Drizzle](https://drizzlepac.readthedocs.io/en/latest/astrodrizzle.html)这里需要提供json文件，理论上获得原始数据的时候也会得到这个文件

![第三步](https://jwst-docs.stsci.edu/files/97980367/97980368/2/1613499229973/CALWEBB_IMAGE3.png)

然后运行一下第三步就会得到：

![stage3](https://github.com/chengchengcode/jwst-pipeline-note/blob/main/figures/stage3.png)
