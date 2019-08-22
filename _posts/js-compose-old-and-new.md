---
layout: post
title:  "Old vs new javascript compose function"
---

# Old vs new javascript compose function


``` javascript
function compose(f, g) {
  return function () {  
      return f(g.apply(null, arguments))
  }
}
```

``` javascript

cont compose = (f, g) => (...a) => f(g(...a))

```

## Key differences

*apply fn* vs *...*

*function* vs *() => {}*

*arguments* vs *...*