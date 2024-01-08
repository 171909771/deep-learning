
## Step1. 预处理推断结果
### 1. 合并model/viz中的train和val文件
##### 如果是what批量inference出的结果，那么就跳过这一步。
```
# Function to read file content
def read_file(file_path):
    with open(file_path, 'r') as file:
        return file.readlines()

# Paths to the training and validation data files
file_path_train = '/home/chan87/jupyter_home/病理组学/resnet18/viz/BST_TRAIN_RESULTS.txt'  # 路径，可能需要修改
file_path_val = '/home/chan87/jupyter_home/病理组学/resnet18/viz/BST_VAL_RESULTS.txt'  # 路径，可能需要修改

# Reading the data from both files
train_data = read_file(file_path_train)
val_data = read_file(file_path_val)

# Merging the training and validation data
merged_data = train_data + val_data

# Writing the merged data to a new file
merged_file_path = '/home/chan87/jupyter_home/merged_results.txt'  # 路径，可能需要修改
with open(merged_file_path, 'w') as file:
    file.writelines(merged_data)

```

### 2. 提取Qupath中需要的数据，并且整理为其能够读取的格式
![image](https://github.com/171909771/deep-learning/assets/41554601/b0d7ca2e-9c13-49c2-98e8-6f328ebd5727)
#### 2.1 以下是提取预测值的表格
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

    first_column , third_column = columns[0], columns[2]
    # Extract x, y, width, height from the first column
    match = re.search(r'_x_(\d+)_y_(\d+)_w_(\d+)_h_(\d+).jpg', first_column)
    if match:
        x, y, width, height = match.groups()
        return [x, y, width, height, third_column]
    return None

# Path to the uploaded file
file_path = '/home/chan87/jupyter_home/predict.csv'

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
output_csv_file_path = '/home/chan87/jupyter_home/processed_data.csv'

# Save the DataFrame as a CSV file
df.to_csv(output_csv_file_path, index=False)
```

#### 2.2 以下是提取预测概率的表格
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
file_path = '/home/chan87/jupyter_home/merged_results.txt'

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
output_csv_file_path = '/home/chan87/jupyter_home/processed_data.csv'

# Save the DataFrame as a CSV file
df.to_csv(output_csv_file_path, index=False)
```



## Step2. Qupath中运行
### 1. 分类热图
### 选取之前得到的newresults.csv
- https://gist.github.com/juliangilbey/49a84e0b0cfe5aeffbd9e1128cfa8d07#file-drawpredictionobjects-groovy
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

### 3.1 概率热图 (修改上面的代码)
### 选取之前得到的probability.csv
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


### 3.2 概率热图 (生成annotation object)，耗时
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
