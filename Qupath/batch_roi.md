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
