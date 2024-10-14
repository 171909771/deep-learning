## 1.1 转换detection in Hierachy 为 annotation
- https://forum.image.sc/t/convert-detections-to-annotations-in-qupath/50627
```
def detections = getDetectionObjects()
def newAnnotations = detections.collect {
    return PathObjects.createAnnotationObject(it.getROI(), it.getPathClass())
}
removeObjects(detections, true)
addObjects(newAnnotations)
```
## 1.2 specific detection to annotation
```
// Define the specific PathClass to filter
String targetPathClassName = "predictor+"

// Get all detection objects
def detections = getDetectionObjects()

// Filter detections to only include those with the specified PathClass
def filteredDetections = detections.findAll { it.getPathClass() != null && it.getPathClass().getName() == targetPathClassName }

// Convert the filtered detections to annotations
def newAnnotations = filteredDetections.collect {
    return PathObjects.createAnnotationObject(it.getROI(), it.getPathClass())
}

// Remove the filtered detection objects
removeObjects(filteredDetections, true)

// Add the new annotations to the project
addObjects(newAnnotations)
```

## 2.1 annotation to detection
```
def annotations = getAnnotationObjects()
def detections = annotations.collect {
    PathObjects.createDetectionObject(it.getROI(), it.getPathClass(), it.getMeasurementList())
}

addObjects(detections)
```

## 2.2 annotation to detection based on the classification
- https://forum.image.sc/t/convert-annotations-into-detections-in-qupath/92287/3
```
// Convert annotations to detections
def annotations = getAnnotationObjects().findAll{it.getPathClass() == getPathClass("Annotation Class")}
def newDetections = annotations.collect{
    return PathObjects.createDetectionObject(it.getROI(), it.getPathClass())
}
// removeObjects(annotations, true) // uncomment to delete original annotations
addObjects(newDetections)
```
