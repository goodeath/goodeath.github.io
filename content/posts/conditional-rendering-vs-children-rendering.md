+++
title = 'Conditional vs Children Rendering Issue'
date = 2026-05-30T14:50:18-03:00
draft = false
+++

### Introduction 
Conditional rendering and children rendering are two different approaches to render components in react.
Not all **children rendering** are conditionals, but they can and that's when things get odd.

Even though we can achieve similar (almost equal) results, react renders them in a different way. Lets analyze the following example (Full code at the end).

Using conditional render, we have:

```typescript
import React from 'react';

const Countdown: React.FC<CountdownProps>  = ({initialCount}) => {
    const [model, setModel] = React.useState<CounterModel | null>(null);

    return <p>
        {model?.value ?? null} // Conditional rendered
    </p>
}
```

This example works just fine. `model` is initially null and we have a blank screen. So far,
nothing new. However, if we use children rendering: 

```typescript
import React from 'react';


const Countdown: React.FC<CountdownProps>  = ({initialCount}) => {
    const [model, setModel] = React.useState<CounterModel | null>(null);
    const [isVisible, setIsVisible] = React.useState(false);

    return (
        <ChildrenRendered condition={isVisible}>
            {model.value}
        </ChildrenRendered>
    )
}
const ChildrenRendered: React.FC<ChildrenRenderedProps> = ({condition, children}) => {
    if(condition) return children;
    return null;
}
```

Now we've got an exception

> TypeError: null is not an object (evaluating 'model.value')

### Root cause
***

Why is that? Lets dig into the react generated code for each case:

**filename**:`conditionalRendering.ts`

The `initialCount` became t0 and the line  `const t1 = isVisible ? model.value : null;`
is testing `isVisible` before evaluating `model.value`

```typescript
import { c as _c } from "react/compiler-runtime";
import React from "react";

const Countdown: React.FC<CountdownProps> = (t0) => {
  const $ = _c(2);
  const [model] = React.useState(null);
  const [isVisible] = React.useState(false);

  const t1 = isVisible ? model.value : null;
  let t2;
  if ($[0] !== t1) {
    t2 = <p>{t1}</p>;
    $[0] = t1;
    $[1] = t2;
  } else {
    t2 = $[1];
  }
  return t2;
};
```

**filename**: `childrenRendering.ts`

Whereas, with **children rendering** react first test if any of the related 
values are different from the previous render. 

`if ($[0] !== isVisible || $[1] !== model.value)`

```typescript
import { c as _c } from "react/compiler-runtime";
import React from "react";

const Countdown: React.FC<CountdownProps> = (t0) => {
  const $ = _c(3);
  const [model] = React.useState(null);
  const [isVisible] = React.useState(false);
  let t1;
  if ($[0] !== isVisible || $[1] !== model.value) {
    t1 = (
      <ChildrenRendered condition={isVisible}>{model.value}</ChildrenRendered>
    );
    $[0] = isVisible;
    $[1] = model.value;
    $[2] = t1;
  } else {
    t1 = $[2];
  }
  return t1;
};
```

The interesting point here is that, while using complex chain of 
**children rendering** you could forget to check if some object is 
null or undefined before accessing its attributes, assuming that
it won't throw, since the condition will be false.

### Full Examples:

`conditionalRendering.ts`
```typescript
import React from 'react';

type CountdownProps = {
    initialCount: number;
};

type CounterModel = {
    value: number;
};

const Countdown: React.FC<CountdownProps>  = ({initialCount}) => {
    const [model, setModel] = React.useState<CounterModel | null>(null);

    React.useEffect(() => {
        const id = setInterval(
            () => setModel( prev => {
                if(!prev) return prev;
                return {
                    value: prev.value - 1
                }
            })
        , 1000);
        return () => clearInterval(id);
    })

    return <p>
        {model?.value ?? null} // Conditional rendered
    </p>
}
```

`childrenRendering.ts`
```typescript
import React from 'react';

type CountdownProps = {
    initialCount: number;
};

type CounterModel = {
    value: number;
};

const Countdown: React.FC<CountdownProps>  = ({initialCount}) => {
    const [model, setModel] = React.useState<CounterModel | null>(null);
    const [isVisible, setIsVisible] = React.useState(false);

    useEffect(() => {
        const id = setInterval(
            () => setModel( prev => {
                if(!prev) return prev;
                return {
                    value: prev.value - 1;
                }
            })
        , 1000);
        return () => clearInterval(id);
    })

    return (
        <ChildrenRendered condition={isVisible}>
            {model.value}
        </ChildrenRendered>
    )
}

type ChildrenRenderedProps = {
    condition: boolean;
    children: React.ReactNode;
};

const ChildrenRendered: React.FC<ChildrenRenderedProps> = ({condition, children}) => {
    if(condition) return children;
    return null;
}
```
