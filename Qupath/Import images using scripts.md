# Importing images
## reference
- https://forum.image.sc/t/qupath-script-for-adding-image-and-renaming-image/92357/5
### 1. importing single file
```
import qupath.lib.images.servers.ImageServers
import java.nio.file.Paths

def imagePath = "/home/chan87/Desktop/WSI/metastasis/metastasis/2400529/2400529_5.svs"
def project = getProject()
if (project == null) {
    println "A project must be opened before running this script"
    return
}

// Extract the filename from the imagePath to use as the image name
def imageName = Paths.get(imagePath).getFileName().toString()
def imageServer = ImageServers.buildServer(imagePath)
def imageEntry = project.addImage(imageServer.getBuilder())
imageEntry.setImageName(imageName)

project.syncChanges()
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
        // Extract the filename to use as the image name
        def imageName = path.getFileName().toString()
        
        // Build server for each .svs file
        def imageServer = ImageServers.buildServer(path.toString())
        if (imageServer != null) {
            // Create an image entry and set the image name
            def imageEntry = project.addImage(imageServer.getBuilder())
            imageEntry.setImageName(imageName)  // Set the image name to the original filename
            
            // Save changes for each entry added
            project.save()
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
                // Extract the filename to use as the image name
                def imageName = path.getFileName().toString()
                
                // Build server for each .svs file
                def imageServer = ImageServers.buildServer(path.toString())
                if (imageServer != null) {
                    // Create an image entry and set the image name
                    def imageEntry = project.addImage(imageServer.getBuilder())
                    imageEntry.setImageName(imageName)  // Set the image name to the original filename
                    
                    // Save changes for each entry added
                    project.save()
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

### 3. Randomly importing one files in all subdirectories in a specific directory
```
import qupath.lib.images.servers.ImageServers
import java.nio.file.Files
import java.nio.file.Paths
import java.nio.file.Path
import java.io.IOException
import java.util.stream.Collectors
import java.util.Random

def dirPath = "/home/chan87/Desktop/WSI/metastasis"

def project = getProject()
if (project == null) {
    println "A project must be opened before running this script"
    return
}

Random random = new Random()

try {
    // Create a map to store subdirectories and their respective files
    Map<String, List<Path>> directoryFiles = new HashMap<>()

    // Walk through all files in the directory and subdirectories
    Files.walk(Paths.get(dirPath)).forEach { path ->
        if (path.toString().toLowerCase().endsWith(".svs")) {
            String parentDir = path.getParent().toString()
            if (!directoryFiles.containsKey(parentDir)) {
                directoryFiles.put(parentDir, new ArrayList<>())
            }
            directoryFiles.get(parentDir).add(path)
        }
    }

    // Process one random file from each subdirectory
    directoryFiles.each { dir, paths ->
        if (!paths.isEmpty()) {
            Path path = paths.get(random.nextInt(paths.size()))
            try {
                def imageName = path.getFileName().toString()
                def imageServer = ImageServers.buildServer(path.toString())
                if (imageServer != null) {
                    def imageEntry = project.addImage(imageServer.getBuilder())
                    imageEntry.setImageName(imageName)
                    project.save()
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

println "One random .svs file from each subdirectory has been added."

```
