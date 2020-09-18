## A 1.5D Lunar Lander Game in Haskell

We are going to write a small graphical game using Haskell, running in the environment provided by [http://code.world/haskell]()

Open it in your browser and we can start writing!

First we need a little preamble which I'm not going to explain, to set up some language options we are going to use, and to import library functions we want to call.

```
{-# LANGUAGE NamedFieldPuns #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE RecordWildCards #-}
import qualified Data.Text as T
import Text.Printf
import CodeWorld

main :: IO ()
main = undefined
```

Paste that code in and click run. You'll get an error, because our `main` function is [`undefined`](https://wiki.haskell.org/Undefined).

A CodeWorld game is implemented by running the `activityOf` function, supplied with the parameters which make up your game.

The type of `activityOf` is:

```
activityOf :: world -> (Event -> world -> world) -> (world -> Picture) -> IO ()
```

The identifier `world` is a type variable, indicating that each `world` in the signature is the same type.

In Java this signature would be:
```
void activityOf(
    T initialState, 
    BiFunction<Event,T,T> updateFn, 
    Function<T,Picture> visualizationFn
)
```

To begin with our `world` type will only have one value. We will always be flying through space:

```
data State = FLYING
```

We'll update our `main` function to be defined as:
```
main = activityOf initial change scene
```

And those functions will be:
```
initial = FLYING

change anyEvent state = state

scene state = codeWorldLogo
```

You can run your program now, and you should see the CodeWorld logo in the top right pane.

Even though our `change` and `scene` functions are being called continuously, we don't see any changes because they always return the same thing.

Let's replace the logo with our lunar lander:[^typeSignatures]

```
lander :: Double -> Picture
lander throttle = translated 0 0.39 ((polyline [(-0.15,-0.10), (-0.25,0), (-0.25,0.2), (-0.15,0.3), 
  (0.15, 0.30), (0.25,0.20), (0.25,0), (0.15,-0.10)]) & 
  (polyline [(-0.25, -0.1),(0.25, -0.1), (0.25, -0.25), (-0.25, -0.25), (-0.25, -0.1)]) &
  (polyline [(-0.25, -0.25), (-0.35, -0.4)]) & polyline([(0.25, -0.25), (0.35, -0.4)]) &
  (polyline [(-0.4, -0.4), (-0.3,-0.4)]) & polyline([(0.4, -0.4), (0.3,-0.4)]) &
  (polyline [(-0.05,-0.25),(-0.1,-0.35),(0.1,-0.35),(0.05,-0.25)]) &
  (polyline [(-0.1,-0.35),(0.1,-0.35),(0,-0.35-throttle * 0.55),(-0.1,-0.35)]))
  
scene state = lander 1.0
```

The `lander` function has a parameter `throttle` which controls the length of the rocket flame, but we are always setting that to one. Let's add the throttle setting to our `State`:

```
data State = FLYING {throttle :: Double}
```
with an initial value:
```
initial = FLYING { throttle = 0.5 }
```
and we'll use the value in the `scene`:[^patternMatching]
```
scene FLYING {throttle} = lander throttle
```

Our `change` function still works, because it's just returning the state unchanged at this point. 

Now we need to be able to change the throttle, which we'll do by adding some new function bodies for `change` which match the events we want to use to change the throttle.

```
change (KeyPress "A") FLYING {throttle} = FLYING {throttle = min 1.0 (throttle + 0.1)}
change (KeyPress "Z") FLYING {throttle} = FLYING {throttle = max 0.0 (throttle - 0.1)}
```
We keep the original `change` body, because that gets used for all the eventys that we don't care about (and would also get used for any State values which didn't match)

Now we can actually see an animation, let's add some physics.

```
data State = FLYING {
    throttle :: Double,
    yvelocity :: Double,
    height :: Double
    }
```

We'll start at rest and 10 units of height above the surface:
```
initial = FLYING {throttle=0.5, yvelocity=0.0, height=10}
```

Now v = ta, but we don't have any t, so we need a new pattern in our handler function:
```
change (TimePassing(dt)) (FLYING {throttle, yvelocity, height})   =
  FLYING {
  throttle = throttle, 
  yvelocity = yvelocity + (throttle - gravity) * dt, 
  height = max (height + yvelocity * dt) 0
  }
```

Where `gravity` is:

```
gravity = 0.5;  
```

The new members of our state data type also need to be added to the existing `change` patterns, so they become:
```
change (KeyPress "A") FLYING {throttle, yvelocity, height} = FLYING {throttle = min 1.0 (throttle + 0.1), yvelocity, height}
change (KeyPress "Z") FLYING {throttle, yvelocity, height} = FLYING {throttle = max 0.0 (throttle - 0.1), yvelocity, height}
```
Now we have a height we need to have the ground as well. We are going to leave the lunar lander fixed in the middle of the screen and move the ground relative to us.

So we add the lunar surface to the `scene`:

The surface is just a straight line.
```
surface = polyline([(-10,0), (10,0)])
```
We add it to the scene, translating it in the y axis by the current height of the lander:

```
scene FLYING{throttle, yvelocity, height} =
  (lander throttle)  & 
  (translated 0 (-height) surface)
```

The game needs to end somehow, so we should have a  'landed' state:
```
data State = FLYING {
    throttle :: Double,
    yvelocity :: Double,
    height :: Double
    } | LANDED
```
Which you reach when your height gets to zero:
```
change _ (FLYING {throttle, yvelocity, height=0.0}) = LANDED
```

It's important that this pattern goes before the `change (TimePassing(dt)) (FLYING {throttle, yvelocity, height}) = ...` pattern, otherwise this pattern with a more specific match for `height` will never match.

## Exercises for the reader

1. Introduce a CRASHED state if your yvelocity is too high when your height reaches zero.
1. Give the lander limited fuel, and set the throttle to zero when you run out.
1. Take the mass of the fuel into account whe n calculating the acceleration due to the rocket engine.
1. Add an instrument panel which tells you your fuel, velocity and height.
1. Use two keys to rotate the lander and add xvelocity and xposition to the FLYING state.
1. Create some more interesting terrain and detect when your lander hits it.

[^typeSignatures]: Type signatures are optional as Haskell can infer all the types in your program, but they are good for reminding yourself what your functions are, and for getting better error messages from the compiler by pinning down some types.

[^patternMatching]: We've provided the structure of our type as the parameter to the function, so that the compiler can unpack it and we can use the fields directly.

