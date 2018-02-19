---
layout: post
title: "Progress bar for Caffe"
categories: programming python deeplearning
---

### Notice
This functionality should work until caffe doesn't change its output format


### Background
I had a chance to introduce __Intel Caffe__ to engineers and I noticed there might be no progress bar functionality like another Deep Learning frameworks.
To show clear and easy to understand output, I made wrapper for this.
If caffe natively has it, please let me know by comment.

You can donwload code below by

``` shell
wget https://gist.githubusercontent.com/ami-GS/50bb3995b337c1ec5b4222b3ee8f5dd1/raw/eb389d864abba124d51854107f587591dbc865b3/progress_bar.py
```

### gist code

<script src="https://gist.github.com/ami-GS/50bb3995b337c1ec5b4222b3ee8f5dd1.js"></script>


Then you might be able to run by
``` python
python progress_bar.py ./build/tools/caffe train --solver=OOOO_solver.prototxt
```

The argument following `progress_bar.py` is same as the way to use caffe. (OOOO can be changed to your .prototxt file name)

### TODO
add example image here

