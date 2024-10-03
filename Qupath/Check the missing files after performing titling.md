## Check the missing files after performing titling
```
import os

# Paths
source_path = "/home/chan87/Desktop/WSI/no_meta"
target_path = "/home/chan87/Desktop/tmp2/tiles"

# Get all file names from the source path's subdirectories (excluding extensions)
file_names = set()
for root, dirs, files in os.walk(source_path):
    for file in files:
        file_name_without_extension = os.path.splitext(file)[0]
        file_names.add(file_name_without_extension)

# Get all directory names from the target path
target_dirs = set(os.listdir(target_path))

# Find which file names do not have corresponding directories
missing_dirs = file_names - target_dirs

# Output the missing directories (actually these are the file names missing as dirs in target)
print("Missing directories based on file names:", missing_dirs)

```
