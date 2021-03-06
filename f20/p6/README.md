# Don't Start Yet!!!  Still under revision.

# P5: Wisconsin Land Use

In this project, you'll explore how land is used in Wisconsin (for
cities, forests, farming, etc).  [This
dataset](https://www.mrlc.gov/data/nlcd-land-cover-conus-all-years)
breaks the United States into 30m squares and categorizes how those
chunks of land have been used from 2001 to 2016 (this data is
collected every 2-3 years, so there only 7 years of data over this
range).

Take a look
[here](https://www.mrlc.gov/data/legends/national-land-cover-database-2016-nlcd2016-legend)
to see the different ways land is used.

Although the data covers the whole US, we'll be looking at a sampling
of places in WI.  Here's an example of the land use around Madison:

<img src="good-colors.png" width=400>

You'll write a Python module to help others analyze this data (like
the bus module for P2).  The module will help pull data from a sqlite3
database, a zip file, and numpy matrices representing land use.

## Corrections/Clarifications
* Apr 9: tester fix: image_load, image_name and image_year tests (Affects functionality, please redownload)
* Apr 7: slight tester fix (does not affect functionality)
* Apr 5: fixed some issues w/ tester.py
* Apr 4: updated tester.py with tests for remaining parts
* Apr 2: forgot to push the tests already in tester.py yesterday

## Dataset

The [original dataset](https://s3-us-west-2.amazonaws.com/mrlc/NLCD_Land_Cover_L48_20190424_full_zip.zip) for the whole US is large (roughly 10GB, compressed).

Thus, we only take data for the following points in WI:

<img src="wi.png" width=400>

The 100 gray dots are randomly chosen, and the black dots are the 10
largest cities.  There are seven years of data for each of the city
points, but only 2016 data for the randomly sampled points.

We're giving you two files with data about these locations:
`images.zip` and `images.db`.  Both are generated by
build-dataset.ipynb (you're welcome to look at how it works if you're
curious, but it's not really related to what you'll do for P5, and
uses several things we haven't talked about in CS 320).

### Zipped Images

Look inside `images.zip`:

```
unzip -l images.zip
```

You'll see a bunch of .npy files, meaning they contain numpy matrices
(these contain the use data).

#### [Read This](map.md) to learn how to generate a map from the matrices.

How do you know where those images are for?  That's where `images.db`
comes in.

### Sqlite Database

Run the following:

```python
import os
import sqlite3
import pandas as pd

assert os.path.exists("images.db")
c = sqlite3.connect("images.db")

pd.read_sql("SELECT tbl_name FROM sqlite_master", c)
```

Run some SQL queries to see what is in each of the tables.

Note, you'll often want to combine (or *join*) data from the two
tables into one big "table", so you can see both time (from `images`
table) and location (from `places` table).  More on this in lab...

## Requirements

You'll need to write a class and some methods in your module, which
should be in a file named `land.py`.

### 1. `Connection` class

Create a `Connection` class in land.py, building on the following:

```python
import zipfile
import sqlite3

def open(name):
    pass # TODO

class Connection:
    def __init__(self, name):
        self.db = sqlite3.connect(name+".db")
        self.zf = zipfile.ZipFile(name+".zip")

    def close(self):
	# TODO
```

People should be able to use your module like this:

```python3
import land

c = land.open("images")
# use connection
c.close()
```

Or this:

```python3
with land.open("images") as c:
    pass # use connection
```

Make sure `db` and `zf` get closed at the end.

`Connection` should have `list_images`, `image_year`, `image_name`,
and `image_load` methods that behave as follows:

```python
with land.open("images") as c:
    # gets alphabetically sorted list of images
    # expected: ['area0.npy', 'area1.npy', 'area10.npy', 'area100.npy', ...]
    print(c.list_images()) 

    # get name from DB corresponding to this image
    # expected: 2001 (of type int, not int64)
    print(c.image_year("area0.npy"))

    # get name from DB corresponding to this image
    # should be "madison"
    print(c.image_name("area0.npy")) 
    
    # get numpy area that encodes area usage
    # should be a 2-dimensional numpy array
    print(c.image_load("area0.npy"))
```

For these functions, you may either do a query each time, or load the
data to a pandas DataFrame when a `Connection` is created.  It's up to
you!

The last three functions take an image name and return some
corresponding data.  "Year" comes from the `images` table in the DB.
"Name" is trickier: the `images` table associates the file name with a
`place_id`, which can then be used in the `places` table to get a
name.

### 2. `Connection.lat_regression` method

Use least-squares to get `slope` and `intercept` values for an
equation like this:

`percent = slope*lat + intercept`

`percent` is the portion (between 0 and 100) that is expected to have
usage code `use_code` at a given latitude.

If `c` is a `Connection` then `c.lat_regression(use_code, ax)` should
return this information as a `(slope, intercept)` tuple.

For example, `c.lat_regression(41, None)` returns `(5.965578948401942,
-243.38363729644226)`.

This means that for every degree of latitude you go north, you can
roughly expect another 6% to be Deciduous Forest (the land type for
code 41).  Or, based on this information, we might note that Madison's
latitude is 43.0731, so we might expect about 13.3% (`5.96 * 43.07 +
-243.38`) of the nearby areas to be Deciduous Forest.

Install scikit-learn (`pip3 install sklearn`) and use
`LinearRegression` to do the regression:

https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LinearRegression.html

If the `ax` value passed is not None, `lat_regression` should plot the
computed points and fit line in that area.  For example, somebody
might use your function like this:

<img src="lat-reg.png" width=500>

**Important**: When using `scatter()` in this project, you need to call it like 
`ax.scatter()` so the tester picks up the points. Otherwise tester.py may not 
recognize the points. 

**Important**: only include points with names that begin with "samp",
  like "samp5" (but not with `name="madison"`) for this function.
  We'll use the city data for the next part.

### 3. `Connection.year_regression` method

This is similar to `lat_regression`, but with the following differences:
1. we'll only use data points with a name, that is passed in as the first argument
2. the x-axis will be year instead of latitude
3. it should be possible to pass a list of codes to list_code

For example, you should be able to run this:

```python
import matplotlib.pyplot as plt
fig, ax = plt.subplots()
ax.set_ylabel("Developed")
ax.set_xlabel("Year")
c.year_regression("madison", [21,22,23,24], ax)
```

And get this:

<img src="developed.png" width=500>

Here, we're grabbing the 7 snapshots of Madison, and counting what
percent of the are is coded between 21 ("Developed, Open Space") and
24 ("Developed, High Intensity").

Once you have that working, explore other trends by making more calls.
For example, what is happening to farming in the Madison area (codes
81 and 82)?  How do the development trends in Madison compare to those
in Milwaukee?

These are the city names in the dataset: madison, milwaukee, greenbay,
kenosha, racine, appleton, waukesha, oshkosh, eauclaire, janesville.
Play around with different trends for various cities and see if you
find any surprising developments.

### 4. `Connection.animate` method

This one should take a city name, then use FuncAnimation to produce
some HTML that can be shown in a Jupyter cell, like this:

```python
from IPython.core.display import HTML
html = c.animate("eauclaire")
HTML(html)
```

It should say the year somewhere on the animation.  It's OK to have
one frame per year of data (this is slightly misleading, because the
time elapsed between frames will sometimes be 2 year and sometimes 3,
but we want to keep it simple).

You can download an example of the video at
[eauclaire.mp4](eauclaire.mp4).  A screenshot of it looks like this:

<img src="eauclaire.png" width=500>
