# Background needed functions for visualization

### Rotate a point


```python
import math
```


```python
"""
    - given a point and an angle, rotate that point
"""
def rotate(point, rot):
    return (point[0]*math.cos(math.radians(rot)) - point[1]*math.sin(math.radians(rot)), 
            point[0]*math.sin(math.radians(rot)) + point[1]*math.cos(math.radians(rot)))
```

### Example


```python
print(rotate([0,1], 90))
#NOTE THAT 1e-17 is almost 0
```

# Coloring Functions


```python
"""
For further description, see /saab/4. Common Functions/Color functions.ipynb
"""
def get_color_scale(arr, scale_type='plasma'):
    arr = list(arr)
    #https://matplotlib.org/gallery/color/colormap_reference.html
    scale = cm.get_cmap(scale_type, len(arr)).colors
    
    arr = np.array(arr)
    sorted_index = np.argsort(arr)
    return {arr[sorted_index[i]]:rgb_to_hex(scale[i]) for i in range(len(arr))}

def rgb_to_hex(rgb):
    if len(rgb) == 4:
        rgb = rgb[:3]
    return '#%02x%02x%02x' % tuple(list([int(elem*256) for elem in rgb]))
def random_hex_color():
    import random
    r = lambda: random.randint(0,255)
    return ('#%02X%02X%02X' % (r(),r(),r()))
```

# Visualization


```python
from ipywidgets import HTML
import ipywidgets as widgets
from ipyleaflet import Map, Polyline, Rectangle, basemaps, basemap_to_tiles, Polygon, FullScreenControl, Popup, WidgetControl
```


```python
#Define the map object
m = Map(center = (-25.353548853000003, -43.935133436), zoom =4)

"""DEFINE THE SLIDER"""
max_steps = max([len(element) for element in ships_info])
ships_slider = widgets.IntSlider(
    value=0,
    min=0,
    max=200,
    step=1,
    description='Ships: ',
    disabled=False,
    continuous_update=False,
    orientation='horizontal',
    readout=True,
    readout_format='d'
)
widget_steps = WidgetControl(widget=ships_slider, position='topright')

"""ADD SLIDER TO MAP"""
m.add_control(widget_steps)
"""ADD A FULL SCREEN CONTROL TO THE MAP"""
m.add_control(FullScreenControl())
"""ADD A DARK LAYER TO MAP (OFC OPTIONAL)"""
dark_matter_layer = basemap_to_tiles(basemaps.CartoDB.DarkMatter)
m.add_layer(dark_matter_layer)

"""PREVIOUS VALUE OF THE SLIDER"""
previous_value = 0


"""FUNCTION THAT WILL GET EXECUTED WHENEVER WE MOVE THE SLIDER"""
def update_map(ships_slider):
    #Global variables (shared between the function and the main code)
    global previous_value, m
    
    #If previous value is higher than the actual one (DOESN'T WORK):
    if previous_value > ships_slider:
        m = Map(center = (-22.884059, 133.714373), zoom =4)#Define the map object
        ini, end = 0, ships_slider
    else:
        ini, end = previous_value, ships_slider
        
        
    for i in range(ini, end):
        color_value = random_hex_color()
        #for each time series in ships_info list --> Paint The line
        line = Polyline(
            locations = [[elem[0],elem[1]] for elem in ships_info[i][29]],
            color = color_value,
            fill_color= "transparent",
            weight = 3,
            opacity = 0.8)

        m.add_layer(line)
        """If ship points != 0"""
        if ships_info[i][29] != []:
            """Add pop Up information"""
            singular_info = ships_info[i][29][-1]
            message = HTML()
            message.value = '<b>**DATE: <br>' + str(singular_info[-1]) + '**</b><br><b>Longitude</b>: ' + str(singular_info[0]) + '<br><b>Latitude</b>:  ' + str(singular_info[1]) + "<br><b>Vessel's Length</b>: " + str(singular_info[3]) + "<br><b>Vessel's Breadth</b>: " + str(singular_info[4]) + "<br><b>Facing:</b>: " + str(singular_info[2]) + ' deg'

            mapbounds = get_bounds(singular_info[3], singular_info[4], singular_info[0], singular_info[1], singular_info[2])
            ship_shape = Polygon(
            locations=[mapbounds],
            color=color_value,
            fill_color=color_value,
            fill_opacity=1
            )
            ship_shape.popup = message
            m.add_layer(ship_shape)
    previous_value = ships_slider
display(m)
widgets.interactive(update_map, ships_slider=ships_slider)
```


```python

```
