# 2021.11  PythonOpencv-bitwise_and-TIF操作数据类型不对应错误
`水平有限，如有错误请指教！相互学习讨论`
### 学习目标
对于一个比较大的tif图片，需要从中以一些标注文件中的坐标来进行多次切割出一个个相应的目标图片，然后进行相应数据读取、分析。
建模学习……
  
### 要做到的
一般运用 Opencv 中的一些函数来进行切割，由于是不规则且根据坐标来切割，所以思路是做一个 mask ，类型一个底片，将其各个 points 坐标打上去，
用 poltlines 做一个多边形，然后运用 fillPoly 填充此多边形， mask 原本其中填充的是 0 的 numpy 数组，而切割的区域则是以 1 来替代掉原本的0数据，
最终用 bitwise_and 的数组操作来进行最后的运算，得到图片。
  
### 遇到的问题
由于之前最开始切割的jpg文件和前端时间研究的tif文件通道数上的不同导致 img.shape 属性得到的值不同，而我之前 zeros 创建的 mask 的数据结构是由
img.shape 决定， 以至加入了一个值来有存放通道， 而切割只需要坐标， 如此加入通道操作有可能会造成通道损失， 或亦是直接切割失败。
同时，我所用到的 tif 文件过大，而切割只是小小一部分，数量多，一次切割一个，每次对大图操作，时间消耗大。
tif 文件数据直接读取和 json 中解析出来的 points 坐标来直接切割似乎会有报错？报错数据类型不一致， 个人认为直接操作tif会导致此报错？
  
### 解决
但是在学习、调试了师兄的代码，发现了如果先将图片根据其矩形切割一次，获得一个小图，而小图的坐标根据其预先切割的坐标对 json 文件中解析的坐标
简单运算一次，即可得到小图对应目标边缘的坐标，用这种预先操作一次图片的方法， python 居然不会再次报出数据类型不匹配的错误！ 而且用小图来进行
bitwise_and 操作，速度大大加快。
报错原因分析只是个人拙见、猜测

---  
### 代码
此代码并没有完全完成， 由于本人要处理的 tif 文件比较多， 在执行代码中写了一个自动生成文件夹的代码存放相应的切割图， 相信在后期项目的时候可以对里面的路径也写
一个可以操作的函数， 最终目的是将整个流程做成一个可以随时可用的包。  
此为文件链接: [ProcessingImage.py](https://github.com/HabsburgGQv/Learning-Log/blob/d512649979adb0eada418595bd20ca276a539f2c/2021.11/PythonOpencv-bitwise_and-TIF%E6%93%8D%E4%BD%9C%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E4%B8%8D%E5%AF%B9%E5%BA%94%E9%94%99%E8%AF%AF/ProcessingImage.py)  

```python

import json
import os
import cv2.cv2 as cv2
import numpy as np


def analysis_tree(file_path):
    points_set_ana = []  # 保存的是一个三维的列表
    for filename in os.listdir(file_path):
        # 判断是否为json文件并记录点
        if filename.endswith(".json"):
            # endsWith() 方法用于测试字符串是否以指定的后缀结束。
            # 得到一条完整的json文件路径
            json_path_ana = os.path.join(file_path, filename)  # .join路径拼接
            # 加载出json文件
            json_data = json.load(open(json_path_ana, 'r'))
            # 获取数据
            for obj in json_data['shapes']:
                if obj['label'] == "tree":
                    points_set_ana.append(obj['points'])
    return points_set_ana


def Matting(img, Coordinate_set):
    points_set_Ma = np.array(Coordinate_set, dtype=int)  # 多边形的顶点集
    # 坐标获取
    col0 = points_set_Ma[:, 0]
    col1 = points_set_Ma[:, 1]
    x1 = np.min(col0)
    y1 = np.min(col1)
    x2 = np.max(col0)
    y2 = np.max(col1)

    single_tree = img[y1:y2, x1:x2]  # 对大图进行初次切割，得到一个包含该树的小图
    # 对坐标进行再次操作，使坐标对应于新的小图
    points_set_Ma[:, 0] -= x1
    points_set_Ma[:, 1] -= y1
    points_final = np.array([points_set_Ma], dtype=int)

    # 将坐标刻画在坐标板上
    Cp = np.zeros(single_tree.shape[:2], np.uint8)  # 获得一个与该图片有同样大小但是里面数据用0填充的数组
    Cpm = cv2.polylines(Cp, points_final, True, 255)  # 打点
    Cpf = cv2.fillPoly(Cpm, points_final, 255)
    img_tree = cv2.bitwise_and(single_tree, single_tree, mask=Cpf)
    # 作图
    # 扣出来的图的数据所形成的图片
    return img_tree


 if __name__ == '__main__':
    save_Tpa1 = r".\tree"
    img_path = r'.\SplicedImage'

    # json 路径
    json_path = r".\labelmefile"
    # 获取坐标
    points_set = analysis_tree(json_path)

    for fileName in os.listdir(img_path):
        if fileName.endswith(".tif"):
            # 得到前缀名
            frontName = fileName.split('.')[0]
            # 读取文件
            img_ini = cv2.imread(os.path.join(img_path, fileName), -1)
            # 创建保存文件夹路径
            savePath = os.path.join(save_Tpa1, frontName)
            # 创建文件夹存放切割图片
            isExist = os.path.exists(savePath)
            if not isExist:
                os.makedirs(savePath)
                print('&-*'*10 + frontName + '文件夹创建成功！' + '&-*'*10)
            else:
                print('&-*'*10 + frontName + '文件夹已经存在' + '&-*'*10)
            numName = 0
            for i in points_set:
                numName += 1
                # 创建保存的文件名
                saveName = '{0}_{1}.tif'.format(frontName, numName)
                print(saveName)
                if img_ini is not None:
                    img_data = Matting(img_ini, i)
                    print('&-*'*10 + 'LOADING' + '&-*'*10)
                    cv2.imwrite(os.path.join(savePath, saveName), img_data)
                    print('Success! ' + saveName)
                else:
                    print('IMG IS NONE')
```

