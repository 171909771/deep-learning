
```
import json

def remove_any_plane_information(feature):
    """
    Recursively removes any 'plane' key from the feature, regardless of its content.
    """
    if isinstance(feature, dict):
        # If the feature is a dictionary and contains 'plane', remove it
        if "plane" in feature:
            del feature["plane"]
        else:
            # Otherwise, recursively check and remove 'plane' from its values
            for key in list(feature.keys()):
                remove_any_plane_information(feature[key])
    elif isinstance(feature, list):
        # If the feature is a list, recursively check each item
        for item in feature:
            remove_any_plane_information(item)

# Path to the GeoJSON file
file_path = '/home/chan87/jupyter_home/5.geojson'

# Load the GeoJSON content from the file
with open(file_path, 'r') as file:
    geojson_data = json.load(file)

# Apply the removal function to each feature in the GeoJSON data
for feature in geojson_data['features']:
    remove_any_plane_information(feature)

# Save the modified content to a new file after removing 'plane' information
modified_file_path = '/home/chan87/jupyter_home/modified_geojson_file.geojson'
with open(modified_file_path, 'w') as file:
    json.dump(geojson_data, file, indent=4)

```
