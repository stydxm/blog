---
title: 整活：玩游戏时，在你笑的时候使用AI自动瞄准
date: 2022-10-08 20:56:46
categories: 技术
tags: AI
cover: https://s1.ax1x.com/2022/10/08/xGowDA.png
---

前两天同学聊天的时候发了个[视频](https://www.bilibili.com/video/BV1Kd4y1i7Nf)，还提了个很有趣的思路——在笑的时候自动瞄准  
一看视频挺有意思的，实现应该也不难，试了一下不到一小时就做完了

<!--more-->

![](https://s1.ax1x.com/2022/10/08/xGowDA.png)
![](https://s1.ax1x.com/2022/10/08/xGHqxS.png)

# 思路
- 读取摄像头输入，检测表情
- 检测到笑了之后，开始捕捉屏幕画面
- 在屏幕画面中识别人形，并读取坐标
- 如果能识别到人，就将鼠标移到头的位置

# 实现
## 技术栈
### 识别表情
看视频里的实现方法[^1]简单而且效果不差，懒得再去找更好的方案了，就直接拿来用吧
```python
# 调用电脑摄像头进行实时人脸+眼睛+微笑识别，可直接复制粘贴运行
# bilibili视频教程:同济子豪兄
# 2019-5-16
import cv2

face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades+'haarcascade_frontalface_default.xml')

eye_cascade = cv2.CascadeClassifier(cv2.data.haarcascades+'haarcascade_eye.xml')

smile_cascade = cv2.CascadeClassifier(cv2.data.haarcascades+'haarcascade_smile.xml')
# 调用摄像头
cap = cv2.VideoCapture(0)

while(True):
    # 获取摄像头拍摄到的画面
    ret, frame = cap.read()
    faces = face_cascade.detectMultiScale(frame, 1.3, 2)
    img = frame
    for (x,y,w,h) in faces:
    	# 画出人脸框，蓝色，画笔宽度微
        img = cv2.rectangle(img,(x,y),(x+w,y+h),(255,0,0),2)
    	# 框选出人脸区域，在人脸区域而不是全图中进行人眼检测，节省计算资源
        face_area = img[y:y+h, x:x+w]
        
        ## 人眼检测
        # 用人眼级联分类器引擎在人脸区域进行人眼识别，返回的eyes为眼睛坐标列表
        eyes = eye_cascade.detectMultiScale(face_area,1.3,10)
        for (ex,ey,ew,eh) in eyes:
            #画出人眼框，绿色，画笔宽度为1
            cv2.rectangle(face_area,(ex,ey),(ex+ew,ey+eh),(0,255,0),1)
        
        ## 微笑检测
        # 用微笑级联分类器引擎在人脸区域进行人眼识别，返回的eyes为眼睛坐标列表
        smiles = smile_cascade.detectMultiScale(face_area,scaleFactor= 1.16,minNeighbors=65,minSize=(25, 25),flags=cv2.CASCADE_SCALE_IMAGE)
        for (ex,ey,ew,eh) in smiles:
            #画出微笑框，红色（BGR色彩体系），画笔宽度为1
            cv2.rectangle(face_area,(ex,ey),(ex+ew,ey+eh),(0,0,255),1)
            cv2.putText(img,'Smile',(x,y-7), 3, 1.2, (0, 0, 255), 2, cv2.LINE_AA)
        
	# 实时展示效果画面
    cv2.imshow('frame2',img)
    # 每5毫秒监听一次键盘动作
    if cv2.waitKey(5) & 0xFF == ord('q'):
        break

# 最后，关闭所有窗口
cap.release()
cv2.destroyAllWindows()
```
### 识别人体和骨骼绑定
暑假里搞了几天[MediaPipe](https://mediapipe.dev/)，不敢说有多精通，至少是搞明白怎么用了，所以毫不犹豫地选了[MediaPipe Pose](https://google.github.io/mediapipe/solutions/pose)
```python
import cv2
import mediapipe as mp
mp_drawing = mp.solutions.drawing_utils
mp_drawing_styles = mp.solutions.drawing_styles
mp_pose = mp.solutions.pose

cap = cv2.VideoCapture(0)
with mp_pose.Pose(
    min_detection_confidence=0.5,
    min_tracking_confidence=0.5) as pose:
  while cap.isOpened():
    success, image = cap.read()
    if not success:
      print("Ignoring empty camera frame.")
      # If loading a video, use 'break' instead of 'continue'.
      continue

    # To improve performance, optionally mark the image as not writeable to
    # pass by reference.
    image.flags.writeable = False
    image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
    results = pose.process(image)

    # Draw the pose annotation on the image.
    image.flags.writeable = True
    image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)
    mp_drawing.draw_landmarks(
        image,
        results.pose_landmarks,
        mp_pose.POSE_CONNECTIONS,
        landmark_drawing_spec=mp_drawing_styles.get_default_pose_landmarks_style())
    # Flip the image horizontally for a selfie-view display.
    cv2.imshow('MediaPipe Pose', cv2.flip(image, 1))
    if cv2.waitKey(5) & 0xFF == 27:
      break
cap.release()

```
### 捕获屏幕
随便Google了一下，找了个叫[mss](https://github.com/BoboTiG/python-mss)的库
```python
from mss import mss
sct = mss()
sct_cap = sct.grab({"top": 0, "left": 0, "width": 1920, "height": 1080})
```

### 移动鼠标
这个也没去找，视频里的代码一眼看到了[PyAutoGUI](https://github.com/asweigart/pyautogui)
```python
import pyautogui
pyautogui.moveTo(x, y)
```
## 代码
注释写的比较详细，基本每步都有注释
```python
import cv2
import numpy as np
import mediapipe as mp
import pyautogui
from mss import mss

face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')
eye_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_eye.xml')
smile_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_smile.xml')
# 调用摄像头
cap = cv2.VideoCapture(0)
# 获取屏幕
sct = mss()

while True:
    # 获取摄像头拍摄到的画面
    ret, img = cap.read()
    # 获取屏幕图像
    sct_cap = np.array(sct.grab({"top": 0, "left": 0, "width": 1920, "height": 1080}))
    faces = face_cascade.detectMultiScale(img, 1.3, 2)
    for (x, y, w, h) in faces:
        # 画出蓝色人脸框
        img = cv2.rectangle(img, (x, y), (x + w, y + h), (255, 0, 0), 2)
        # 框选出人脸区域，在人脸区域而不是全图中进行人眼检测，节省计算资源
        face_area = img[y:y + h, x:x + w]
        # 用人眼级联分类器引擎在人脸区域进行人眼识别，返回的eyes为眼睛坐标列表
        eyes = eye_cascade.detectMultiScale(face_area, 1.3, 10)
        for (ex, ey, ew, eh) in eyes:
            # 画出人眼框，绿色，画笔宽度为1
            cv2.rectangle(face_area, (ex, ey), (ex + ew, ey + eh), (0, 255, 0), 1)
        # 用微笑级联分类器引擎在人脸区域进行人眼识别，返回的eyes为眼睛坐标列表
        smiles = smile_cascade.detectMultiScale(face_area, scaleFactor=1.16, minNeighbors=65, minSize=(25, 25),
                                                flags=cv2.CASCADE_SCALE_IMAGE)
        for (ex, ey, ew, eh) in smiles:
            # 画出微笑框，红色（BGR色彩体系），画笔宽度为1
            cv2.rectangle(face_area, (ex, ey), (ex + ew, ey + eh), (0, 0, 255), 1)
            cv2.putText(img, 'Smile', (x, y - 7), 3, 1.2, (0, 0, 255), 2, cv2.LINE_AA)

        # 当摄像头画面中检测到微笑时检测屏幕图像
        if len(smiles) > 0:
            # 检测人体并获取骨骼
            pose = mp.solutions.pose.Pose(
                min_detection_confidence=0.5,
                min_tracking_confidence=0.5)
            sct_cap.flags.writeable = False
            sct_cap = cv2.cvtColor(sct_cap, cv2.COLOR_BGR2RGB)
            results = pose.process(sct_cap)
            sct_cap.flags.writeable = True
            sct_cap = cv2.cvtColor(sct_cap, cv2.COLOR_RGB2BGR)
            # 绘制骨骼
            mp.solutions.drawing_utils.draw_landmarks(
                sct_cap,
                results.pose_landmarks,
                mp.solutions.pose.POSE_CONNECTIONS,
                landmark_drawing_spec=mp.solutions.drawing_styles.get_default_pose_landmarks_style())
            # 当检测到人体时
            if results.pose_landmarks:
                # 获取0号地标
                # 地标编号参考：https://mediapipe.dev/images/mobile/pose_tracking_full_body_landmarks.png
                landmark = results.pose_landmarks.landmark[0]
                x = sct_cap.shape[1] * landmark.x
                y = sct_cap.shape[0] * landmark.y
                # 打印坐标
                print(x, y)
                cv2.circle(sct_cap, (int(x), int(y)), 10, (0, 0, 255), -1)
                # 将鼠标移到对应坐标
                pyautogui.moveTo(x, y)

    # 实时展示效果画面
    cv2.imshow('frame2', img)
    cv2.imshow('screen', sct_cap)
    # 每5毫秒监听一次键盘动作，按Q时退出
    if cv2.waitKey(5) & 0xFF == ord('q'):
        break

# 关闭所有窗口
cap.release()
cv2.destroyAllWindows()

```

# 效果
在游戏里没敢试，因为怕被反作弊识别然后号没了  
识别表情、识别人体都没啥问题，也能正确地把鼠标移到屏幕里人的头上  
## 表情
找了各种肤色、情绪的笑脸照片，都能正确识别，唯一的问题是图片放到摄像头之后要过几秒钟才能识别出微笑，不过也没什么太大问题了
## 人体姿态
为了模拟真实情况 ~~谁真去用啊~~ ，不仅试了常规照片，玩过的几个游戏（GTA5，BF5，RDR2等）截图也试了一下，人物靠得近一点就没什么没问题了（太小的识别不到）  
~~Steve和Alex都识别不了，差评~~

# 展望
检测微笑的卷积神经网络性能很好，至少在我的机器上完全观察不到卡顿；但mediapipe的pose模型检测1080P的画面时，在我的5800H上目测最多只有十几帧  
mediapipe的模型是跑在CPU上的，我电脑上还有一块3070 mobile，所以我的想法是使用GPU加速，虽然文档里有GPU加速相关的内容[^2]，但是写的是面对Linux的，可惜大多数游戏在Linux上无法运行或体验很不好  
并且，mediapipe的模型是TF lite的，而TF lite主要面向移动设备，并没有计划开发windows平台的GPU加速[^3]，所以未来如果要解决这个问题，我的想法是使用性能更好的模型或者有较完善GPU加速支持的模型

[^1]: [第十章：物体检测与人脸识别【子豪兄opencv-python教程】](https://github.com/TommyZihao/zihaoopencv/blob/master/%E7%AC%AC%E5%8D%81%E7%AB%A0%EF%BC%9A%E7%89%A9%E4%BD%93%E6%A3%80%E6%B5%8B%E4%B8%8E%E4%BA%BA%E8%84%B8%E8%AF%86%E5%88%AB%E3%80%90%E5%AD%90%E8%B1%AA%E5%85%84opencv-python%E6%95%99%E7%A8%8B%E3%80%91.md)

[^2]: [GPU support](https://google.github.io/mediapipe/getting_started/gpu_support.html)

[^3]: [tensorflow contributor的comment](https://github.com/tensorflow/tensorflow/issues/40325#issuecomment-642143623)