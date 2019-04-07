---
title: A neat DIY solution to animating on scroll (for any framework)
published: false
description: AOS libraries sure are slick, but why add dependencies when it's so dead simple to implement yourself?
tags: javascript, react, DIY
---

Looking around the world wide web for inspiration, I've found that many sites I love incorporate fun little "reveal" animations whenever I scroll to certain elements. Though subtle, these extra touches make the page feel much less static and more **responsive**. The question is though... what's the best way to implement this?

Just scrolling through CodePen examples, I have found [time](https://codepen.io/syedrafeeq/pen/yEJKn) and [time again]() that people are reaching for catch-all libraries that can handle it for them. There are countless options out there for animating on scroll, the most prevalent being the aptly named [AOS](https://github.com/michalsnik/aos). I myself was hoping to spice 🌶 up my site with some scroll anime and I naturally thought to turn to AOS library for this. However, as my implementation go more and more specialized (ex. how to I avoid loading this iFrame until I scroll to it?) I began to wonder...

*Can't I just build this myself?*

##Maybe, let's see how

Just starting with basic, vanilla JS and no frameworks, the approach is actually pretty simple. All we need an on scroll handler and whatever elements we actually want to animate. Starting with the basics, say we have an element of a specific ID we want to trigger an animation for. As you might imagine, we can reach for the DOM window's `onScroll` event to figure out where our element is on screen whenever you, well, scroll:

```javascript
window.onScroll = ({target}) => {
    const element = document.getElementById('animate-me')
    const elementTop = element.getBoundingClientRect().top
    if (elementTop < document.body.clientHeight) {
        // Make sure we don't add a duplicate class
        if (!element.classList.contains('scrolled-to')) {
            element.classList.add('scrolled-to')
        }
    }
}
```

There are a few nested object attributes we need to grab for this. First, we need to get the pixel value for where the top of the element is on screen. There are a few valid ways of finding this, but through a quick internet search it seems `getBoundingClientRect()` is the most reliable way of doing so across browsers. 

With this, we should compare against the fixed height of the document. This is basically just the height of your browser window, being the `clientHeight`. If the top of our element is less than this height, then some part of it must be on screen 👍 

###Okay great, we basically recreated a MDN help page example...

With that out of the way, let's actually make this useable in the real world. Firstly, if you got curious and threw a `console.log` statement in there, you likely got this whenever you so much as twitched your scroll wheel.

![Console log of on scroll handler](https://thepracticaldev.s3.amazonaws.com/i/defuaddw82tdkyrzkfpo.png)

This reflects how expensive analyzing every scroll event actually is. We're executing a function for every pixel we scroll, and as we start making this function more robust, that can start causing lags and stutters. 

One way to resolve this is using a `requestAnimationFrame` to decide when our callback gets fired. This is another window-level function where you may queue up callbacks for the browser to call, and when it feels it is ready to execute those functions without un-buttery-smoothing the animation, it will fire the callback. Thankfully, this option has seen [relatively high browser adoption](https://caniuse.com/#search=requestanimationframe) as well. All we need for this is a wrapper around our onScroll handler to `requestAnimationFrame`, along with a `boolean` flag to let us know whether or not our previous callback is done executing:

```javascript
let waitingOnAnimRequest = false

const animChecker = (target) => {
    // Our old handler
    const element = document.getElementById('animate-me')
    const elementTop = element.getBoundingClientRect().top
    if (elementTop < document.body.clientHeight) {
        if (!element.classList.contains('scrolled-to')) {
            element.classList.add('scrolled-to')
        }
    }
}

window.onScroll = ({target}) => {
    if (!waitingOnAnimRequest) {
        animChecker(target)
        waitingOnAnimRequest = false
    }
    waitingOnAnimRequest = true
}
```

Great! Now our calls should be a bit more efficient. But let's address a more pressing issue: how do we get this working for **any** element in the document we may want to animation on scroll?

It certainly wouldn't make sense to keep adding callbacks for each possible ID or className well would need, so why not just create a centralized array we can append all of our element selectors we want to animate?

##Time for some loops

This addition is fairly straightforward leveraging `querySelectorAll`. Just create a global array with all selectors that should animate (either IDs or classes) and loop over them like so:

```javascript
let animationSelectors = ['#ID-to-animate', 'class-to-animate']

const animChecker = (target) => {
    animationSelectors.forEach(selector => {
      target.querySelectorAll(selector).forEach(element => {
        const elementTop = element.getBoundingClientRect().top
        if (elementTop < bodyHeight) {
          if (!element.classList.contains('scrolled-to')) {
            element.classList.add('scrolled-to')
          }
        }
      })
    })
}
...
```

Now our scroll animation checker should be able to handle any element we throw at it!

##Neat! But I use X framework, and I don't think I could use this because of Y

Now hold it right there. I understand everyone's tooling has its own set of quirks, so let's try to address some of them.

###Problem A: I use a component system, so how do I centralize this logic?

Well, this solution thankfully just needs a single array of strings to get working, so any global state manager should do the trick! I used this in a recent project myself built on SvelteJS, which uses a subscription-based global store. To update `animationSelectors`, I just created it as a store...

```javascript
export const animationTriggers = writable({})
```

... and added the class name from whichever component when it gets created.

```javascript
import { animationTriggers } from '../stores'

onMount(() => {
    animationTriggers.set([
      ...$animationTriggers,
      '.doubleSidedCard',
      ['#' + sectionId],
    ])
  })
```

This works just as well for common global state solutions like Redux too:

```javascript
```