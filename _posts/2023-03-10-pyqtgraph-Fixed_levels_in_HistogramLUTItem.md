---
layout: post
title: pyqtgraph - Fixed levels in HistogramLUTItem
category: user interface
date: 2023-03-10 
tags:
  - PyQt5
  - pyqtgraph
  - python
---
This short post is about the taming of the pyqtgraph widget [HistogramLUTItem](https://pyqtgraph.readthedocs.io/en/latest/api_reference/graphicsItems/histogramlutitem.html) and preventing a linked `ImageItem` to reverse manually set levels in the histogram.
<!--more-->
# Problem
I love the histogram widget that shows the distribution of the grey-levels of an image and can be easily linked to an `ImageItem`  
```python
import pyqtgraph as pg
...
win = pg.GraphicsLayoutWidget()
view = win.addViewBox(lockAspect=True)
self.pg_img = pg.ImageItem()
view.addItem(self.pg_img)

self.histogram = pg.HistogramLUTItem(self.pg_img, fillHistogram=True)
win.addItem(self.histogram)
...
```
Now, say your camera is pushing new images every 1s to  `ImageItem` and you want to keep the max,min levels fixed such that you can do alignment or alike. I initially thought I could prevent these updates in the parent `ViewBox` of the `ImageItem` with methods such as `disableAutoRange()` or inside of the histogram itself with `disableAutoHistogramRange()` - I was mistaken unfortunately. 
# Solution
Whenever the histogram levels are updated there is a `sigLevelsChanged` signal emitted. This is the culprit for resetting our manually set levels. My solution allows the user to toggle between auto-update and allowing to adjust the histogram levels through the widget sliders. Let's check in which mode we are when setting the image data
```python
if self.bln_fix_hg_levels:
	self.pg_img.setImage(img, levels=self.hg_levels)
else:
	self.pg_img.setImage(img)
```
and elsewhere maybe we have a checkbox which clicked event leads to a toggled flag and get's us to update the `self.hg_levels` manually
```python
check = QCheckBox("Fix hg levels")
check.clicked.connect(self.upon_click)
...
def upon_click(self):
	self.bln_fix_hg_levels = self.sender().isChecked()
	if self.bln_fix_hg_levels:
		self.update_hg_levels()
		self.histogram.sigLevelsChanged.connect(self.update_hg_levels)
	else:
		self.histogram.sigLevelsChanged.disconnect(self.update_hg_levels)
```
in the update_hg_levels method we finally grab our levels
```python
def update_hg_levels(self):
	self.hg_levels = self.histogram.getLevels()
```
It works pretty well - happy me. As always it is good to look into the documentation ;). 