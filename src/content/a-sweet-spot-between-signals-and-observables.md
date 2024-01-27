---
title: A sweet spot between signals and observables 🍬
slug: a-sweet-spot-between-signals-and-observables
description: The migration wave to signals is real, state management libraries have started to add support to support both observables and signals. Angular has had observables since the beginning and the devs are…
coverImage: '/a-sweet-spot-between-signals-and-observables.webp'
coverImageAlt: 'Photo by Patrick Fore on Unsplash'
tags: [ 'angular', 'javascript', 'observables', 'signals' ]
publishedAt: '2023-08-18T00:00:00.000Z'
---

# A sweet spot between signals and observables 🍬

> In collaboration with [Chau Tran](https://twitter.com/Nartc1410).

The migration wave to signals is real, state management libraries have started to add support to support both observables and signals. 

This is my shot together with Chau to combine observables and signals into one. 

## Little history
Angular has had observables since the beginning and the devs are used to it and have used it in almost every part of their apps. 

**Angular's reactivity** was tied to **rxjs** (except zone.js automagic change detection)!

Then, SIGNALS 🚦 came! They changed everything! The way we see the templates, change detection and reactivity in Angular in general!  

Who would have thought Angular recommends calling functions in the template?! 

```ts
@Component({
    template: `<div>Count: {{ count() }}</div>`
})
export class MyCmp {
    count = signal(0);
}
```

(I did LOL! Read more here: [It’s ok to use function calls in Angular templates!](./blog/its-ok-to-use-function-calls-in-angular-templates))

Signals are great! But we are used to rxjs patterns in Angular, our services are tied to rxjs subjects, observables, operators and everything else!

## Example scenario
I have a **GalleryComponent** that retrieves some data from the API and it depends on an id (retrieved from route params) and global form filters. 

```ts
@Component({
    template: `
        <div *ngIf="data$ | async as data">
            {{ data | json }}
        </div>
    `
})
export class GalleryComponent {
    private route = inject(ActivatedRoute);
    private galleryService = inject(GalleryService);
    private filterService = inject(GlobalFilterService);
    
    galleryId$ = this.route.paramMap.pipe(map(p => p.get('id')!));
    
    data$ = combineLatest([
        this.filterService.filters$,
        this.galleryId$
    ]).pipe(
        switchMap(([filters, id]) => 
            this.galleryService.getGalleryItems(id, filters)
        )
    );
    
    favoritesCount$ = this.data$.pipe(map(data => getFavoritesCount(data)));
}
```

I want to use signals in the template for better performance, and because they are the future?! _BUT, I don't want to get rid of rxjs, I see them as complementary to each other! _

## Angular helper functions way

First thing I do is use `toSignal` to wrap my `data$` observable. 

```ts
data = toSignal(this.data$, { initialValue: [] });
```

Now I can use my data directly in the template as a signal without needing async pipe! Great! 

Because `favoritesCount$` depends only on `data$` I can convert that one into a signal too. Computed is the perfect fit for this!

```diff
- favoritesCount$ = this.data$.pipe(map(data => getFavoritesCount(data)));
+ favoritesCount = computed(() => getFavoritesCount(this.data()));
```

_Perfect!_

I go and update my `GlobalFilterService` from `BehaviorSubjects` or [`RxAngular`](https://www.rx-angular.io/) / `ComponentStore` to use signals!

And, now I cannot use my `filters` (that is now a signal) anymore in the `combineLatest`! Let's convert it back to observable in order to use it inside `combineLatest` again! 

Let's use `toObservable` from Angular. 


```diff
@Component()
export class GalleryComponent {
    ...
    private filterService = inject(GlobalFilterService);
    
+   filters$ = toObservable(this.filterService.filters);

    data$ = combineLatest([
-       this.filterService.filters$,
+       this.filters$,
        this.galleryId$
    ]).pipe(...);
}
```

Great until now! The thing is, I need to add an input to the component! This input will be used in the api call! So, I create it using a setter and a `BehaviorSubject`. 

```ts
showStars$ = new BehaviorSubject(false);

@Input({ required: true }) set showStars(x: boolean) {
    this.showStars$.next(x);
}
```

We can easily use that one in the `combineLatest` because it's just an observable! But I also need to use that in the template, so I use the `toSignal` again to convert it! 

```ts
showStars = toSignal(this.showStars$);
```

Great again! The thing is Angular RFC showed us that a new kind of input may be coming [RFC](https://github.com/angular/angular/discussions/49682);

So we may want to prepare when they come, to easy refactor to them, so what we may think is, let's make the setter set the value to a `signal` rather than a `BehaviorSubject` and from there convert it to an `observable`. 

And we get something like:

```ts
showStars = signal(false);
showStars$ = toObservable(this.showStars);

@Input({ required: true }) set showStars(x: boolean) {
    this.showStars.set(x);
}
```

The result of everything would look like this: 

```ts
@Component()
export class GalleryComponent {
    private route = inject(ActivatedRoute);
    private galleryService = inject(GalleryService);
    private filterService = inject(GlobalFilterService);
        
    galleryId$ = this.route.paramMap.pipe(map(p => p.get('id')!));
    filters$ = toObservable(this.filterService.filters);

    showStars = signal(false);
    showStars$ = toObservable(this.showStars);
    @Input({ required: true }) set showStars(x: boolean) {
        this.showStars.set(x);
    }

    data$ = combineLatest([
        this.filters$,
        this.galleryId$,
        this.showStars$
    ]).pipe(
        switchMap(([filters, id, showStars]) => 
            this.galleryService.getGalleryItems(id, filters, showStars)
        )
    );
    
    data = toSignal(this.data$, { initialValue: [] });
    
    favoritesCount = computed(() => getFavoritesCount(this.data()));
}
```

All this back and forth and decision dilemma because we just need to put the value in the `combineLatest` and be ready for _tomorrow's Angular_. 

What if we had a better alternative, something that also takes signals into consideration? 

## Chau's initial shot
Chau is one of the first devs who I talked to about the new Signals, and he was really hyped about it! He had been thinking about these patterns a lot! And showed me a solution he had created for combining signals with observables. 

The usage (different example from the initial one) looks like this: 

```ts
githubUsers = computed$(
    this.query,
    pipe(
      debounceTime(500),
      switchMap((query) => api.getUsers(query)),
      startWith([])
    )
);

// or with explicit initial value

readonly githubUsers = computed$(
    this.query,
    [], // initial value
    pipe(
      debounceTime(500),
      switchMap((query) => api.getUsers(query))
    )
);
```

`computed$` is a primitive where the first argument can be a signal, an observable or a promise, and the second arg can be either an **initialValue** or a pipe operator where we can pass all our rxjs operators. The operators would fire everytime the signal or observable changed!

Find the whole implementation here: 
https://gist.github.com/eneajaho/dd74aeecb877069129e269f912e6e472

What does `computed$` solve? Let's re-assess what we have and what we want to achieve:
- We want our `githubUsers` to be a `Signal` so we can use it on the template without `AsyncPipe`
- We have a `query` which is a `Signal` that reacts to a search input. The `query` value is used to call Github API for users
- We want to debounce the rate of which we call the Github API

> Please don't go and create a `debounceSignal()`. Use RxJS for asynchronous operations.


`@angular/core/rxjs-interop` provides two other primitives: `toSignal()` and `toObservable()` to solve this for us

```ts
githubUsers = toSignal(
    toObservable(this.query).pipe(
        debounceTime(500),
        switchMap((query) => api.getUsers(query))
    ),
    { initialValue: [] }
);
```

This approach works fine but the back-and-forth between `toSignal` and `toObservable` can be messy quickly. This is where `computed$` comes in. But it only works with a single source.

## My shot
Because I had more than one source that were of different types (both signals and observables), I took Chau's computed and converted the initial argument to be an array of sources (that could be either signals or observables). As a first time fixing Typescript types, I think it went really well! Find the implementation here. 

```ts
export function computedFrom<R, A extends readonly unknown[]>(
	sources: readonly [...ObservableSignalInputTuple<A>],
	initialValueOrOperator?: R | OperatorFunction<A, R>,
	operator?: OperatorFunction<A, R>
): Signal<R> {
	const obsSources = sources.map(x => {
		const obs$ = sourceToObservable(x);
		return obs$.pipe(distinctUntilChanged());
	});

	const initialValue =
		typeof initialValueOrOperator === 'function' ? undefined : initialValueOrOperator;

	const pipeOperators =
		typeof initialValueOrOperator === 'function' ? initialValueOrOperator : operator;

	const result$ = combineLatest(obsSources).pipe(
		pipeOperators as unknown as OperatorFunction<any, R>
	);

	return toSignal(result$, { initialValue }) as Signal<R>;
}

function sourceToObservable<T>(source: ObservableSignalInput<T>): Observable<T> {
	if (typeof source === 'function' && isSignal(source)) return toObservable(source);
	if (isObservable(source)) return source;
	return from(source);
}
```

Full implementation here: https://gist.github.com/eneajaho/53c0eca983c1800c4df9a5517bdb07a3

If I take the inital example I had before, and apply the updated computed$ (now called `computedFrom`) approach, it would look like this: 


```ts
@Component()
export class GalleryComponent {
    private route = inject(ActivatedRoute);
    private galleryService = inject(GalleryService);
    private filterService = inject(GlobalFilterService);
        
    galleryId$ = this.route.paramMap.pipe(map(p => p.get('id')!));

    showStars = signal(false);

    @Input({ required: true }) set showStars(x: boolean) {
        this.showStars.set(x);
    }
    
    data = computedFrom(
        [this.filterService.filters, this.showStars, this.galleryId$],
        [], // initial value
        pipe(
            switchMap(([filters, showStars, id]) =>
                this.galleryService.getGalleryItems(id, filters, showStars)
            )
        )
    );

        
    favoritesCount = computed(() => getFavoritesCount(this.data()));
}
```

As we can see, there's no need to convert things back and forth to combine signals and observable together! 

As I said before, this is fully typed. And there's one catch, if we don't pass an intial value our signal would be of type `Signal<T | undefined>` and if we pass an initial value it would be of type `Signal<T>`. 

I think this is important (passing an initial value) in cases when we have to use this signal in the template or in other computations, because we first have to check if our signal has a value or not.

Example with `favoritesCount`: 
```ts
favoritesCount = computed(() => {
    const data = this.data();
    if (data === undefined) return 0;
    return getFavoritesCount(data);
});
```

## Chau's bazooka
I go back to Chau with my solution, he initially likes it! And goes to improve it! Comes back saying that a flaw of my solution is the `initialValue`. 

So, if I don't provide an operator, the intial value of **computedFrom** will be of type **any**. But we don't want that, because we have values, and at least we should get back the values of our sources inside an array. 

Let's take an example to better understand it: 

```ts
const first = signal(1); // Signal with default value
const second$ = of(1); // Observable that emits first value synchronously
const third$ = timer(5000); // Observable that emits first value asynchronously

const combined = computedFrom(
    [first, second$, third$],
);
```

My implementation would return `Signal<any>` and it's value would be undefined! _Bad right? Yes, of course._

We at least want to get something like this: 

```ts
const combined = computedFrom(
    [first, second$, third$],
);

// typeof combined - Signal<[number, number, number]>
```

While this other example would return `Signal<number>` and the value would be 0.

```ts
const combined = computedFrom(
    [first, second$, third$],
    0, // initial value
);
```

But, Chau thinks that explicit `initialValue` is redundant because:
- Signals already have initial values
- Observables can emit values synchronously

He believes **only** `Observables` which emit values **asynchronously** should have their initial values **explicitly** defined. In the above example, only the `third$` source needs an initial value.

```ts
const combined = computedFrom(
    [first, second$, third$.pipe(startWith(0))],
    pipe(map(([f, s, t]) => f + s + t))
);

// if we have only one operator we can write it directly like: 
const combined = computedFrom(
    [first, second$, third$.pipe(startWith(0))],
    map(([f, s, t]) => f + s + t)
);
```

Last but not least, Chau's version of `computedFrom()` does also accept a `Dictionary` just like `combineLatest()` does. Which is a big plus in some scenarios I'd say.

```ts
const combined = computedFrom(
    {
        first: first, 
        second: second$, 
        third: third$.pipe(startWith(0))
    },
    map(({first, second, third}) => first + second + third)
);
```

## Use `computedFrom` from `ngxtension`
If you want to use this function, it's already published [here](https://ngxtension.netlify.app/utilities/signals/computed-from/) 

```ts
import { computedFrom } from 'ngxtension/computed-from';
```

# Thanks for reading!
If this article was interesting and useful to you, and you want to learn more about Angular, support me by [buying me a coffee ☕️](https://ko-fi.com/eneajahollari) or follow me on X (formerly Twitter) [@Enea_Jahollari](https://twitter.com/Enea_Jahollari) where I tweet and blog a lot about `Angular` latest news, signals, videos, podcasts, updates, RFCs, pull requests and so much more. 💎
