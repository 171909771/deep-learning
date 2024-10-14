#### 1.2 (Binary-calssification) Loading the .csv file into detections in Qupath
```
/* Read a CSV file containing detection objects and measurement values for them in the range 0 to 1 */

import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException
import java.lang.Integer
import java.lang.Float

/* The CSV header line should be as follows:
 * x,y,width,height,VALUENAME1,VALUENAME2,...
 * The VALUENAMEs are used as the labels for the detection object measurements.
 * 
 * The first value is used to label each object as a positive detection (>=0.5)
 * or negative detection (<0.5).
 */

/* Get the location of the CSV file */ 
def csvfile = Dialogs.getChooser(null).promptForFile("Load detection CSV file",
                                                     null, "CSV files", new String[]{".csv"})

int z = 0
int t = 0
def plane = ImagePlane.getPlane(z, t)
def pathclass0 = PathClassFactory.getPathClass("predictor-", 0x800000)
def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)
def roi = ROIs.createRectangleROI(0, 0, 0, 0, plane)
def detection = PathObjects.createDetectionObject(roi, pathclass0)

// create a reader
try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
    // CSV file delimiter
    String DELIMITER = ","
    String line, valuename
    String[] headers
    int i, x, y, width, height
    float value
    def detections = []

    // read CSV header line
    if ((line = br.readLine()) != null) {
        headers = line.split(DELIMITER)
        if (headers.length < 5 ||
            ! headers[0].equals("x") || ! headers[1].equals("y") ||
            ! headers[2].equals("width") || ! headers[3].equals("height")) {
            Dialogs.showErrorMessage(
                "CSV error", "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1,...'")
            return
        }
    } else {
        Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
        return
    }

    // Create dummy detection objects to make the measurement limits 0 and 1
    roi = ROIs.createRectangleROI(0, 0, 1, 1, plane)
    detection = PathObjects.createDetectionObject(roi, pathclass0)
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

        x = Integer.parseInt(columns[0])
        y = Integer.parseInt(columns[1])
        width = Integer.parseInt(columns[2])
        height = Integer.parseInt(columns[3])
        
        roi = ROIs.createRectangleROI(x, y, width, height, plane)
 
        // Use the first value to determine the PathClass
        value = Float.parseFloat(columns[4])
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

    addObjects(detections)
} catch (IOException ex) {
    Dialogs.showErrorMessage("File open error", ex.getMessage())
}
```
##### 1.2.1 ONLY ONE Detection (Binary-calssification) Loading the .csv file into detections in Qupath
```
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException
import java.lang.Integer
import java.lang.Float

def csvfile = Dialogs.getChooser(null).promptForFile("Load detection CSV file", null, "CSV files", new String[]{".csv"})

if (csvfile == null) {
    Dialogs.showErrorMessage("File Selection", "No file was selected.")
    return
}

int z = 0
int t = 0
def plane = ImagePlane.getPlane(z, t)
def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)
def detections = []

try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
    String DELIMITER = ","
    String line = br.readLine()
    if (line == null) {
        Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
        return
    }

    String[] headers = line.split(DELIMITER)
    if (headers.length < 5 || headers[0] != "x" || headers[1] != "y" || headers[2] != "width" || headers[3] != "height") {
        Dialogs.showErrorMessage("CSV error", "Invalid CSV header. Expected 'x,y,width,height,VALUENAME1,...'")
        return
    }

    int linenum = 1
    while ((line = br.readLine()) != null) {
        linenum++
        String[] columns = line.split(DELIMITER)
        if (columns.length != headers.length) {
            Dialogs.showErrorMessage("CSV file error", "Line " + linenum + " does not match header length.")
            break
        }

        float value = Float.parseFloat(columns[4])
        if (value >= 0.5) {
            int x = Integer.parseInt(columns[0])
            int y = Integer.parseInt(columns[1])
            int width = Integer.parseInt(columns[2])
            int height = Integer.parseInt(columns[3])
            def roi = ROIs.createRectangleROI(x, y, width, height, plane)
            def detection = PathObjects.createDetectionObject(roi, pathclass1)

            for (int i = 4; i < headers.length; i++) {
                value = Float.parseFloat(columns[i])
                detection.getMeasurementList().putMeasurement(headers[i], value)
            }
            detection.getMeasurementList().close()
            detections.add(detection)
        }
    }
} catch (IOException ex) {
    Dialogs.showErrorMessage("File open error", ex.getMessage())
}

if (detections.isEmpty()) {
    Dialogs.showInformationMessage("Import Result", "No 'predictor+' detections found.")
} else {
    addObjects(detections)
}

```
#### 1.3 (Multiple-calssification) Loading the .csv file into detections in Qupath
- tip: based on the 1.2 change color in the line codes *pathClass = PathClassFactory.getPathClass("Category-" + category)*
```
import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException

/* The CSV header line should be as follows:
 * x,y,width,height,VALUENAME1
 * VALUENAME1 is used to determine the class of each detection object.
 */

def csvfile = Dialogs.getChooser(null).promptForFile("Load detection CSV file",
                                                     null, "CSV files", new String[]{".csv"})

int z = 0
int t = 0
def plane = ImagePlane.getPlane(z, t)
def pathClasses = [:]  // A map to store PathClass instances

// Create a reader
try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
    String DELIMITER = ","
    String line
    String[] headers

    def detections = []

    // Read the CSV header line
    if ((line = br.readLine()) != null) {
        headers = line.split(DELIMITER)
        if (headers.length != 5 ||
            !headers[0].equals("x") || !headers[1].equals("y") ||
            !headers[2].equals("width") || !headers[3].equals("height")) {
            Dialogs.showErrorMessage("CSV error",
                "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1'")
            return
        }
    } else {
        Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
        return
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
        String category = columns[4]  // Use VALUENAME1 as the category label

        def roi = ROIs.createRectangleROI(x, y, width, height, plane)

        // Get or create the PathClass based on VALUENAME1
        def pathClass = pathClasses[category]
        if (pathClass == null) {
            pathClass = PathClassFactory.getPathClass("Category-" + category)
            pathClasses[category] = pathClass
        }

        def detection = PathObjects.createDetectionObject(roi, pathClass)
        detections.add(detection)
    }

    // Add all detections to the current QuPath project
    addObjects(detections)
} catch (IOException ex) {
    Dialogs.showErrorMessage("File open error", ex.getMessage())
}
```
### 2. Vision for the predicted probability
#### 2.1 convert the file to .csv file that can be read in Qupath
```
import pandas as pd
import re

# Function to determine the delimiter in a line
def detect_delimiter(line):
    if '\t' in line:
        return '\t'
    else:
        return r',(?![^{]*})'  # Regex for comma outside curly braces

# Function to process a line and extract data
def process_line(line, delimiter):
    columns = re.split(delimiter, line.strip())

    # Ensure there are enough columns
    if len(columns) < 3:
        return None

    first_column, second_colunm, third_column = columns[0], float(columns[1]), int(columns[2])
    if third_column == 0:
        third_column = 1 - second_colunm
    else:
        third_column = second_colunm
    # Extract x, y, width, height from the first column
    match = re.search(r'_x_(\d+)_y_(\d+)_w_(\d+)_h_(\d+).jpg', first_column)
    if match:
        x, y, width, height = match.groups()
        return [x, y, width, height, third_column]
    return None

# Path to the uploaded file
file_path = 'results/inference/predictions.txt'

# Read and process the file
data = []
with open(file_path, 'r') as file:
    # Read the first line to detect the delimiter
    first_line = file.readline()
    delimiter = detect_delimiter(first_line)
    processed_data = process_line(first_line, delimiter)
    if processed_data:
        data.append(processed_data)
    
    # Process the rest of the lines with the detected delimiter
    for line in file:
        processed_data = process_line(line, delimiter)
        if processed_data:
            data.append(processed_data)

# Creating a DataFrame
df = pd.DataFrame(data, columns=['x', 'y', 'width', 'height', 'VALUENAME1'])

# Define the path where the CSV file will be saved
output_csv_file_path = 'results/inference/processed_probability_data.csv'

# Save the DataFrame as a CSV file
df.to_csv(output_csv_file_path, index=False)
```
#### 2.2 Loading the .csv file into detection in Qupath
```
/* Read a CSV file containing detection objects and measurement values for them in the range 0 to 1 */

import qupath.lib.objects.PathObjects
import qupath.lib.roi.ROIs
import qupath.lib.regions.ImagePlane
import qupath.lib.objects.classes.PathClassFactory
import qupath.lib.gui.dialogs.Dialogs
import java.io.BufferedReader
import java.io.FileReader
import java.io.IOException
import java.lang.Integer
import java.lang.Float
import qupath.lib.objects.PathAnnotationObject
import qupath.lib.roi.RectangleROI
import qupath.lib.scripting.QP
import java.awt.Color
import qupath.lib.objects.classes.PathClass

/* The CSV header line should be as follows:
 * x,y,width,height,VALUENAME1,VALUENAME2,...
 * The VALUENAMEs are used as the labels for the detection object measurements.
 * 
 * The first value is used to label each object as a positive detection (>=0.5)
 * or negative detection (<0.5).
 */

/* Get the location of the CSV file */ 
def csvfile = Dialogs.getChooser(null).promptForFile("Load detection CSV file",
                                                     null, "CSV files", new String[]{".csv"})

int z = 0
int t = 0
def plane = ImagePlane.getPlane(z, t)
def pathclass0 = PathClass.fromString("predictor-", 0x008000)
def pathclass1 = PathClassFactory.getPathClass("predictor+", 0x008000)
def roi = ROIs.createRectangleROI(0, 0, 0, 0, plane)
def detection = PathObjects.createDetectionObject(roi, pathclass0)

Color interpolateColor(float value) {
    if (value <= 0.25) {
        // Blue to Green
        return new Color(0, (int)(255 * value * 4), 255);
    } else if (value <= 0.5) {
        // Green to Yellow
        return new Color((int)(255 * (value - 0.25) * 4), 255, 255 - (int)(255 * (value - 0.25) * 4));
    } else if (value <= 0.75) {
        // Yellow to Orange
        return new Color(255, 255 - (int)(255 * (value - 0.5) * 4), 0);
    } else {
        // Orange to Red
        return new Color(255, (int)(255 * (1 - value) * 4), 0);
    }
}


// create a reader
try (BufferedReader br = new BufferedReader(new FileReader(csvfile))) {
    // CSV file delimiter
    String DELIMITER = ","
    String line, valuename
    String[] headers
    int i, x, y, width, height
    float value
    def detections = []

    // read CSV header line
    if ((line = br.readLine()) != null) {
        headers = line.split(DELIMITER)
        if (headers.length < 5 ||
            ! headers[0].equals("x") || ! headers[1].equals("y") ||
            ! headers[2].equals("width") || ! headers[3].equals("height")) {
            Dialogs.showErrorMessage(
                "CSV error", "The CSV header line is invalid; should be 'x,y,width,height,VALUENAME1,...'")
            return
        }
    } else {
        Dialogs.showErrorMessage("CSV error", "The CSV file is empty")
        return
    }

    // Create dummy detection objects to make the measurement limits 0 and 1
    roi = ROIs.createRectangleROI(0, 0, 1, 1, plane)
    detection = PathObjects.createDetectionObject(roi, pathclass0)
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

        x = Integer.parseInt(columns[0])
        y = Integer.parseInt(columns[1])
        width = Integer.parseInt(columns[2])
        height = Integer.parseInt(columns[3])
        
        roi = ROIs.createRectangleROI(x, y, width, height, plane)
 
        // Use the first value to determine the PathClass
        value = Float.parseFloat(columns[4])
        Color color = interpolateColor(value)


        pathclass0 = PathClass.fromString(value.toString(),color.getRGB())       
        detection = PathObjects.createDetectionObject(roi, pathclass0)
 
        for (i = 4; i < headers.length; i++) {
            value = Float.parseFloat(columns[i])
            detection.getMeasurementList().putMeasurement(headers[i], value)
        }
        detection.getMeasurementList().close()
        detections.add(detection)
    }

    addObjects(detections)
} catch (IOException ex) {
    Dialogs.showErrorMessage("File open error", ex.getMessage())
}
```
#### 2.3 Loading the .csv file into annotation in Qupath
```
import qupath.lib.objects.PathAnnotationObject
import qupath.lib.roi.RectangleROI
import qupath.lib.scripting.QP
import java.awt.Color
import qupath.lib.objects.classes.PathClass

// Set the path to your CSV file
String csvPath = "D:/桌面/fsdownload/processed_data.csv"

// Read the CSV file
def file = new File(csvPath)
def lines = file.readLines()

// Skip the header line
lines = lines.drop(1)

// Process each line
lines.each { line ->
    def tokens = line.split(",")
    
    // Extract x, y, width, height, and value
    double x = Double.parseDouble(tokens[0])
    double y = Double.parseDouble(tokens[1])
    double width = Double.parseDouble(tokens[2])
    double height = Double.parseDouble(tokens[3])
    double value = Double.parseDouble(tokens[4])

    // Create a rectangle ROI
    def roi = new RectangleROI(x, y, width, height)

    // Create an annotation
    def annotation = new PathAnnotationObject(roi)

    // Calculate color based on value
    Color color
    if (value <= 0.5) {
        // Transition from blue to green to yellow
        int red = (int)(510 * value)
        int green = (int)(510 * Math.min(value, 0.5))
        int blue = 255 - red
        color = new Color(red, green, blue)
    } else {
        // Transition from yellow to red
        int red = 255
        int green = 255 - (int)(510 * (value - 0.5))
        color = new Color(red, green, 0)
    }

    // Create or get the class with the specified color
    def aClass = PathClass.fromString(value.toString(), color.getRGB())
    annotation.setPathClass(aClass)

    // Add the annotation to the image
    QP.addObjects(annotation)
}

// Fire an update to refresh the display
QP.fireHierarchyUpdate()
```
