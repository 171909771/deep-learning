## Identify if the ROIs are considered as the targeted area. "ture" represents yes
.annotatedTilesOnly(true)

## The resolution of the output picture depends on the method you choose
- only choose one method Method.1 or Method.2

```
/**
 * Script to export image tiles (can be customized in various ways).
 */
extension = '.jpg'
// Get the current image (supports 'Run for project')
def imageData = getCurrentImageData()

// Define output path (here, relative to project)
def name = 
GeneralTools.getNameWithoutExtension(imageData.getServer().getMetadata().getName())
def pathOutput = buildFilePath(PROJECT_BASE_DIR, 'tiles', name)
mkdirs(pathOutput)

// Method.1. To export at full resolution
// 1 represents full resolution, the number is bigger the resolution is lower
double downsample = 1              
// Method.1. To export at full resolution

// Method.2. To export at a fix resolution
// Define output resolution in calibrated units (e.g. µm if available)
double requestedPixelSize = 1  // Define export resolution; find the specific resolution value in the image item in Qupath
double pixelSize = imageData.getServer().getPixelCalibration().getAveragedPixelSize()
double downsample = requestedPixelSize / pixelSize    // Define export resolution  这里可以统一所有图像的分辨率, step 2
// Method.2. To export at a fix resolution

// Create an exporter that requests corresponding tiles from the original & labeled image servers
// If there are fluorescence images with multiple channels, and wanna choose a specific channel, could add ".channels(int)" parameter in the TileExporter functions
// and the output picture type should be .tif
new TileExporter(imageData)
    .downsample(downsample)   // Define export resolution
    .imageExtension(extension)   // Define file extension for original pixels (often .tif, .jpg, '.png' or '.ome.tif')
    .tileSize(1024)            // Define size of each tile, in pixels
    .annotatedTilesOnly(true) // If true, only export tiles if there is a (classified or unclassified) annotation present
    .overlap(0)              // Define overlap, in pixel units at the export resolution
    .writeTiles(pathOutput)   // Write tiles to the specified directory

def dirOutput = new File(pathOutput)
for (def file in dirOutput.listFiles()) {
    if (!file.isFile() || file.isHidden() || !file.getName().endsWith(extension))
        continue
    def newName = file.getName().replaceAll("=","-").replaceAll("\\[","").replaceAll("\\]","").replaceAll(",","_").replaceAll(" ","_")
    if (file.getName() == newName)
        continue
    def fileUpdated = new File(file.getParent(), newName)
    println("Renaming ${file.getName()} ---> ${fileUpdated.getName()}")
    file.renameTo(fileUpdated)
}

println('Done!')
```
