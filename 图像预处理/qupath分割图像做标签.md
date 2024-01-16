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

// Define output resolution
double requestedPixelSize = 1  // Maintain original resolution

// Convert to downsample
double downsample = requestedPixelSize / imageData.getServer().getPixelCalibration().getAveragedPixelSize()

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
    .imageExtension('.png')
    .tileSize(300)
    .labeledServer(labelServer)
    .annotatedTilesOnly(false)
    .overlap(64)
    .writeTiles(pathOutput)

// Renaming files in the output directory
def dirOutput = new File(pathOutput)
dirOutput.listFiles().each { file ->
    if (file.isFile() && !file.isHidden() && file.getName().endsWith('.png')) {
        def newName = file.getName().replaceAll("=", "-").replaceAll("\\[", "").replaceAll("\\]", "").replaceAll(",", "_")
        if (file.getName() != newName) {
            def fileUpdated = new File(file.getParent(), newName)
            println("Renaming ${file.getName()} ---> ${fileUpdated.getName()}")
            file.renameTo(fileUpdated)
        }
    }
}

println('Done!')
```
