# Importing images
## reference
- https://forum.image.sc/t/qupath-script-for-adding-image-and-renaming-image/92357/5
### 1. importing single file
```
import qupath.lib.images.servers.ImageServers
def imagePath = "/home/chan87/Desktop/WSI/metastasis/metastasis/2400529/2400529_5.svs"
def project = getProject()
if (project == null) {
    println "A project must be opened before running this script"
    return
}
def imageServer = ImageServers.buildServer(imagePath)
def imageEntry = project.addImage(imageServer.getBuilder())
getQuPath().refreshProject()
```

### 2. importing all files in single directory
```
import qupath.lib.images.servers.ImageServers
import java.nio.file.Files
import java.nio.file.Paths

def dirPath = "/home/chan87/Desktop/WSI/metastasis/metastasis/2400529"
def project = getProject()
if (project == null) {
    println "A project must be opened before running this script"
    return
}

// List all files in the directory
Files.newDirectoryStream(Paths.get(dirPath), "*.svs").each { path ->
    try {
        // Build server for each .svs file
        def imageServer = ImageServers.buildServer(path.toString())
        if (imageServer != null) {
            // Add the image to the project
            project.addImage(imageServer.getBuilder())
        } else {
            println "Failed to build server for: ${path}"
        }
    } catch (Exception e) {
        println "Error processing file ${path}: ${e.getMessage()}"
    }
}

// Refresh the project view to show new images
getQuPath().refreshProject()

println "All .svs files from the directory have been added."

```

### 3. importing all files in all subdirectories in a specific directory
```
import qupath.lib.images.servers.ImageServers
import java.nio.file.Files
import java.nio.file.Paths
import java.io.IOException

def dirPath = "/home/chan87/Desktop/WSI/metastasis/metastasis"

def project = getProject()
if (project == null) {
    println "A project must be opened before running this script"
    return
}

try {
    // Walk through all files in the directory and subdirectories
    Files.walk(Paths.get(dirPath)).forEach { path ->
        if (path.toString().toLowerCase().endsWith(".svs")) {
            try {
                // Build server for each .svs file
                def imageServer = ImageServers.buildServer(path.toString())
                if (imageServer != null) {
                    // Add the image to the project
                    project.addImage(imageServer.getBuilder())
                } else {
                    println "Failed to build server for: ${path}"
                }
            } catch (Exception e) {
                println "Error processing file ${path}: ${e.getMessage()}"
            }
        }
    }
} catch (IOException e) {
    println "Error walking through the directory: ${e.getMessage()}"
}

// Refresh the project view to show new images
getQuPath().refreshProject()

println "All .svs files from the directory and its subdirectories have been added."

```
