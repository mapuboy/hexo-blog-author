---
title: Pyqt5 图形视图框架中的坐标系
categories: 
  - Python
tags:
  - pyqt
  - python
  - qt
  - QGraphicsView
---

**转自 https://blog.csdn.net/seniorwizard/article/details/110633027**

图形视图基于笛卡尔坐标系，图元在场景中的位置和几何形状由x坐标和y坐标两个数字表示。使用未做变换的视图观察场景时，场景中的一个单位等于屏幕上的一个像素。

注意：“图形视图”使用Qt的坐标系，因此不支持反转的Y轴坐标系（向上增长）。

图形视图中有三个有效的坐标系在起作用：图元坐标，场景坐标和视图坐标。为了简化实现，“图形视图”提供了方便的函数，允许在三个坐标系之间进行映射。渲染时，图形视图的场景坐标对应于QPainter的逻辑坐标，视图坐标与设备坐标相同。

![实战PyQt5: 083-图形视图框架中的坐标系](https://img-blog.csdnimg.cn/img_convert/6d9e2f3276e217bcd1fca7a55d42e70b.png)

图片来源(doc.qt.io)

# 图元(Item)坐标

图元位于它们自己的坐标系中。它的坐标都以点(0,0)为中心点，这也是所有变换的中心点。在图元坐标系中的几何图元，经常被称为图元点，图元线，图元矩形。

当创建一个自定义的图元时,必须考虑图元的坐标。QGraphicsScene与QGraphicsView可以执行所有转换，这使得实现自定义的图元变得容易。例如，假如收到鼠标按下或是拖动输入事件，事件的位置以图元坐标的形式给出。当某个点的位置在指定的图元内时，QGraphicsItem.contain()返回True,否则返回False。这个点参数使用图元坐标，相似地，图元的边界矩形与形状也使用图元坐标。

图元位置指的是图元的中心点在其父亲的坐标系中的坐标，有时称其为父坐标。从这个意义上说，场景被视为所有无父亲图元的‘父图元’，因此顶层的图元位置就是在在场景坐标中的位置。

子坐标是相对于父坐标的，假如子图元未经变换，则子坐标与父坐标之间的差值等于在父坐标系下，父图元与子图元之间的距离。例如，假如一个未经变换的子图元位置与其父图元的中心重合，那么这两个图元的坐标系统完全相同。如果子图元的位置是（10，0），那么子图元坐标系中的(0,10)点，对应于父坐标系中的（10，10）点。

因为图元的位置与变换是相对于父图元的，子图元的坐标不会被父亲的变换影响，尽管父图元的变换隐含地对子图元做了变换。在上面的例子中，即使父图元旋转，缩放，子图元的(0,10)点依然对应于父图元的(10,10)点。然而，相对于场景来讲，子图元会遵循父图元的变换。假如父图元被缩放(2X,2X),子图元 的位置在场景中的坐标是（20，0），它的（10，0）点则与场景中的（40，0）对应 。

除QGraphicsItem.pos()函数以外，QGraphicsItem的函数在图元坐标中操作，例如一个图元的边界矩形总是以图元坐标的形式给出。

# 场景(Scene)坐标

场景代表所有图元的基本坐标系,场景坐标系统描述了每个最顶级图元的位置，也是从视图向场景传递所有场景事件的基础。场景中的每个图元有场景位置与边界矩形（QGraphicsItem::scenePos(),QGraphicsItem::sceneBoundingRect()), 另外，它有自己本地图元位置与边界矩形。场景位置描述了图元在场景坐标下的位置，它的场景边界矩形则用于QGraphicsScene决定场景中哪块区域发生了变化。场景中的变化通过QGraphicsScene.changed()信号来通知，它的参数是场景矩形列表。

# 视图(View)坐标

视图坐标是部件的坐标，视图坐标中每个单位对应一个像素。这种坐标的特殊之处在于它是相对于部件或是视口的，不会被所观察的场景所影响。QGraphicsView的视口的左上角总是（0，0），右下角总是(视口宽，视口高）。所有鼠标事件和拖放事件最初都是作为视图坐标接收的，因此需要将这些坐标映射到场景坐标以便与图元进行交互。

# 坐标映射

通常在处理场景中的图元时，要在场景与图元之间，图元与图元之间，视图与场景之间进行坐标映射。举例来讲，当在QGraphicsView的视口中点击鼠标时，应该通过调用QGraphicsView.mapToScence()与QGraphicsScene.itemAt()来获知光标下是场景中的哪个图元。假如想获知一个图元位于视口中的什么位置，应该先在图元上调用QGraphicsItem.mapToScene(), 然后调用QGraphicsView.mapFromScene()。最后，假如想知道在一个椭圆中有哪些图元,则可以将QPainterPath传递到mapToScene(),然后再把映射后的路径传递给QGraphicsScene.items()。

# 演示代码

使用多种颜色绘制螺旋线，操作下方的滑动条，可旋转这个彩色的螺旋线。完整代码如下:

``` python
import sys,math
from PyQt5 import QtCore, QtGui, QtWidgets
from PyQt5.QtCore import Qt, QPointF
from PyQt5.QtGui import QPen, QBrush, QFont, QTransform
from PyQt5.QtWidgets import (QApplication, QWidget, QGraphicsScene, QGraphicsView,
                             QVBoxLayout, QSlider, QGraphicsLineItem)
 
class DemoGvCoordinate(QWidget):
    def __init__(self, parent=None):
        super(DemoGvCoordinate, self).__init__(parent)   
        
        # 设置窗口标题
        self.setWindowTitle('实战PyQt5: Graphics View 框架坐标系统演示')      
        # 设置窗口大小
        self.resize(480, 480)
      
        self.initUi()
        
    def initUi(self):
        #场景部分
        scene = QGraphicsScene()
                
        scene.setBackgroundBrush(QBrush(Qt.black))
        
        #绘制螺旋线
        colors=[Qt.red, Qt.darkMagenta, Qt.blue, Qt.green, Qt.yellow, Qt.darkCyan]
        x0 = 0
        y0 = 0
        ratio = 0.35
        deg = 59 * math.pi / 180
        for i in range (360):
            line = QGraphicsLineItem()
            line.setPen(QPen(colors[i%6]))
            x1 = x0 + i * ratio * math.cos(i * deg)
            y1 = y0 + i * ratio * math.sin(i * deg)
            line.setLine(x0, y0, x1, y1)
            scene.addItem(line)
            x0 = x1
            y0 = y1
              
        self.view = QGraphicsView()
        self.view.setScene(scene)
        
        #滑动条
        slider = QSlider(Qt.Horizontal, self)
        slider.setRange(-360, 360)
        slider.setPageStep(10)
        slider.setValue(0)
        slider.valueChanged.connect(self.onRotateValueChanged)
        
        layout = QVBoxLayout()
        layout.addWidget(self.view)
        layout.addWidget(slider)
        
        self.setLayout(layout)
    
    def onRotateValueChanged(self, value):
        #是个累积效应，先对变化矩阵进行复位操作
        self.view.setTransform(QTransform())
        self.view.rotate(value)
    
if __name__ == '__main__':
    app = QApplication(sys.argv)
    window = DemoGvCoordinate()
    window.show()
    sys.exit(app.exec())

```


运行结果如下图：

![实战PyQt5: 083-图形视图框架中的坐标系](https://img-blog.csdnimg.cn/img_convert/6b067bca27d2689d9e9b10e90254822e.gif)

测试Graphics View框架中的坐标系

# 本文知识点

* 图形视图基于笛卡尔坐标系；
* 场景坐标，视图坐标和图元坐标；
* 场景坐标，视图坐标，图元坐标之间的映射关系；
* 螺旋线的绘制方法。