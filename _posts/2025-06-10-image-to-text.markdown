---
layout: post
title:  "How to Make Images Searchable by Converting Them to Text"
date:   2025-06-10 06:15:27 +0400
categories: tutorial
---

When we upload many images, it's hard to search for the one we want. Unlike text files, images are not easy to index or search by content. So, the best way is to convert important details from image into text format. Once it's in text, we can store and search it like we do for normal data.

![image to text]({{ site.baseurl }}/assets/image-to-text.png)

To do this, we need to detect what is inside the image. For example, what objects are present, what colors are visible, and where the photo was taken.

Let’s see how to do this step by step using simple tools.

---

### 1. Detect Objects Using YOLO

YOLO (You Only Look Once) is a popular tool for detecting objects in images. It is a deep learning model. It looks at the image and tells us what objects are there like "person", "car", "dog", etc.

Here is a simple code to run YOLO:

```python
from ultralytics import YOLO
import cv2

def get_image_objects():
  model = YOLO("yolo11n.pt")
  img = cv2.imread(image_path)
  img_rgb = cv2.cvtColor(img_resized, cv2.COLOR_BGR2RGB)
  results = model(img_rgb)
  object_names = set()
  for r in results:
      for c in r.boxes.cls:
          object_names.add(model.names[int(c)])
  return list(object_names)
```

This gives us names of objects found in the image. We can save them as keywords.

---

### 2. Get Location from EXIF Data

Many photos taken from phone or camera have location info saved in EXIF data. We can read this data using `PIL` or `exifread`.

```python
from PIL import Image
from PIL.ExifTags import TAGS, GPSTAGS

def get_exif_data(self, image_path:str):
  """Extract EXIF data from an image."""
  try:
      image = Image.open(image_path)
      exif_data = image._getexif()
      if exif_data is None:
          return None
      exif = {
          TAGS.get(tag, tag): value
          for tag, value in exif_data.items()
          if tag in TAGS
      }
      return exif
  except Exception as e:
      print(f"Error extracting EXIF data: {e}")
      return None

def get_gps_info(self, exif_data):
  """Extract GPS information from EXIF data."""
  if not exif_data:
      return None
  gps_info = {}
  for key, value in exif_data.items():
      if key == "GPSInfo":
          gps_info = {
              GPSTAGS.get(tag, tag): value[tag]
              for tag in value
              if tag in GPSTAGS
          }
  return gps_info

```

This gives GPS coordinates like latitude and longitude.

---

### 3. Convert Coordinates to City Name

Once we get GPS coordinates, we can find which city it belongs to. For this, we need a database of cities with their GPS coordinates. Then we can use distance formula to find the closest city.

```python
import sqlite3
import math

def gps_to_city(self,location):
    db_path = f"/app/data/geo_names.db"
    query = """
        SELECT
            c.id,
            c.name,
            c.label,
            c.latitude,
            c.longitude,
            SQRT(POWER((c.latitude - ?), 2) + POWER((c.longitude -  ?), 2)) AS distance
        FROM
            cities c
        ORDER BY
            distance ASC
        LIMIT 1;
    """
    conn = None

    conn = sqlite3.connect(db_path)
    conn.create_function("SQRT", 1, math.sqrt)
    conn.create_function("POWER", 2, lambda x, y: math.pow(x, y))
    # Connect to the SQLite database
    conn.row_factory = sqlite3.Row  # This enables fetching rows as dictionaries
    cursor = conn.cursor()
    cursor.execute(query, location)

    # Fetch one row as a dictionary
    row = cursor.fetchone()
    # Close the connection
    conn.close()
    if row:
      return dict(row)
    return None
```

You can get a free database of city names with coordinates from [here](https://public.opendatasoft.com/explore/dataset/geonames-all-cities-with-a-population-1000/table/?disjunctive.cou_name_en&sort=name)

---

### 4. Detect Major Colors in Image

To find out main colors in photo, we can use a Python library called `colorthief`. It picks most dominant colors.

```python
from colorthief import ColorThief

color_thief = ColorThief('image.jpg')
main_color = color_thief.get_color(quality=1)
palette = color_thief.get_palette(color_count=5)
```

This gives us top 5 colors in RGB format.

---

### 5. Get Color Names with WebColors

RGB values are just numbers. We want to convert them into color names like “red”, “blue”, “dark green”, etc. For this, we use `webcolors`.

```python
import webcolors

def closest_color_name(self, requested_rgb):
  min_colors = {}
  names = webcolors.names(spec=webcolors.CSS21)

  for name in names:
      r_c, g_c, b_c = webcolors.name_to_rgb(name)
      rd = (r_c - requested_rgb[0]) ** 2
      gd = (g_c - requested_rgb[1]) ** 2
      bd = (b_c - requested_rgb[2]) ** 2
      min_colors[(rd + gd + bd)] = name
  return min_colors[min(min_colors.keys())]
```

This helps us add meaningful text tags for image colors.

---


### Result


![image to text]({{ site.baseurl }}/assets/location-image.jpg)



"Its an image type file. Image was taken at location near to Abu Dhabi, United Arab Emirates. Image was taken noon on Friday May 2025. Some of visible objects in the image are bowl, dining table, vase, chair. Most dominant color in image is gray. Some other visible colors in image are black, maroon, olive"

---

### Summary

Now, after following all these steps, we get useful text data from an image like:

* Detected objects
* Location (city name)
* Major colors and their names

We can save this as metadata. Later, we can search images using this text. For example: “Show me all beach photos with blue sky from Dubai.”

This is the simple way to make images searchable using Python.

