---
title: Pyqt5 QGraphicsView的一些坑
categories: 
  - Python
---


## QGraphicsView

### View

#### 基本设置
``` python
# 设置放大缩小时跟随鼠标
self.setTransformationAnchor(QGraphicsView.AnchorUnderMouse)
self.setResizeAnchor(QGraphicsView.AnchorUnderMouse)
self.setAttribute(Qt.WA_AcceptTouchEvents, True)
QCoreApplication.setAttribute(Qt.AA_SynthesizeTouchForUnhandledMouseEvents, True)  # 禁用将触摸事件转为鼠标事件
QCoreApplication.setAttribute(Qt.AA_SynthesizeMouseForUnhandledTouchEvents, True)

# 监听手势
self.viewport().grabGesture(Qt.PinchGesture)
self.viewport().grabGesture(Qt.TapGesture)
self.viewport().setMouseTracking(True)
```

####  监听手势来的缩放

``` python
    def viewportEvent(self, event: QEvent):
        if event.type() == QEvent.Gesture:
            return self.gesture_event(event)
        return super().viewportEvent(event)

    def gesture_event(self, event: QGestureEvent) -> bool:
        if not self.zoom_mode:
            return True
        pinch: QGesture = event.gesture(Qt.PinchGesture)
        if pinch is not None:
            zoom_factor = pinch.totalScaleFactor()
            # self.setTransformationAnchor(QGraphicsView.NoAnchor)
            # self.setResizeAnchor(QGraphicsView.NoAnchor)
            print('zoom_factor', zoom_factor)
            self.get_scale()
            if self.current_ratio > 3:
                self.scale(0.9, 0.9)
            elif self.current_ratio < 0.1:
                self.scale(1.1, 1.1)
            else:
                self.scale((zoom_factor - 1) / 10 + 1, (zoom_factor - 1) / 10 + 1)
        return True
```
  
  
### Scene

``` python
self.scene.setFocusOnTouch(True)

```


### Item


#### 放大镜

``` python
width = self.magnifier_width / self.current_ratio

pix = self.pixmap().copy(x - width / 3, y - width / 3, width * 2 / 3, width * 2 / 3)
tr = QTransform()
tr.translate(x - width, y - width)
tr.scale(3.0, 3.0)  # 设置3倍放大

brush = QBrush(pix)
brush.setTransform(tr)
painter.setBrush(brush)
painter.drawEllipse(x - width / 2, y - width / 2, width, width)
```

