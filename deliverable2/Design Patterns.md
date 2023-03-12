# **Design Patterns**

## **Pattern 1: Composite Pattern**
In this [code snippet](https://github.com/matplotlib/matplotlib/blob/d6dd1b79142e4e2483244c25ff1e3ddb664722d3/lib/matplotlib/figure.py#L469-L502), class FigureBase, which extends Artist, provides a method add_artist(artist, clip=False) to add the artist passed into the list of Artists it holds.
```python
def add_artist(self, artist, clip=False):
	...
	artist.set_figure(self)
	self.artists.append(artist)
	artist._remove_method = self.artists.remove

	if not artist.is_transform_set():
		artist.set_transform(self.transSubfigure)

	if clip:
		artist.set_clip_path(self.patch)

	self.stale = True
	return artist
```

This [line](https://github.com/matplotlib/matplotlib/blob/d6dd1b79142e4e2483244c25ff1e3ddb664722d3/lib/matplotlib/figure.py#L180) is the initialization of the list of Artists in FigureBase.
```python
def __init__(self, **kwargs):
		...
		self.artists = []
		...
```

The following methods draw all the Artists in the list.
- [draw(renderer)](https://github.com/matplotlib/matplotlib/blob/d6dd1b79142e4e2483244c25ff1e3ddb664722d3/lib/matplotlib/figure.py#L2306-L2325)
- [_draw_list_compositing_images(renderer, parent, artists, suppress_composite=None)](https://github.com/matplotlib/matplotlib/blob/6a9a07155c0e7f91c20dd4c7e280198ec652c4ae/lib/matplotlib/image.py#L113-L157)
```python
def draw(self, renderer):
	...
	mimage._draw_list_compositing_images(
		renderer, self, artists, self.figure.suppressComposite)
	...

def _draw_list_compositing_images(
	renderer, parent, artists, suppress_composite=None):
	...

	if not_composite or not has_images:
		for a in artists:
			a.draw(renderer)
	...
```
<br>

### UML Diagram:
![screensh](./diagrams/Composite%20Design%20Pattern.png)

FigureBase class here acts as the Composite class, which extends the Component class, Artist, and owns a list of Artists. The Leaf classes are the other components extending Artist, for example in the UML diagram, Axes is a Leaf class. Because a FigureBase instance can have 0..* Artist components, FigureBase and those components form a tree structure. As they all have method draw(renderer), the composition, FigureBase, and individual components, like Axes, can be treated uniformly. For users who want to draw a Figure with Matplotlib, the users can use the same draw(renderer) methods like drawing individual components, and the design pattern would traverse the tree structure inside the composite Figure to draw each component for you. Therefore, for a client, how to draw a composite Figure and how to draw individual Axes have no difference.


<br>
<br>
<br>

## **Pattern 2: Observer Pattern**

### Motivation for Observer Pattern
Matplotlib has a lot of interactive features from panning and zooming, to button presses and mouse hovering. To reduce redundancy, the Observer pattern is followed to allow any Artist element to be interactive (if they want to be).

<br>

All of the common event names are defined in FigureCanvasBase in [backend_bases.py line 1667](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/backend_bases.py#L1667-L1682). 
```python
events = [
	'resize_event',
	'draw_event',
	'key_press_event',
	'key_release_event',
	'button_press_event',
	'button_release_event',
	'scroll_event',
	'motion_notify_event',
	'pick_event',
	'figure_enter_event',
	'figure_leave_event',
	'axes_enter_event',
	'axes_leave_event',
	'close_event'
]
```

FigureCanvasBase acts as the canvas for which Figure objects can be rendered. It then follows that FigureCanvasBase has a reference to a Figure object it is supposed to render ([backend_bases.py line 1699](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/backend_bases.py#L1699-L1702)) and the Figure object has a reference to the canvas it is being rendered to ([figure.py line 2881](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/figure.py#L2881-L2889)). 
```python
# backend_bases.py line 1699
def __init__(self, figure=None):
	# ...
	if figure is None:
		figure = Figure()
	figure.set_canvas(self)
	self.figure = figure
	# ...
```
```python
# figure.py line 2881
def set_canvas(self, canvas):
	"""
	Set the canvas that contains the figure
	Parameters
	----------
	canvas : FigureCanvas
	"""
	self.canvas = canvas
```

The class Figure is responsible for containing elements of Artists ([figure.py line 505](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/figure.py#L2881-L2889)) and an Artist has a reference for which Figure it belongs to ([artist.py line 756](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/figure.py#L2881-L2889)). 
```python
# figure.py line 505
def add_artist(self, artist, clip=False):
	# ...
	artist.set_figure(self)
	# ...
```
```python
# artist.py line 756
def set_figure(self, fig):
	# ...
	if self.figure is fig:
		return
	if self.figure is not None:
		raise RuntimeError("Can not put single artist in "
		  "more than one figure")
	self.figure = fig
	# ...
```

This means if we want to add some interactivity to an Artist element, we can go from Artist to Figure then Figure to FigureCanvasBase to register a callback function for a particular signal (event name).

<br>

### *"FigureCanvasBase contains a list of the common event names, but how are they registered?"*

Upon initialization, FigureCanvasBase is either assigned a Figure object or if one does not exist, creates a Figure object. This Figure object at initialization will create a CallbackRegistry object in cbook.py passing the common event names to be stored in CallbackRegistry.\_signals ([figure.py line 2498](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/figure.py#L2366-L2553) and [cbook.py line 175](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/cbook.py#L174-L181)).
```python
# figure.py line 2498
@_api.make_keyword_only("3.6", "facecolor")
def __init__(self,
		figsize=None,
		dpi=None,
		facecolor=None,
		edgecolor=None,
		linewidth=0.0,
		frameon=None,
		subplotpars=None,  # rc figure.subplot.*
		tight_layout=None,  # rc figure.autolayout
		constrained_layout=None,  # rc figure.constrained_layout.use
		*,
		layout=None,
		**kwargs
		):
	""" ... """
	# ...
	self._canvas_callbacks = cbook.CallbackRegistry(
	signals=FigureCanvasBase.events)
	# ...
```
```python
# cbook.py line 175
def __init__(self, exception_handler=_exception_printer, *, signals=None):
	self._signals = None if signals is None else list(signals)  # Copy it.
	# ...
```

The CallbackRegistry object maintains two mappings ([cbook.py line 171](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/cbook.py#L170-L172)):
```python
# cbook.py line 171
* callbacks: signal -> {cid -> weakref-to-callback}
* _func_cid_map: signal -> {weakref-to-callback -> cid}
```

* The first mapping stores for each signal another mapping that maps callback ids (cid) to a reference of the callback function
* The second mapping is the same but inverted. For each signal, there is a map that maps a reference of the callback function to cid.

The CallbackRegistry has three important functions:
* `connect()` - Adds the callback function to a given signal and updates the mappings ([cbook.py line 205](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/cbook.py#L205-L217)).
```python
# cbook.py line 205
def connect(self, signal, func):
    	"""Register *func* to be called when signal *signal* is generated."""
	# ...
```

* `process()` - For a given signal, it will call the associated callback function with any additional args ([cbook.py line 275](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/cbook.py#L275-L295)).
```python
# cbook.py line 275
def process(self, s, *args, **kwargs):
	"""
	Process signal *s*.

	All of the functions registered to receive callbacks on *s* will be
	called with ``*args`` and ``**kwargs``.
	"""
	# ...
```

* `disconnect()` - Removes the callback function based on the callback id (cid) and updates the mappings ([cbook.py line 249](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/cbook.py#L249-L263)).
```python
# cbook.py line 249
def disconnect(self, cid):
	"""
	Disconnect the callback registered with callback id *cid*.

	No error is raised if such a callback does not exist.
	"""
	# ...
```

These functions are then called in FigureCanvasBase under mpl_connect and mpl_disconnect respectively ([backend_bases.py line 2504](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/backend_bases.py#L2443-L2504) and [backend_bases.py line 2518](https://github.com/matplotlib/matplotlib/blob/0ed7cf586e0c6b7b81d2cf62df7be20425c9ed8b/lib/matplotlib/backend_bases.py#L2506-L2518)). 
```python
# backend_bases.py line 2504
def mpl_connect(self, s, func):
	"""
	Bind function *func* to event *s*.

	...
	"""
	return self.callbacks.connect(s, func)
```
```python
# backend_bases.py line 2518
def mpl_disconnect(self, cid):
	"""
	Disconnect the callback with id *cid*.

	...
	"""
	return self.callbacks.disconnect(cid)
```

When the user clicks on a key or a button for example, matplotlib will generate the proper event and call `process()`. The user would only have to concern themselves with defining a callback function to connect to a particular signal and disconnect it later once they're done using it. 

Here’s an [example](https://matplotlib.org/stable/users/explain/event_handling.html):
```python
fig, ax = plt.subplots()
ax.plot(np.random.rand(10))

def onclick(event):
    print('%s click: button=%d, x=%d, y=%d, xdata=%f, ydata=%f' %
          ('double' if event.dblclick else 'single', event.button,
           event.x, event.y, event.xdata, event.ydata))

cid = fig.canvas.mpl_connect('button_press_event', onclick)
```
<br>

### *"How is this related to the Observer pattern?"*

Recall that the Observer pattern in general has a publisher class that keeps track of subscribers and contains methods to add and remove subscribers. This allows classes that want to listen to a particular event to be notified.

Now observe the CallbackRegistry class. It maintains mappings that map a particular signal (an event that classes might want to know if/when it occurs) to callback functions so the class can do something with that information when the time comes. This gives users the flexibility to assign particular elements to listen for certain events via a callback function in response.

<br>

### UML Diagram:
![screensh](./diagrams/Observer%20Design%20Pattern.png)

This UML diagram further illustrates the Observer pattern. Matplotlib does a variation of the Observer pattern because it is designed to store the callback functions instead of the objects that inherit the Subscriber interface. CallbackRegistry is the publisher that has functions to add and remove callback functions with `connect()` and `disconnect()` and to notify is to call `process()`, iterating through the map of callbacks that match the signal.
