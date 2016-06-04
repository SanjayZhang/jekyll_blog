---
title: ViewRootImpl performTraversals
layout: post
tags:
 - android
---

流程图
---

```flow
st=>start
perform.traversals=>operation: performTraversals()
cond.measure=>condition: 重新measure()
cond.layout=>condition: 重新layout()
cond.draw=>condition: 重新draw()
sub.measure=>subroutine: view.measure()
sub.layout=>subroutine: view.layout()
sub.draw=>subroutine: view.draw()
e=>end

perform.traversals->cond.measure
cond.measure(yes, right)->sub.measure->cond.layout
cond.measure(no)->cond.layout
cond.layout(yes, right)->sub.layout->cond.draw
cond.layout(no)->cond.draw
cond.draw(yes, right)->sub.draw->e
cond.draw(no)->e
```

