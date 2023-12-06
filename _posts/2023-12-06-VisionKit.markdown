---
layout:     post
title:      "Vision Kit"
subtitle:   " \"Vision Kit简单使用\""
date:       2023-12-06 15:20:00
author:     "ifbear"
header-img: "img/home-bg.jpg"
tags:
    - iOS


---

# # Vision Kit

`Vision` 框架是一个在 iOS 和 macOS 上进行图像和面部分析的高级框架。该框架提供了一系列的工具和 API，使开发者能够轻松进行图像处理、特征检测、物体识别、文本识别以及人脸分析等任务。

### 1. 图像处理和分析

- **图像请求处理：** 使用 `VNImageRequestHandler` 处理图像请求，支持从图像、摄像头帧或 Core ML 模型中获取输入。

- **图像分类：** 通过 `VNClassifyImageRequest` 实现图像分类，将图像分为不同的类别。

  ```swift
  let handler = VNClassifyImageRequest { request, error in
      guard var results = (request.results as? [VNClassificationObservation]) else { return }
      // 按识别进度筛选结果
      results = results.filter({ $0.hasMinimumPrecision(0.5, forRecall: 0.7) })
      print(results.map(\.identifier).joined(separator: "|")) 
      // adult_cat|animal|cat|feline|mammal|clothing|headgear
  }
  guard let cgimage = UIImage(named: "cat.png")?.cgImage else { return }
  let request = VNImageRequestHandler(cgImage: cgimage)
  try? request.perform([handler])
  ```

  

- **物体检测与跟踪：** 使用 `VNDetectRectanglesRequest` 可以检测图像中的矩形区域，`VNDetectFaceRectanglesRequest` 可以检测人脸。

### 2. 人脸分析

- **人脸检测与标志：** `VNDetectFaceRectanglesRequest` 可以检测图像中的人脸，而 `VNDetectFaceLandmarksRequest` 可以检测人脸的关键特征点，如眼睛、嘴巴等。

  ```swift
  // 面部区域识别
  let request = VNDetectFaceRectanglesRequest { request, error in
      guard let results = request.results as? [VNFaceObservation] else { return }
      results.forEach { face in
          DispatchQueue.main.async {
              self.highlightFace(with: face)
          }
      }
  }
  guard let cgimage = UIImage(named: "face.jpg")?.cgImage else { return }
  let handler = VNImageRequestHandler(cgImage: cgimage, options: [:])
  try? handler.perform([request])
  ```

  <img src="https://raw.githubusercontent.com/ifbear/ifbear.github.io/main/_posts/2023-12-06-VisionKit.assets/IMG_2939.jpg" alt="IMG_2939" style="zoom:25%;" />

  ```swift
  // 显示面部识别框
  func highlightFace(with landmark: VNFaceObservation) {
      guard let source = imageView.image else { return }
      let boundary = landmark.boundingBox
  
      UIGraphicsBeginImageContextWithOptions(source.size, false, 1)
  
      let context = UIGraphicsGetCurrentContext()!
      context.setShouldAntialias(true)
      context.setAllowsAntialiasing(true)
      context.translateBy(x: 0, y: source.size.height)
      context.scaleBy(x: 1.0, y: -1.0)
      context.setLineJoin(.round)
      context.setLineCap(.round)
  
      let rect = CGRect(x: 0, y:0, width: source.size.width, height: source.size.height)
      context.draw(source.cgImage!, in: rect)
  
      let fillColor: UIColor = .green
      fillColor.setStroke()
  
      let rectangleWidth = source.size.width * boundary.size.width
      let rectangleHeight = source.size.height * boundary.size.height
      context.setLineWidth(5)
      context.addRect(CGRect(x: boundary.origin.x * source.size.width, y:boundary.origin.y * source.size.height, width:
                              rectangleWidth, height: rectangleHeight))
      context.drawPath(using: CGPathDrawingMode.stroke)
      let highlightedImage : UIImage = UIGraphicsGetImageFromCurrentImageContext()!
      UIGraphicsEndImageContext()
      DispatchQueue.main.async {
          self.imageView.image = highlightedImage
      }
  }
  ```

  ```swift
  // 面部特征
  let request = VNDetectFaceLandmarksRequest { request, error in
      guard let results = request.results as? [VNFaceObservation] else { return }
      DispatchQueue.main.async { [unowned self] in
  
          results.forEach { face in
              self.addFaceLandmarksToImage(face)
          }
      }
  }
  
  guard let cgimage = UIImage(named: "face.jpg")?.cgImage else { return }
  let handler = VNImageRequestHandler(cgImage: cgimage, options: [:])
  try? handler.perform([request])	
  ```

  ```swift
  // 绘制面部特征
  func addFaceLandmarksToImage(_ face: VNFaceObservation) {
      guard let image = imageView.image else { return }
      UIGraphicsBeginImageContextWithOptions(image.size, true, 0.0)
      let context = UIGraphicsGetCurrentContext()
  
      // draw the image
      image.draw(in: CGRect(x: 0, y: 0, width: image.size.width, height: image.size.height))
  
      context?.translateBy(x: 0, y: image.size.height)
      context?.scaleBy(x: 1.0, y: -1.0)
  
      // 面部区域
      // draw the face rect
      let w = face.boundingBox.size.width * image.size.width
      let h = face.boundingBox.size.height * image.size.height
      let x = face.boundingBox.origin.x * image.size.width
      let y = face.boundingBox.origin.y * image.size.height
      let faceRect = CGRect(x: x, y: y, width: w, height: h)
      context?.saveGState()
      context?.setStrokeColor(UIColor.red.cgColor)
      context?.setLineWidth(1.0)
      context?.addRect(faceRect)
      context?.drawPath(using: .stroke)
      context?.restoreGState()
  
      // 从脸颊到下巴到脸颊的面部轮廓的点的区域。
      // face contour
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.faceContour {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 外唇 包含描述嘴唇外部轮廓的点的区域。
      // outer lips
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.outerLips {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 嘴唇之间空间轮廓的点的区域。
      // inner lips
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.innerLips {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 左眼轮廓的点的区域
      // left eye
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.leftEye {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 右眼眼轮廓的点的区域
      // right eye
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.rightEye {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 左瞳孔所在点的区域
      // left pupil
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.leftPupil {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 右瞳孔所在点的区域
      // right pupil
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.rightPupil {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 左眉毛轨迹的点的区域
      // left eyebrow
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.leftEyebrow {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 右眉毛轨迹的点的区域
      // right eyebrow
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.rightEyebrow {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 鼻子轮廓的点的区域。
      // nose
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.nose {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.closePath()
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 鼻峰中心轨迹的点的区域
      // nose crest
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.noseCrest {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // 面部中心线轨迹的点的区域。
      // median line
      context?.saveGState()
      context?.setStrokeColor(UIColor.yellow.cgColor)
      if let landmark = face.landmarks?.medianLine {
          for i in 0...landmark.pointCount - 1 { // last point is 0,0
              let point = landmark.normalizedPoints[i] //.point(at: i)
              if i == 0 {
                  context?.move(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              } else {
                  context?.addLine(to: CGPoint(x: x + CGFloat(point.x) * w, y: y + CGFloat(point.y) * h))
              }
          }
      }
      context?.setLineWidth(1.0)
      context?.drawPath(using: .stroke)
      context?.saveGState()
  
      // get the final image
      let finalImage = UIGraphicsGetImageFromCurrentImageContext()
  
      // end drawing context
      UIGraphicsEndImageContext()
  
      imageView.image = finalImage
  }
  ```

  <img src="https://raw.githubusercontent.com/ifbear/ifbear.github.io/main/_posts/2023-12-06-VisionKit.assets/IMG_D15077B3EAFF-1-1856835.jpeg" alt="IMG_D15077B3EAFF-1" style="zoom:25%;" />

​		

- **情绪识别：** `VNRecognizeTextRequest` 可以通过 `VNFaceObservation` 获取人脸的情绪信息。

### 3. 文本识别

- **文本检测与识别：** 使用 `VNRecognizeTextRequest` 可以检测图像中的文本，并进行文本识别。

  ```swift
  let recognizeTextRequest = VNRecognizeTextRequest { request, error in
      guard let observations = request.results as? [VNRecognizedTextObservation] else { return }
      print(observations.compactMap({ $0.topCandidates(1).first?.string }).joined())
      // 前言在ioS12时，苹果推出了text .....
  }
  recognizeTextRequest.progressHandler = {request, progress, error in
      print(progress)
  }
  recognizeTextRequest.recognitionLevel = .accurate
  if #available(iOS 16.0, *) {
      recognizeTextRequest.automaticallyDetectsLanguage = true
  }
  guard let image = UIImage(named: "2.png")?.cgImage else { return }
  let request = VNImageRequestHandler(cgImage: image, options: [:])
  try? request.perform([recognizeTextRequest])
  ```

### 4. Core ML 集成

- **Core ML 模型：** `Vision` 可以与 Core ML 集成，允许使用 Core ML 模型执行更高级的任务，如图像风格转换、图像分割等。

### 5. 用于增强现实（AR）的支持

- **AR 特征检测：** `VNImageRequestHandler` 可以用于检测 ARKit 中的 AR 特征，如水平面、垂直面等。

### 6. 性能优化

- **异步处理：** 多个图像请求可以并发进行异步处理，提高处理效率。
- **Metal 支持：** 利用 Metal 框架，`Vision` 可以在 GPU 上执行计算，提高图像处理性能。



## 最后

其中部分代码参考网上实现，并经过个人验证。图片来源自网络，如果侵权可联系[本人](mailto:sdxdevin@giamil.com)删除。