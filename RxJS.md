# RxJS

## Thinking Reactively with RxJS

Difficulty compounds with software complexity 

Decision to use RxJS: 

- Is there async or wait time? (timing is involved)
- Do we need to coordinate lots of even types? (clicks, keyboard events, http requests)

Requirements for app: 

- Show a loading spinner when anything is happening in the background
    - definitely time based!
    - involves coordinating all events happening in background
    - perfect for RxJS solution!

### Breaking Down the Problem:

- Show a spinner...until we hide it...
    - When does the loader show? When the count of async tasks goes from 0 to 1
    - When does the loader hide? When the count goes back to 0
- Define when an async task starts
- Define when an async task ends
- What does "show spinner" even mean!?

**Start with simple definitions:** `taskStarts`, `taskCompletions`, `showSpinner` as `Observable()`

remember to `import { Observable } from 'rxjs'` 

### First Problem:

How Do we Count? 

- Start count from zero
- when an async task starts, increase count by 1
- when a task ends, decrease count by 1

Create a `loadUp` and `loadDown` `Observable` to emit 1 and -1 when tasks start or complete, using mapTo: 

- `import { mapTo } from 'rxjs/operators'` , because we will be using `pipe`
- `const loadUp = taskStarts.pipe(mapTo(1))`
- `const loadDown = taskCompletions.pipe(mapTo(-1))`

import `merge` from rxjs to create a new `Observable`

- `const loadVariations = merge(loadUp, loadDown)` creates new `Observable` that combines the two

`**loadVariations` is now all we need to solve our problem!** 

Cognitive demand is much lower when we only need to deal with fewer pieces of information, and they are intelligently designed/named so that we can easily understand what they do (re: the naming convention of `loadUp`, `loadDown` and `loadVariations`. 

We can consider our problem solved if: we have an `observable` that gives the `currentLoadCount` of tasks in our app. So lets make it! 

    const currentLoadCount = loadVariations.pipe(
    	scan((totalCurrentLoads, changeInLoads) => {
    		return totalCurrentLoads + changeInLoads;
    }, 0)

- `scan` should be imported from `rxjs/operators`, and accepts the same parameters as `reduce` in JS, including the starting value after the function (which is `0` here)

We need to make `currentLoadCount` work as predictably as possible. 

*Good abstractions are predicatable* 

We won't get anything until `currentLoadCount` emits something, so we need it to initially emit `0`

To accomplish this: 

- import `startWith` from `rxjs/operators`
- add `startWith` to our `currentLoadCount` function like this:

    const currentLoadCount = loadVariations.pipe(
    	startWith(0);
    	scan((totalCurrentLoads, changeInLoads) => {
    		return totalCurrentLoads + changeInLoads;
    })

 *We can now remove our starting value `0`, and let our `startsWith(0)` flow through the `scan` function to return `0` initially* 

### What happens if we get more `taskCompletions` than `taskStarts`?

*This shouldn't happen*, but we can safeguard against this anyway by changing the function to check if `newLoadCount` is `< 0` and return `0` if it is, to prevent going into the negative. 

BUT then we might emit `0` over and over! 

**RxJS to the rescue!** 

- import `distinctUntilChanged` from `rxjs/operators`
- place this at the very end of `currentLoadCount`
- this will filter subsequent values that are equal (like repeating `0`s)

### -Quick Break-

- Taking a look at an emitter similar to our app, we can see that `scan` will keep separate states for separate subscribers, allowing us to return a running total for each subscriber
- Another way to consider this:
    - what state is `scan` holding?
        - transient?
        - a single source of truth?
    - Can add `share()` to give each subscriber the same value, but we don't get the current value when we add a subscriber
    - If we instead use `shareReplay(1)`, we get the current value only for the added subscriber, and then we continue to increment thereafter, for each added subscriber
    - **Problem:** default of `shareReplay` will keep a subscription source "alive" in the background which will keep racking up values, even after everything has unsubscribed from it
        - to fix this we can alter `shareReplay` :
            - `shareReplay({ bufferSize: 1, refCount: true })`
            - `refCount` will keep a reference of our subscribers, and when the number of subs drops to zero, it drops its source.

### Back to Our App...

- Our `currentLoadCount` is a single source of truth, so we *do not* want to keep a background task count.
- Add `shareReplay` the same as above at the end of the `currentLoadCount` function

### **Now we can move up a layer in abstraction...**

- When does our loader need to hide/show?
    - hide when the async count gets to `0`
    - show when the async count goes from `0` to `1`

Let's name our ``Observable`` accordingly: `shouldHideSpinner` 

- we import the `filter` operator and use `pipe` to pass it to the current load count, and check that the count is equal to `0`

    const shouldHideSpinner = currentLoadCount.pipe(
    	filter(count => count === 0);
    ); 

Now we can check when to **show ****our spinner... 

- again, we name accordingly: `shouldShowSpinner`
- again, we listen on the `currentLoadCount`
- and again, we use `filter`

We *could* filter and return any time the `count` is equal to `1`, but that's not really right

*what about when the count goes from 2 to 1? We don't need to show the spinner again. We just need to know when to show the spinner initially, when the count goes from 0 to 1.* 

So, we need to keep track of the previous count... But how? 

- We can import the `pairwise` operator, which emits the previous and current count.

Here is our final `shouldShowSpinner` function: 

    const shouldShowSpinner = currentLoadCount.pipe(
    	pairwise();
    	filter(([prevCount, currCount]) => prevCount === 0 && currCount === 1) 
    )

### Highest Level of Abstraction Achieved! 🎉

Now we can simply tackle our top level abstraction: showing the spinner until it's time to hide it... 

Currently, we consider `showSpinner` an ``Observable`` that when "activated" shows the spinner. 

*But* we're not worrying about how it does that just yet. 

When a task starts, switch to displaying the spinner, until it's time to hide it... 

    shouldShowSpinner.pipe(
    	switchMap(() => showSpinner.pipe(takeUntil(shouldHideSpinner)))
    ).subscribe(); 

### Now we can focus on making `taskStarts` and `taskCompletions` emit.

Tasks could be: 

- an `observable`
- a set timeout
- a fetch request
- etc

To be widely applicable, we'll go generic and use a function called `newTaskStarted`

(and `existingTaskCompleted` for when a task is completed) 

- Change each `observable` to a `Subject`
- Then, inside these functions, call the associated subject with `next()`
    - ie: `export function newTaskStarted() { taskStarts.next() }`

Then we import these functions into our React app's "slow" component, and call the `newTaskStarted` function when the associated buttons are clicked. 

We then call `existingTaskCompleted` inside of the `subscribe` for each `observable`. 

- we consider them completed whenever they emit

Inside of our other "quick" component we are using promises. So in this case: 

- call the `newTaskStarted` function right before the promise
- and call `existingTaskCompleted` when the promise resolves

### RECAP:

- took advantage of RxJS to create readable streams of logic
- paid attention to how we maintain these abstractions
- to keep  features usable, we exposed two simple functions that trigger all of our internal reactive logic via subject
- then imported these functions into a React app

### Display and Hide the Spinner:

- import loading spinner.
    - (implementation doesn't matter for this. it's just a spinner we are importing.)
- `Observable` accepts a callback that will be invoked anytime the `Observable` is subscribed to
    - we put the code to show our spinner inside this callback on our `showSpinner` `Observable`
- then we put a return inside the `Observable` callback, which will be invoked when the `observable`d is unsubscribed from
    - we put the code to hide the spinner inside this return

### **What if our async call is too quick to show the spinner?**

- if an async call resolves too quickly the action of showing the spinner will appear as a glitch
- we should only show the spinner once it's been active for at least 2 seconds... but how?

**New Abstraction:** When the spinner becomes active, wait for 2 seconds before showing it. BUT cancel showing it, if it becomes inactive again in the meantime. 

**When does the spinner need to show?** 

- Rename `shouldHideSpinner` to `spinnerDeactivated` and rename `shouldShowSpinner` to `spinnerActivated` to be more indicative of what they actually do (*replace any usage of these functions in the rest of your code!)*
- New `shouldShowSpinner` function:

    const shouldShowSpinner = spinnerActivated.pipe(
    	switchMap(() => timer(2000).pipe(takeUntil(spinnerDeactivated)))
    )

- Now the spinner will wait for 2 seconds before showing, and will not show when the action is too quick to warrant it

**But now our spinner looks glitchy when action takes slightly more than 2 seconds...** 

**New Requirement:** once the spinner is showing, keep showing it for *at least* 2 seconds 

- We need to listen for 2 events in parallel...
    - has it been showing for 2 seconds?
    - have the number of active tasks reached 0?
    - if both are true, we can hide the spinner...

- New `shouldHideSpinner` function:

    const shouldHideSpinner = combineLatest(
    	spinnerDeactivated, 
    	timer(2000)
    )

- Extract `timer(2000)` as our `flashThreshold`, or the minimum amount of time we want something to show on screen.... Replace any instance of `timer(2000)`

    const flashThreshold = timer(2000)
    
    const shouldHideSpinner = combineLatest(
    	spinnerDeactivated, 
    	flashThreshold
    )

- Spinner now hides *either* when it has been showing for 2 seconds, *or* when the number of tasks reaches 0 (in the case when this takes *longer than 2 seconds*)

### Percentage Progress Indicator...

- First we need to really understand the problem...
    - Imagine an array of running tasks...
        - as tasks in the array complete, the percentage completed goes up
        - if we add more tasks to the array while other tasks are still loading, the *total percentage completed* goes back down
        - then as more tasks complete the percentage goes back up until all tasks have completed

 **How do we show this on screen??** 

- our `initLoadingSpinner` (from our spinner library) takes two arguments: `total` and `completed`
- if we assign `6` to `total` and `5` to `completed`, we see a display of 83% when the spinner appears...

**How do we make these numbers dynamic?** 

- First, wrap this observable in a function that accepts these two variables as arguments
- If our tasks are an array, we can use the `.length` of the array as the total, and then calculated based on how many have completed, and how many have yet to load...
- We can use `currentLoadCount` to find this out...
    - when this count goes up, we have more loading
    - when this count goes down, it indicates that a task has completed
- Define a `loadStats` observable to track changes in loading tasks, with initial values set to 0

    const loadStats = currentLoadCount.pipe(
    	scan((loadStats, loadingUpdate) => {
    		const loadsWentDown = loadingUpdate < loadStats.previousLoading
    		const currentCompleted = loadsWentDown ? loadStats.completed + 1 : loadStats.completed
    		return {
    			total: currentCompleted + loadingUpdate,
    			completed: currentCompleted,
    			previousLoadingCount: loadingUpdate
    		}
    	}, {total: 0, completed: 0, previousLoadingCount: 0})
    )

**Now what happens when all tasks complete?** 

- We need to reset `total` and `completed` back to `0`
- It turns out we don't have to think about resetting our stats...
- We can tie the lifecycle of our `spinner` and `loadStats` together, so that both will be created and disposed of together

    const spinnerWithStats = loadStats.pipe(
    	switchMap( stats => showSpinner(stats.total, stats.completed))
    )

- This ensures that the `loadStats` state is local to the instance of spinner that it is tied to

### Combo Observable

**New Requirement:** Disable spinner when a certain combination of keys has been pressed (to facilitate testing, etc) 

- Whenever someone starts pressing the key combo, listen for them to complete the rest of the combo
- keep listening until the timer for listening has run out or a user presses an incorrect key
- stop listening when they complete the key combo (got combo.length - 1 key presses back)

    const anyKeyPresses = fromEvent(document, 'keypress').pipe(
    	map.(event = event.key)
    )
    
    function keyPressed(key) {
    	return anyKeyPresses.pipe(filter(pressedKey => pressedKey === key))
    }
    
    function keyCombo(keyCombo) {
    	const comboInitiator = keyCombo[0]
    	return keyPressed(comboInitiator).pipe(
    		switchMap(() => {
    			//WE ARE NOW IN COMBO MODE 
    			return anyKeyPresses.pipe(
    				takeUntil(timer(3000))
    				takeWhile((keyPressed, index) => keyCombo[index+1] === keyPressed)
    				skip(keyCombo.length - 2)
    				take(1)
    			)
    		})
    	)
    }
    
    const comboTriggered = keyCombo(['a','s','d','f'])
    
    interval(1000).pipe(
    	takeUntil(comboTriggered)
    ).subscribe({
    	next: x => console.log(x),
    	complete: () => console.log('COMPLETED')
    })

**RECAP:** 

- separate combo starting condition from the rest of our logic
- use various `take` operators to listen and dispose of the event listener based on various parameters