1. 图像操作，打入标签
![example](https://github.com/171909771/deep-learning/assets/41554601/606c355d-f754-41ee-adba-bee93176ed12)


2. 编程出tiles
```
import qupath.lib.images.servers.LabeledImageServer

def imageData = getCurrentImageData()

// Define output path (relative to project)
def name = GeneralTools.getNameWithoutExtension(imageData.getServer().getMetadata().getName())
def pathOutput = buildFilePath(PROJECT_BASE_DIR, 'tiles', name)
mkdirs(pathOutput)

// 以下两步得到的downsample是每个pixel对应的实际大小，如果用（double downsample = 1， 表明是下采样，本例为512，double downsample = 1，本例为1024）
// Define output resolution in calibrated units (e.g. µm if available)
double requestedPixelSize = 5.0  // Define export resolution

// Convert output resolution to a downsample factor
double pixelSize = imageData.getServer().getPixelCalibration().getAveragedPixelSize()
double downsample = requestedPixelSize / pixelSize    // Define export resolution

// Create an ImageServer for annotation-derived pixels
def labelServer = new LabeledImageServer.Builder(imageData)
    .backgroundLabel(0, ColorTools.WHITE)
    .downsample(downsample)
    .addLabel('Tumor', 1)
    .multichannelOutput(false)
    .build()

// Create and configure an exporter
new TileExporter(imageData)
    .downsample(downsample)
    .imageExtension('.jpg')
    .tileSize(512)
    .labeledServer(labelServer)
    .annotatedTilesOnly(false)
    .overlap(0)
    .writeTiles(pathOutput)

// Renaming files in the output directory
def dirOutput = new File(pathOutput)
dirOutput.listFiles().each { file ->
    if (file.isFile() && !file.isHidden() && file.getName()) {
        def newName = file.getName().replaceAll("=", "_").replaceAll(" ", "_").replaceAll("\\[", "").replaceAll("\\]", "").replaceAll(",", "_")
        if (file.getName() != newName) {
            def fileUpdated = new File(file.getParent(), newName)
            println("Renaming ${file.getName()} ---> ${fileUpdated.getName()}")
            file.renameTo(fileUpdated)
        }
    }
}

println('Done!')
```
