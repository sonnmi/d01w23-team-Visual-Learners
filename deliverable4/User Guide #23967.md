# User Guide #23967

## User Defined Colors

Matplotlib allows users to define and manage the names of color. `pyplot.custom_colors` (which is a wrap of `matplotlib.colors._custom_colors`) is provided to register and unregister user defined color name and color pairs.

```python
import matplotlib.pyplot as plt
import matplotlib.colors as mcolors
import numpy as np

t = np.linspace(0.0, 2.0, 201)
s = np.sin(2 * np.pi * t)

# unregister all user defined colors
plt.custom_colors.unregister_all()
print(mcolors.get_custom_colors_mapping())
# {}
plt.custom_colors.register('cust_green', '#eaffd5')
plt.custom_colors.register('cust_blue', (0, 0, 1))
plt.custom_colors.register('cust_orange', '#ff7f0e')
print(mcolors.get_custom_colors_mapping())
# {'cust_green': '#eaffd5', 'cust_blue': (0, 0, 1), 'cust_orange': '#ff7f0e'}
fig, ax = plt.subplots(facecolor=(.18, .31, .31))
# set color to user defined colors directly
ax.set_facecolor('cust_green')
ax.set_ylabel('Voltage [mV]', color='cust_blue')
ax.tick_params(labelcolor='cust_orange')
ax.set_title('Voltage vs. time chart')
ax.set_xlabel('Time [s]')
ax.plot(t, s, 'xkcd:crimson')
ax.plot(t, .7*s, linestyle='--')
plt.custom_colors.unregister('cust_green')
# 'cust_blue' is overridden when *override* is True
plt.custom_colors.register('cust_blue', (0, 1, 1), override=True)
print(mcolors.get_custom_colors_mapping())
# {'cust_blue': (0, 1, 1), 'cust_orange': '#ff7f0e'}
plt.show()
```
![user_defined_color_plot](https://user-images.githubusercontent.com/57647309/230742533-e9fc38e4-e7b7-45bf-8d29-441ddc106c30.png)


After registering and unregistering, the updated user defined colors are persisted by Matplotlib. Therefore users do not need to maintain a dictionary of color names and colors and import the dictionary every time when they use it.

For `pyplot.custom_colors.register`, error will be raised if the *name* passed in is reserved for built-in color names, *name* already exists and *override* is not set to True, or *color* is an invalid color definition. For `pyplot.custom_colors.unregister`, error will be raised if *name* is not in customized color names.
