### batch for detections
```
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import qupath.lib.gui.scripting.QPEx
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException

// Get the project and list of images
def project = getProject()
def images = project.getImageList()

// Prompt user to select the directory containing the CSV files
def csvDir = Dialogs.getChooser(null).promptForDirectory("Select directory containing CSV files", null)
if (csvDir == null) {
    Dialogs.showErrorMessage("No directory selected", "You must select a directory containing CSV files.")
    return
}

// Iterate over each image in the project
for (entry in images) {
    def imageName = entry.getImageName()
    
    // Handle image names without an extension
    def dotIndex = imageName.lastIndexOf('.')
    def baseName
    if (dotIndex == -1) {
        baseName = imageName
    } else {
        baseName = imageName.substring(0, dotIndex)
    }
    
    def csvFile = new File(csvDir, baseName + ".csv")
    if (!csvFile.exists()) {
        Dialogs.showErrorMessage("CSV file not found", "CSV file not found for image " + imageName)
        continue
    }

    // Open the image
    def imageData = entry.readImageData()

    // Process the CSV file and create detections
    def detections = processCSVFile(csvFile)

    // Add the detections to the image
    imageData.getHierarchy().addObjects(detections) // Updated method

    // Update the hierarchy for the image data
    imageData.getHierarchy().fireHierarchyChangedEvent(this) // Pass 'this' as the source

    // Save the image data back to the project
    entry.saveImageData(imageData)
}

// Function to process the CSV file and create detections
def processCSVFile(csvfile) {
    int z = 0
    int t = 0
    def plane = ImagePlane.getPlane(z, t)
    def pathclass0 = PathClassFactory.getPathClass("predictor-", 0x800000)
    def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)
    def detections = []

    try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
        String DELIMITER = ","
        String line
        String[] headers
        int i

        // Read CSV header line
        if ((line = br.readLine()) != null) {
            headers = line.split(DELIMITER)
            if (headers.length < 5 ||
                !headers[0].equals("x") || !headers[1].equals("y") ||
                !headers[2].equals("width") || !headers[3].equals("height")) {
                Dialogs.showErrorMessage(
                    "CSV error", "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1,...'")
                return detections
            }
        } else {
            Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
            return detections
        }

        // Create dummy detection objects to set measurement limits
        def roi = ROIs.createRectangleROI(0, 0, 1, 1, plane)
        def detection = PathObjects.createDetectionObject(roi, pathclass0)
        for (i = 4; i < headers.length; i++) {
            detection.getMeasurementList().putMeasurement(headers[i], 0.0)
        }
        detection.getMeasurementList().close()
        detections.add(detection)

        roi = ROIs.createRectangleROI(0, 0, 1, 1, plane)
        detection = PathObjects.createDetectionObject(roi, pathclass1)
        for (i = 4; i < headers.length; i++) {
            detection.getMeasurementList().putMeasurement(headers[i], 1.0)
        }
        detection.getMeasurementList().close()
        detections.add(detection)

        int linenum = 1
        while ((line = br.readLine()) != null) {
            linenum++
            String[] columns = line.split(DELIMITER)
            if (columns.length != headers.length) {
                Dialogs.showErrorMessage("CSV file error",
                    "CSV file line " + linenum + " has incorrect number of fields")
                break
            }

            int x = Integer.parseInt(columns[0])
            int y = Integer.parseInt(columns[1])
            int width = Integer.parseInt(columns[2])
            int height = Integer.parseInt(columns[3])

            roi = ROIs.createRectangleROI(x, y, width, height, plane)

            // Use the first value to determine the PathClass
            float value = Float.parseFloat(columns[4])
            if (value < 0.5) {
                detection = PathObjects.createDetectionObject(roi, pathclass0)
            } else {
                detection = PathObjects.createDetectionObject(roi, pathclass1)
            }

            for (i = 4; i < headers.length; i++) {
                value = Float.parseFloat(columns[i])
                detection.getMeasurementList().putMeasurement(headers[i], value)
            }
            detection.getMeasurementList().close()
            detections.add(detection)
        }
    } catch (IOException ex) {
        Dialogs.showErrorMessage("File open error", ex.getMessage())
    }
    return detections
}

```

### batch for annotations
```
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException
import java.io.File

// Get the project and list of images
def project = getProject()
def images = project.getImageList()

// Prompt user to select the directory containing the CSV files
def csvDir = Dialogs.getChooser(null).promptForDirectory("Select directory containing CSV files", null)
if (csvDir == null) {
    Dialogs.showErrorMessage("No directory selected", "You must select a directory containing CSV files.")
    return
}

// Iterate over each image in the project
for (entry in images) {
    def imageName = entry.getImageName()
    
    // Handle image names without an extension
    def dotIndex = imageName.lastIndexOf('.')
    def baseName
    if (dotIndex == -1) {
        baseName = imageName
    } else {
        baseName = imageName.substring(0, dotIndex)
    }
    
    def csvFile = new File(csvDir, baseName + ".csv")
    if (!csvFile.exists()) {
        Dialogs.showErrorMessage("CSV file not found", "CSV file not found for image " + imageName)
        continue
    }

    // Open the image
    def imageData = entry.readImageData()

    // Process the CSV file and create annotations
    def annotations = processCSVFile(csvFile)

    // Add the annotations to the image
    imageData.getHierarchy().addObjects(annotations)

    // Notify QuPath of the hierarchy changes
    imageData.getHierarchy().fireHierarchyChangedEvent(this)

    // Save the image data back to the project
    entry.saveImageData(imageData)
}

// Function to process the CSV file and create annotations
def processCSVFile(csvfile) {
    int z = 0
    int t = 0
    def plane = ImagePlane.getPlane(z, t)
    def pathclass0 = PathClassFactory.getPathClass("predictor-", 0x800000)
    def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)
    def annotations = []

    try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
        String DELIMITER = ","
        String line
        String[] headers
        int i

        // Read CSV header line
        if ((line = br.readLine()) != null) {
            headers = line.split(DELIMITER)
            if (headers.length < 5 ||
                !headers[0].equals("x") || !headers[1].equals("y") ||
                !headers[2].equals("width") || !headers[3].equals("height")) {
                Dialogs.showErrorMessage(
                    "CSV error", "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1,...'")
                return annotations
            }
        } else {
            Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
            return annotations
        }

        // Create dummy annotation objects to set measurement limits
        def roi = ROIs.createRectangleROI(0, 0, 1, 1, plane)
        def annotation = PathObjects.createAnnotationObject(roi, pathclass0)
        for (i = 4; i < headers.length; i++) {
            annotation.getMeasurementList().putMeasurement(headers[i], 0.0)
        }
        annotation.getMeasurementList().close()
        annotations.add(annotation)

        roi = ROIs.createRectangleROI(0, 0, 1, 1, plane)
        annotation = PathObjects.createAnnotationObject(roi, pathclass1)
        for (i = 4; i < headers.length; i++) {
            annotation.getMeasurementList().putMeasurement(headers[i], 1.0)
        }
        annotation.getMeasurementList().close()
        annotations.add(annotation)

        int linenum = 1
        while ((line = br.readLine()) != null) {
            linenum++
            String[] columns = line.split(DELIMITER)
            if (columns.length != headers.length) {
                Dialogs.showErrorMessage("CSV file error",
                    "CSV file line " + linenum + " has incorrect number of fields")
                break
            }

            int x = Integer.parseInt(columns[0])
            int y = Integer.parseInt(columns[1])
            int width = Integer.parseInt(columns[2])
            int height = Integer.parseInt(columns[3])

            roi = ROIs.createRectangleROI(x, y, width, height, plane)

            // Use the first value to determine the PathClass
            float value = Float.parseFloat(columns[4])
            if (value < 0.5) {
                annotation = PathObjects.createAnnotationObject(roi, pathclass0)
            } else {
                annotation = PathObjects.createAnnotationObject(roi, pathclass1)
            }

            for (i = 4; i < headers.length; i++) {
                value = Float.parseFloat(columns[i])
                annotation.getMeasurementList().putMeasurement(headers[i], value)
            }
            annotation.getMeasurementList().close()
            annotations.add(annotation)
        }
    } catch (IOException ex) {
        Dialogs.showErrorMessage("File open error", ex.getMessage())
    }
    return annotations
}

```

### batch for annotations and merge
```
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.roi.RoiTools
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException
import java.io.File

// Get the project and list of images
def project = getProject()
def images = project.getImageList()

// Prompt user to select the directory containing the CSV files
def csvDir = Dialogs.getChooser(null).promptForDirectory("Select directory containing CSV files", null)
if (csvDir == null) {
    Dialogs.showErrorMessage("No directory selected", "You must select a directory containing CSV files.")
    return
}

// Iterate over each image in the project
for (entry in images) {
    def imageName = entry.getImageName()

    // Handle image names without an extension
    def dotIndex = imageName.lastIndexOf('.')
    def baseName
    if (dotIndex == -1) {
        baseName = imageName
    } else {
        baseName = imageName.substring(0, dotIndex)
    }

    def csvFile = new File(csvDir, baseName + ".csv")
    if (!csvFile.exists()) {
        Dialogs.showErrorMessage("CSV file not found", "CSV file not found for image " + imageName)
        continue
    }

    // Open the image
    def imageData = entry.readImageData()

    // Process the CSV file and create annotations
    def annotations = processCSVFile(csvFile)

    // Add the annotations to the image
    imageData.getHierarchy().addObjects(annotations)

    // Merge the annotations for the current image
    mergeAnnotations(imageData)

    // Notify QuPath of the hierarchy changes
    imageData.getHierarchy().fireHierarchyChangedEvent(this)

    // Save the image data back to the project
    entry.saveImageData(imageData)
}

// Function to process the CSV file and create annotations
def processCSVFile(csvfile) {
    int z = 0
    int t = 0
    def plane = ImagePlane.getPlane(z, t)
    def pathclass0 = PathClassFactory.getPathClass("predictor-", 0x800000)
    def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)
    def annotations = []

    try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
        String DELIMITER = ","
        String line
        String[] headers
        int i

        // Read CSV header line
        if ((line = br.readLine()) != null) {
            headers = line.split(DELIMITER)
            if (headers.length < 5 ||
                !headers[0].equals("x") || !headers[1].equals("y") ||
                !headers[2].equals("width") || !headers[3].equals("height")) {
                Dialogs.showErrorMessage(
                    "CSV error", "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1,...'")
                return annotations
            }
        } else {
            Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
            return annotations
        }

        int linenum = 1
        while ((line = br.readLine()) != null) {
            linenum++
            String[] columns = line.split(DELIMITER)
            if (columns.length != headers.length) {
                Dialogs.showErrorMessage("CSV file error",
                    "CSV file line " + linenum + " has incorrect number of fields")
                break
            }

            int x = Integer.parseInt(columns[0])
            int y = Integer.parseInt(columns[1])
            int width = Integer.parseInt(columns[2])
            int height = Integer.parseInt(columns[3])

            def roi = ROIs.createRectangleROI(x, y, width, height, plane)

            // Use the first value to determine the PathClass
            float value = Float.parseFloat(columns[4])
            def annotation
            if (value < 0.5) {
                annotation = PathObjects.createAnnotationObject(roi, pathclass0)
            } else {
                annotation = PathObjects.createAnnotationObject(roi, pathclass1)
            }

            for (i = 4; i < headers.length; i++) {
                value = Float.parseFloat(columns[i])
                annotation.getMeasurementList().putMeasurement(headers[i], value)
            }
            annotation.getMeasurementList().close()
            annotations.add(annotation)
        }
    } catch (IOException ex) {
        Dialogs.showErrorMessage("File open error", ex.getMessage())
    }
    return annotations
}

// Function to merge annotations for the current image
def mergeAnnotations(imageData) {
    def hierarchy = imageData.getHierarchy()
    def annotationsToMerge = hierarchy.getAnnotationObjects()
    if (annotationsToMerge.isEmpty()) {
        Dialogs.showErrorMessage("Merge error", "No annotations to merge for image " + imageData.getServer().getMetadata().getName())
        return
    }

    // Group annotations by their class
    def classGroups = annotationsToMerge.groupBy { it.getPathClass() }

    // Create a list to hold merged annotations
    def mergedAnnotations = []

    classGroups.each { pathClass, annotations ->
        // Merge ROIs of annotations with the same class
        def rois = annotations.collect { it.getROI() }
        def mergedROI = RoiTools.union(rois)

        // Create a new annotation with the merged ROI and the same class
        def mergedAnnotation = PathObjects.createAnnotationObject(mergedROI, pathClass)

        // Optionally, aggregate measurements if needed
        // For simplicity, we're not aggregating measurements here

        mergedAnnotations.add(mergedAnnotation)
    }

    // Remove the original annotations
    hierarchy.removeObjects(annotationsToMerge, true)

    // Add the merged annotations
    hierarchy.addPathObjects(mergedAnnotations)
}
```

### batch for annotations(predictor+) and merge
```
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.roi.RoiTools
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException
import java.io.File
import groovy.transform.Field  // Import the Field annotation

// Get the project and list of images
def project = getProject()
def images = project.getImageList()

// Define the predictor+ PathClass at the top level and annotate with @Field
@Field
def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)

// Prompt user to select the directory containing the CSV files
def csvDir = Dialogs.getChooser(null).promptForDirectory("Select directory containing CSV files", null)
if (csvDir == null) {
    Dialogs.showErrorMessage("No directory selected", "You must select a directory containing CSV files.")
    return
}

// Iterate over each image in the project
for (entry in images) {
    def imageName = entry.getImageName()

    // Handle image names without an extension
    def dotIndex = imageName.lastIndexOf('.')
    def baseName
    if (dotIndex == -1) {
        baseName = imageName
    } else {
        baseName = imageName.substring(0, dotIndex)
    }

    def csvFile = new File(csvDir, baseName + ".csv")
    if (!csvFile.exists()) {
        Dialogs.showErrorMessage("CSV file not found", "CSV file not found for image " + imageName)
        continue
    }

    // Open the image
    def imageData = entry.readImageData()

    // Process the CSV file and create annotations
    def annotations = processCSVFile(csvFile)

    // Add the annotations to the image
    imageData.getHierarchy().addObjects(annotations)

    // Merge the annotations for the current image
    mergeAnnotations(imageData)

    // Notify QuPath of the hierarchy changes
    imageData.getHierarchy().fireHierarchyChangedEvent(this)

    // Save the image data back to the project
    entry.saveImageData(imageData)
}

// Function to process the CSV file and create annotations
def processCSVFile(csvfile) {
    int z = 0
    int t = 0
    def plane = ImagePlane.getPlane(z, t)
    def annotations = []

    try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
        String DELIMITER = ","
        String line
        String[] headers
        int i

        // Read CSV header line
        if ((line = br.readLine()) != null) {
            headers = line.split(DELIMITER)
            if (headers.length < 5 ||
                !headers[0].equals("x") || !headers[1].equals("y") ||
                !headers[2].equals("width") || !headers[3].equals("height")) {
                Dialogs.showErrorMessage(
                    "CSV error", "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1,...'")
                return annotations
            }
        } else {
            Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
            return annotations
        }

        int linenum = 1
        while ((line = br.readLine()) != null) {
            linenum++
            String[] columns = line.split(DELIMITER)
            if (columns.length != headers.length) {
                Dialogs.showErrorMessage("CSV file error",
                    "CSV file line " + linenum + " has incorrect number of fields")
                break
            }

            int x = Integer.parseInt(columns[0])
            int y = Integer.parseInt(columns[1])
            int width = Integer.parseInt(columns[2])
            int height = Integer.parseInt(columns[3])

            // Use the first value to determine if we create an annotation
            float value = Float.parseFloat(columns[4])
            if (value >= 0.5) {
                def roi = ROIs.createRectangleROI(x, y, width, height, plane)
                def annotation = PathObjects.createAnnotationObject(roi, pathclass1)

                for (i = 4; i < headers.length; i++) {
                    value = Float.parseFloat(columns[i])
                    annotation.getMeasurementList().putMeasurement(headers[i], value)
                }
                annotation.getMeasurementList().close()
                annotations.add(annotation)
            }
        }
    } catch (IOException ex) {
        Dialogs.showErrorMessage("File open error", ex.getMessage())
    }
    return annotations
}

// Function to merge predictor+ annotations for the current image
def mergeAnnotations(imageData) {
    def hierarchy = imageData.getHierarchy()
    // Get only the predictor+ annotations
    def annotationsToMerge = hierarchy.getAnnotationObjects().findAll { it.getPathClass() == pathclass1 }
    if (annotationsToMerge.isEmpty()) {
        Dialogs.showErrorMessage("Merge error", "No predictor+ annotations to merge for image " + imageData.getServer().getMetadata().getName())
        return
    }

    // Merge ROIs of annotations with the predictor+ class
    def rois = annotationsToMerge.collect { it.getROI() }
    def mergedROI = RoiTools.union(rois)

    // Create a new annotation with the merged ROI and the same class
    def mergedAnnotation = PathObjects.createAnnotationObject(mergedROI, pathclass1)

    // Remove the original annotations
    hierarchy.removeObjects(annotationsToMerge, true)

    // Add the merged annotation
    hierarchy.addPathObject(mergedAnnotation)
}
```

### remove all annotations in a project
```

// Import necessary QuPath classes
import qupath.lib.objects.PathAnnotationObject
import qupath.lib.gui.dialogs.Dialogs
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory

// Get the current project
def project = getProject()

if (project == null) {
    Dialogs.showErrorMessage("No Project", "Please open a project before running this script.")
    return
}

// Get the list of images (slides) in the project
def imageList = project.getImageList()

if (imageList.isEmpty()) {
    Dialogs.showPlainMessage("No Images Found", "The project contains no images.")
    return
}

// Confirm with the user before proceeding
def proceed = Dialogs.showConfirmDialog(
    "Confirm Deletion", 
    "This script will remove ALL annotations from all images in the project.\n" +
    "Are you sure you want to proceed?"
)

if (!proceed) {
    Dialogs.showPlainMessage("Operation Cancelled", "No annotations were removed.")
    return
}

int totalAnnotationsRemoved = 0

// Iterate over each image in the project
for (def entry : imageList) {
    def imageName = entry.getImageName()
    print "Processing image: ${imageName}\n"
    
    try {
        // Read the image data
        def imageData = entry.readImageData()
        def hierarchy = imageData.getHierarchy()
        
        // Get all annotation objects
        def annotations = hierarchy.getAnnotationObjects()
        int numAnnotations = annotations.size()
        
        if (numAnnotations > 0) {
            // Remove all annotations
            hierarchy.removeObjects(annotations, true)
            totalAnnotationsRemoved += numAnnotations
            print "Removed ${numAnnotations} annotations from image: ${imageName}\n"
        } else {
            print "No annotations found in image: ${imageName}\n"
        }
        
        // Notify QuPath of the hierarchy changes
        hierarchy.fireHierarchyChangedEvent(this)
        
        // Save the updated image data back to the project
        entry.saveImageData(imageData)
    } catch (Exception e) {
        // Convert GString to String using concatenation
        Dialogs.showErrorMessage(
            "Error Processing Image", 
            "An error occurred while processing image '" + imageName + "':\n" + e.getMessage()
        )
    }
}

// Convert GString to String using concatenation
Dialogs.showPlainMessage(
    "Annotation Removal Complete", 
    "Total annotations removed: " + totalAnnotationsRemoved
)
print "Annotation removal process completed. Total annotations removed: " + totalAnnotationsRemoved + "\n"
```
