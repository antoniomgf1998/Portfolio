# Packages


```python
from ipywidgets import HTML
from ipyleaflet import Map, Polyline, Rectangle, basemaps, basemap_to_tiles, Polygon, FullScreenControl, Popup
import pandas as pd
import numpy as np
import math
```

# Functions


```python
def rotate(point, rot):
    
    """Given a 'natural' position of the rectangle printed while representing a ship
    rotate it to the angle it is currently facing
    """
    
    return (point[0]*math.cos(math.radians(rot)) - point[1]*math.sin(math.radians(rot)), 
            point[0]*math.sin(math.radians(rot)) + point[1]*math.cos(math.radians(rot)))

def get_bounds(breadth, length, longitude, latitude, rot_angle):
    """This function gets the real bounds of a boat ==> Realistic plotting
    Knowing that according to: https://en.wikipedia.org/wiki/Decimal_degrees
    Assuming 0.00001deg is equal to 1.1132 m"""
    
    length = length*0.001/1.1132
    breadth = breadth*0.001/1.1132
    
    center_point = np.array((longitude, latitude))
    
    ship_shape_ini = [(-breadth/2, length/2), (breadth/2, length/2), (breadth/2, -length/2), (-breadth/2, -length/2)]
    
    ship_shape_rot = [np.array(rotate(point, rot_angle)) for point in ship_shape_ini]
    
    xy1 = list(ship_shape_rot[0] + center_point)
    xy2 = list(ship_shape_rot[1] + center_point)
    xy3 = list(ship_shape_rot[2] + center_point)
    xy4 = list(ship_shape_rot[3] + center_point)
    
    return [xy1,xy2,xy3,xy4]



def random_hex_color():
    """Get a random color to plot a ship"""
    import random
    r = lambda: random.randint(0,255)
    return ('#%02X%02X%02X' % (r(),r(),r()))
```

# Loading DataSet


```python
data = pd.read_csv('/datc/saab/Australia_ship_data_2019_8.csv')
```

    /opt/jupyterhub/anaconda/lib/python3.6/site-packages/IPython/core/interactiveshell.py:2728: DtypeWarning: Columns (6) have mixed types. Specify dtype option on import or set low_memory=False.
      interactivity=interactivity, compiler=compiler, result=result)


## Visualize the 5 first rows of data


```python
data.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>CRAFT_ID</th>
      <th>LON</th>
      <th>LAT</th>
      <th>COURSE</th>
      <th>SPEED</th>
      <th>TYPE</th>
      <th>SUBTYPE</th>
      <th>LENGTH</th>
      <th>BEAM</th>
      <th>DRAUGHT</th>
      <th>TIMESTAMP</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-2.145699e+09</td>
      <td>83.747228</td>
      <td>-2.149778</td>
      <td>228.9</td>
      <td>13.9</td>
      <td>Cargo ship - Carrying DG, HS, or MP, IMO Hazar...</td>
      <td>NaN</td>
      <td>249</td>
      <td>37</td>
      <td>0.0</td>
      <td>3/08/2019 5:42:31 PM</td>
    </tr>
    <tr>
      <th>1</th>
      <td>-2.145699e+09</td>
      <td>83.464902</td>
      <td>-2.382468</td>
      <td>231.4</td>
      <td>14.0</td>
      <td>Cargo ship - Carrying DG, HS, or MP, IMO Hazar...</td>
      <td>NaN</td>
      <td>249</td>
      <td>37</td>
      <td>0.0</td>
      <td>3/08/2019 7:16:44 PM</td>
    </tr>
    <tr>
      <th>2</th>
      <td>-2.145699e+09</td>
      <td>83.170673</td>
      <td>-2.614127</td>
      <td>232.4</td>
      <td>13.9</td>
      <td>Cargo ship - Carrying DG, HS, or MP, IMO Hazar...</td>
      <td>NaN</td>
      <td>249</td>
      <td>37</td>
      <td>0.0</td>
      <td>3/08/2019 8:51:44 PM</td>
    </tr>
    <tr>
      <th>3</th>
      <td>-2.145699e+09</td>
      <td>82.966347</td>
      <td>-2.770928</td>
      <td>233.2</td>
      <td>13.8</td>
      <td>Cargo ship - Carrying DG, HS, or MP, IMO Hazar...</td>
      <td>NaN</td>
      <td>249</td>
      <td>37</td>
      <td>0.0</td>
      <td>3/08/2019 9:57:32 PM</td>
    </tr>
    <tr>
      <th>4</th>
      <td>-2.145699e+09</td>
      <td>82.656687</td>
      <td>-2.973700</td>
      <td>233.9</td>
      <td>14.1</td>
      <td>Cargo ship - Carrying DG, HS, or MP, IMO Hazar...</td>
      <td>NaN</td>
      <td>249</td>
      <td>37</td>
      <td>0.0</td>
      <td>3/08/2019 11:30:56 PM</td>
    </tr>
  </tbody>
</table>
</div>



# Data visualization (map)

## Creating the map


```python
CRAFT_ID_list = data.CRAFT_ID.unique()#Get the mmsi unique values into a list:

ships_info = []#List that will storage a list of lists == a list of time-series(which will as well be represented as a list)

m = Map(center = (-27.117040, -37.788292), zoom =5)#Define the map object

for rowid in CRAFT_ID_list[:50]:
    #Start with empty lists
    npinfo, infolist = [], []
    #Get a numpy array composed by 'latitude', 'longitude', 'orientation', 'length', 'breadth'
    npinfo = data[data.CRAFT_ID == rowid][['LAT', 'LON', 'COURSE', 'LENGTH', 'BEAM', 'TIMESTAMP']].values
    #Convert it to a python list so it can be an attribute of the multypoligon functionality of ipyleaflet
    infolist = coordslist = list([list(coords) for coords in npinfo])
    ships_info.append(infolist)

    
    
for i in range(len(ships_info)):#for each time series in ships_info list
    color_value = random_hex_color()
    line = Polyline(
        locations = [[elem[0],elem[1]] for elem in ships_info[i]],
        color = color_value,
        fill_color= "transparent",
        weight = 3,
        opacity = 0.8)
    m.add_layer(line)
    for singular_info in [ships_info[i][e] for e in range(len(ships_info[i])) if e%5==0]:
        
        message = HTML()
        message.value = '<b>**DATE: <br>' + str(singular_info[-1]) + '**</b><br><b>Longitude</b>: (' + str(singular_info[0]) + '<br><b>Latitude</b>:  ' + str(singular_info[1]) + "<br><b>Vessel's Length</b>: " + str(singular_info[3]) + "<br><b>Vessel's Breadth</b>: " + str(singular_info[4])
        
        
        mapbounds = get_bounds(singular_info[3], singular_info[4], singular_info[0], singular_info[1], singular_info[2])
        ship_shape = Polygon(
        locations=[mapbounds],
        color=color_value,
        fill_color=color_value,
        fill_opacity=1
        )
        ship_shape.popup = message
        #ship_shade = Rectangle(bounds=(mapbounds[0], mapbounds[1]), fill_opacity=1, fill_color = color_value, color = color_value)
        m.add_layer(ship_shape)
```


```python
m.add_control(FullScreenControl())
m
```


    Map(basemap={'url': 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', 'max_zoom': 19, 'attribution': 'Map …


# Save the map


```python
from ipywidgets.embed import embed_minimal_html
embed_minimal_html('australia_50_random_color_visualization.html', views=[m])
```


```python

```
