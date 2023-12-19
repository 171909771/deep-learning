## 转换detection in Hierachy 为 annotation
- https://forum.image.sc/t/convert-detections-to-annotations-in-qupath/50627
```
def detections = getDetectionObjects()
def newAnnotations = detections.collect {
    return PathObjects.createAnnotationObject(it.getROI(), it.getPathClass())
}
removeObjects(detections, true)
addObjects(newAnnotations)
```
