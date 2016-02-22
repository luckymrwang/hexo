title: Draggable时阻止单击click事件
date: 2015-12-15 11:46:27
tags: [jQurye]
---

Draggable时阻止单击事件

```js
$('.selector').draggable({
    stop: function(event, ui) {
        // event.toElement is the element that was responsible
        // for triggering this event. The handle, in case of a draggable.
        $( event.toElement ).one('click', function(e){ e.stopImmediatePropagation(); } );
    }
});
```
This works because "one-listeners" are fired before "normal" listeners. So if a one-listener stops propagation, it will never reach your previously set listeners.