<h1 align="center">Angular best practices</h1>
<p align="center">
  <a href="https://angular.io" target="_blank"><img src="https://img.shields.io/badge/-Angular-red.svg?logo=Angular"></a>
  <a href="https://ngrx.io/" target="_blank"><img src="https://img.shields.io/badge/Angular-NgRX-purple.svg?logo=Angular"></a>
  <a href="https://webpack.js.org/" target="_blank"><img src="https://img.shields.io/badge/Bundler-Webpack-%238DD6F9.svg?logo=Webpack"></a>
  <a href="https://material.angular.io/" target="_blank"><img src="https://img.shields.io/badge/Angular-Material-blue.svg?logo=Angular"></a>
  <a href="https://angular.io/guide/universal" target="_blank"><img src="https://img.shields.io/badge/Angular-Universal-red.svg?logo=Angular"></a>
  <a href="https://angular.io/guide/router" target="_blank"><img src="https://img.shields.io/badge/Angular-Router-orange.svg?logo=Angular"></a></p>
<p align="center">
  <a href="https://angular.io/" target="_blank"><img src="https://toddmotto.com/assets/img/categories/angular.svg" width="150"></a>
</p>

### Description

Angular is a comprehensive JavaScript framework that is frequently used by developers for building cross-platform applications. Angular applications are quick, light, and simple-to-use. Therefore Angular is one of our favourite frameworks for web applications development.

But we should take into account the many subtleties due to which the application can turn into a continuous mess.

So let's begin!

### File structure

```
app/
├── animations/
├── components/
|   ├── main/
|   ├── profile/
|   ├── shop/
|   ├── ...
├── modules/
|   ├── admin/
|   ├── sections/
|   |   ├── section1/
|   |   ├── section2/
|   |   ├── section3/
|   |   ├── shared/           => shared only between these sections
|   |   ├── section.module.ts
├── directives/
├── pipes/
├── models/
├── resolvers/
├── services/
├── guards/
├── store/
|   ├── actions/
|   ├── reducers/
├── assets/
|   ├── images/
|   ├── svg/
|   ├── sprites/
|   ├── SCSS/
|   |   ├── media_queries/
|   |   ├── _components.scss
|   |   ├── _base.scss
|   |   ├── _buttons.scss
|   |	├── _cards.scss
|   |	├── ...
|   |   └── main.scss
├── shared/                  => shared through all app
|   ├── components/
|   ├── pipes/
|   ├── directives/
|   ├── shared.module.ts
app.component.html
app.component.ts
app.module.ts
```

> NOTE: such structure may not be suitable for all projects.

#### trackBy
Angular will re-render only those elements that have changed rather than whole DOM.
```js
// Template
<li *ngFor="let product of products; trackBy: trackByFn">{{ product.title }}</li>
// Component
trackByFn(index, product) {
   return product.id; // must be unique
}
```
#### Use async pipe
`async` pipes unsubscribe themselves automatically. It makes component more clean and prevents memory leaks if you forget to unsubscribe manually.
```javascript
// Template
<p>{{ iAmObservable$ | async }}</p>
// Component
this.iAmObservable$ = this.yourService.someObservable();
```

#### Do not forget to unsubscribe
Forgetting to unsubscribe from observables will lead to memory leaks even after a component has been destroyed.
```js
ngOnInit() {
  this.subscription = this.yourService.someObservable().subscribe(...);
}

ngOnDestroy() {
  this.subscription.unsubscribe();
}
```
- or you can consider the following approach

```js
ngUnsubscribe: Subject<void> = new Subject();

ngOnInit() {
  yourObservable.pipe(takeUntil(this.ngUnsubscribe)).subscribe();
}

ngOnDestroy() {
  this.ngUnsubscribe.next();
  this.ngUnsubscribe.complete();
}
```

Operators like `{ first, take, takeWhile }` might help you in some circumstances.

#### Use RxJS properly
Let's consider the most common operators (functions) and the differences between them.

- `throttleTime` vs `debounceTime`

```js
// wait 2 seconds before emit a new value
this.someObservable.pipe(throttleTime(2000))
```

```js
// new value will be emitted only if nothing happens for 2 seconds
this.someObservable.pipe(debounceTime(2000))
```

- `distinctUntilChanged`:

```js
searchField: FormControl = new FormControl();

// new value will be emitted only if it isn't the same as previous with 2 seconds delay
ngOnInit() {
  this.searchField.valueChanges.pipe(debounceTime(2000), distinctUntilChanged()).subscribe(...);
}
```

- `mergeMap`:

if you want to concurrently handle all the emissions


```js
field1: FormControl = new FormControl();
field2: FormControl = new FormControl();

// let's say "field1" you fieled with "hello" and "field2" with "world"
ngOnInit() {
  this.field1.valueChanges.pipe(
    mergeMap(value1 => this.field2.valueChanges.pipe(
      map(value2 => `${value1}--${value2}`)
    ))
  ).subscribe(...);  => final value is "hello--world"
}
```
> NOTE: A new value will be emitted only when all observables have emitted new value

- `switchMap`:

if you want to ignore the previous emissions when there is a new emission. It means when new value emits the previous subscription will be cancelled.

```js
someObservable1.pipe(
  switchMap(event => someObservable2)
).subscribe(...);
```

- `concatMap`:

if you want to handle the emissions one after another as they are emitted. It means `someObservable2` will emit new value in strict sequence one by one.

```js
someObservable1.pipe(
  concatMap(event => someObservable2)
).subscribe(...);
```

- `exhaustMap`:

if you want to cancel all the new emissions while processing a previous emission. It means `someObservable2` will listen to new emission only when previous is finished.
```js
someObservable1.pipe(
  exhaustMap(event => someObservable2)
).subscribe(...);
```

I strongly advise to go through this [documentation](https://xgrommx.github.io/rx-book/content/guidelines/introduction/index.html).

> NOTE: Avoid having subscriptions inside subscriptions.

#### Use Lazy loading
Perhaps one of the most useful features. Most likely that users will not visit all pages of the site, therefore consider splitting the app into small modules, decreasing the main bundle size. It can also help users with a weak internet connection to load the site faster.

```js
# app.routing-module.ts
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: '/path/to/admin/module#AdminModule'
  }
];
```

#### Choose Right Preload Strategy
By splitting our application into stand-alone modules we can lazy load the module when the user clicks the link leading to this module. But we can go further using build into Angular preload strategy or defining our own. Preload strategy gives us an ability to load all or some of modules in the background. It means Angular immediately starts rendering modules instead of waiting for the module to download over the network.

```js
@NgModule({
  imports: [RouterModule.forRoot(routes, { preloadingStrategy: PreloadAllModules })],
  exports: [RouterModule],
  providers: [AppCustomPreloader]
})
export class AppRoutingModule { }

```

But this strategy `PreloadAllModules` may not be the best solution since Angular will load all modules, even those that the user visits very rarely. What we can do?
Lets define our own Preload Strategy.

```js
# app.routing-module.ts
const routes: Routes = [
  { path: '', redirectTo: 'home', pathMatch: 'full' },
  { path: 'home', component: HomeComponent },
  {
    path: 'profile',
    loadChildren: '/path/to/profile/module#ProfileModule',
    data: { preload: true }                             // => will be preloaded in the backround
  },
  {
    path: 'admin',
    loadChildren: '/path/to/admin/module#AdminModule'   // => won't be preloaded in the backround
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes, { preloadingStrategy: CustomPreloader })],
  exports: [RouterModule],
  providers: [CustomPreloader]
})
export class AppRoutingModule { }
```

```js
# ./custom-preload-strategy.ts
import { PreloadingStrategy, Route } from '@angular/router';

import { Observable, of } from 'rxjs';

export class CustomPreloader implements PreloadingStrategy {
  preload(route: Route, load: Function): Observable<any> {
    return route.data && route.data.preload ? load() : of(null);
  }
}
```

#### Handle failed requests

Quite often API calls might fail. Consider building some logic to handle them.

```js
  ngOnInit() {
    this.productService.allProducts.retryWhen(this.handleRetry()).subscribe(...);
  }

  handleRetry() {
    return err => {
      return err.scan(count => {
        count++;
        if (count < 6) {
          return count;
        } else {
          throw(err);
        }
      }, 0).delay(5000);
    };
  }
```
> NOTE: This is just a basic example. A real project could have more complex logic.

#### Always declare variables and constants with a type


```js
myVar: string = 'Ruki bazuki';
someArray: Array<number> = [1, 2, 3];
anotherVar: 'a'|'b' = 'a';
```
#### Enforce certain rules in your code base
Always use lint rules. Find more [here](https://palantir.github.io/tslint/).

#### Small Reusable Components
Components should only deal with the display logic. Try to make them reusable if possible.

#### Add Caching

Do not make multiple unnecessary API calls to the same endpoint if data doesn't change often. Check if requested data is already present and update it only when necessary.

#### Avoid logic in templates

```html
// Template
<p *ngIf="showMe"> Hello world </p>
// Component
showMe (): boolean {
    return couldShow ? true : false;
}
```

#### Consider Using State Management

[@ngrx/store](https://github.com/ngrx/platform) can help maintain the state of your application more easily, keeping
state related logic separately, reducing complexity. It also has a nice memoization mechanism which might save your app performance.

#### Performance Tips

- **Webpack Bundle Analyzer**. Visualize size of Webpack output files with an interactive zoomable treemap. [Docs](https://www.npmjs.com/package/webpack-bundle-analyzer).

- **Lazy loading for images**. Well explained [here](https://blog.angularindepth.com/a-modern-solution-to-lazy-loading-using-intersection-observer-9280c149bbc).

- **Virtual Scrolling**. Find more info [here](https://material.angular.io/cdk/scrolling/overview).

- **Font loading strategy**. Nice article about the topic [here](https://www.malthemilthers.com/font-loading-strategy-acceptable-flash-of-invisible-text/).

#### Angular Universal

In case users are using not only Google and performance on mobile and low-powered devices is a must have, you might consider using Server-Side Rendering (SSR). Read more about Angular Universal [here](https://angular.io/guide/universal).


## About Codica

[![Codica logo](https://www.codica.com/assets/images/logo/logo.svg)](https://www.codica.com)

The names and logos for Codica are trademarks of Codica.

We love open source software! See [our other projects](https://github.com/codica2) or [hire us](https://www.codica.com/) to design, develop, and grow your product.
