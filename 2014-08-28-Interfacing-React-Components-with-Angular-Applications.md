A guest post by Patrick Marabeas, a freelance frontend developer who loves learning and working with cutting edge web technologies. He spends much of his free time developing Angular Modules, such as: [ng-FitText](https://github.com/patrickmarabeas/ng-FitText.js), [ng-Slider](https://github.com/patrickmarabeas/ng-Slider.js), [ng-YouTubeAPI](https://github.com/patrickmarabeas/ng-YouTubeAPI.js) and [ng-ScrollSpy](https://github.com/patrickmarabeas/ng-ScrollSpy.js). You can follow him on Twitter: [@patrickmarabeas](https://twitter.com/PatrickMarabeas)

# Interfacing React Components with Angular Applications

There's been talk lately of using React as the view within Angular's MVC architecture. Angular, as we all know, uses dirty checking. As I'll touch on later, it accepts the fact of (minor) performance loss to gain the great 2 way data binding it has. React, on the other hand, uses a virtual DOM and only renders the difference. This results in very fast performance.

So, how do we leverage React's performance from our Angular application? Can we retain 2 way data flow? And just how significant is the performance increase?

The nrg module and demo code can be found [over on my GitHub](https://github.com/patrickmarabeas/nrg.js).

## The Application

To demonstrate communication between the two frameworks let's build a reusable Angular module (`nrg` [Angular(ng) + React(r) = energy(nrg)!]) which will render (and re-render) a React component when our model changes. The React component will be composed of an `input` and `p` element which will display our model and will also update the model on change. To show this, we'll add an `input` and `p` to our view - bound to the model. In essence, changes to either input should result in all elements being kept in sync. We'll also add a button to our component which will demonstrate component unmounting on scope destruction.


```javascript
;(function(window, document, angular, undefined) {
  'use strict';
  
  angular.module('app', ['nrg'])
  
    .controller('MainController', ['$scope', function($scope) {
      $scope.text = 'This is set in Angular';
      
      $scope.destroy = function() {
        $scope.$destroy();
      }
      
    }]);
    
})(window, document, angular);
```


`data-component` specifies the React component we want to mount. `data-ctrl` (optional) specifies the controller we want to inject into the directive - this will allow component specific to be accessible on `scope` itself rather than `scope.$parent`. `data-ng-model` is the model we are going to passing between our Angular controller and our React view.


```html
<div data-ng-controller="MainController">

  <!-- React component -->
  <div data-component="reactComponent" data-ctrl="" data-ng-model="text">
    <!-- <input /> -->
    <!-- <button></button> -->
    <!-- <p></p> -->
  </div>
  
  <!-- Angular view -->
  <input type="text" data-ng-model="text" />
  <p>{{text}}</p>
  
</div>
```


As you can see, the view has meaning when using Angular to render React components. `<div data-component="reactComponent" data-ctrl="" data-ng-model="text"></div>` has meaning when compared to `<div id="reactComponent"></div>` which requires referencing a script file to see what component (and settings) will be mounted on that element.


## The Angular Module - nrg.js

The main functions of this reusable Angular module will be to:

1. specify the DOM element the component should be mounted onto
1. render the React component when changes have been made to the model
1. pass the scope and element attributes to the component
1. unmount the React component when the Angular scope is destroyed

The skeleton of our module looks like so:


```javascript
;(function(window, document, angular, React, undefined) {
  'use strict';
  
  angular.module('nrg', [])
```


To keep our code modular and extensible we'll create a factory which will house our component functions - currently just `render` and `unmount`.


```javascript
    .factory('ComponentFactory', [function() {
      return {
        render: function() {
        
        },
        unmount: function() {
        
        }
      }
    }])
```


This will be injected into our directive.


```javascript
    .directive('component', ['$controller', 'ComponentFactory', function($controller, ComponentFactory) {
      return {
        restrict: 'EA',
```


If a controller has been specified on the elements via `data-ctrl`, then inject the `$controller` service. As mentioned earlier, this will allow scope variables and functions to be used within the React component to be accessible directly on `scope`, rather than `scope.$parent` (the controller also doesn't need to be declared in the view with `ng-controller`).


```javascript
        controller: function($scope, $element, $attrs){
          return ($attrs.ctrl) 
            ? $controller($attrs.ctrl, {$scope:$scope, $element:$element, $attrs:$attrs}) 
            : null;
        },
```


Isolated scope with two-way-binding on `data-ng-model`.


```javascript
        scope: {
          ngModel: '='
        },
        link: function(scope, element, attrs) {
        
          // Calling ComponentFactory.render() & watching ng-model
        
        }
      }
    }]);
    
})(window, document, angular, React);
```


### ComponentFactory

Fleshing out the `ComponentFactory`, we'll need to know how to render and unmount components.


```javascript
React.renderComponent(
  ReactComponent component,
  DOMElement container,
  [function callback]
)

React.unmountComponentAtNode(DOMElement container)
```


As such, we'll need to pass the component we wish to mount (`component`), the container we want to mount it at (`element`) and any properties (`attrs` & `scope`) we wish to pass to the component. This render function will be called every time the model is updated - so the updated scope will be pushed through each time.

> "If the React component was previously rendered into container, this (React.renderComponent) will perform an update on it and only mutate the DOM as necessary to reflect the latest React component." - [React docs](http://facebook.github.io/react/docs/top-level-api.html#react.rendercomponent)


```javascript
.factory('ComponentFactory', [function() {
  return {
    render: function(component, element, scope, attrs) {
    
      // If you have name-spaced your components, you'll want to specify that here - or pass it in via an attribute etc
      React.renderComponent(window[component]({
        scope: scope,
        attrs: attrs
      }), element[0]);
    },
    
    unmount: function(element) {
      React.unmountComponentAtNode(element[0]);
    }
  }
}])
```


### Component Directive

Back in our directive we can now setup when we are going to call these two functions.


```javascript
link: function(scope, element, attrs) {

  // Collect the elements attrs in a nice usable object
  var attributes = {};
  angular.forEach(element[0].attributes, function(a) {
    attributes[a.name.replace('data-','')] = a.value;
  });
  
  // Render the component when the directive loads
  ComponentFactory.render(attrs.component, element, scope, attributes);
  
  // Watch the model and re-render the component
  scope.$watch('ngModel', function() {
    ComponentFactory.render(attrs.component, element, scope, attributes);
  }, true);
  
  // Unmount the component when the scope is destroyed
  scope.$on('$destroy', function () {
    ComponentFactory.unmount(element);
  });
  
}
```


This implements dirty checking to see if the model has been updated. I haven't played around too much to see if there's a notable difference in performance between this and using a broadcast/listener. That said - to get a listener working as expected, you will need to wrap the render call in a `$timeout` to push it to the bottom of the stack to ensure `scope` is updated.


```javascript
scope.$on('renderMe', function() {
  $timeout(function() {
    ComponentFactory.render(attrs.component, element, scope, attributes);
  });
});
```


## The React Component

We can now build our React component which will use the model we defined as well as inform Angular of any updates it performs.


```javascript
/** @jsx React.DOM */

;(function(window, document, React, undefined) {
  'use strict';
  
  window.reactComponent = React.createClass({
```


This is the content which will be rendered into the container. The properties that we passed to the component (`{ scope: scope, attrs: attrs }`) when we called `React.renderComponent` back in our component directive are now accessible via `this.props`.


```javascript
    render: function(){
      return (
        <div>
          <input type='text' value={this.props.scope.ngModel} onChange={this.handleChange} />
          <button onClick={this.deleteScope}>Destroy Scope</button>
          <p>{this.props.scope.ngModel}</p>
        </div>
      )
    },
```

  
Via the `onChange` event we can call for Angular to run a digest - just as we normally would, but accessing `scope` via `this.props`.


```javascript
    handleChange: function(event) {
      var _this = this;
      this.props.scope.$apply(function() {
        _this.props.scope.ngModel = event.target.value;
      });
    },
```
  
  
Here we deal with the click event `deleteScope`. The controller is accessible via `scope.$parent`. If we had injected a controller into the component directive, its contents would be accessible directly on `scope` - just as ngModel is.
  
  
```javascript
    deleteScope: function() {
      this.props.scope.$parent.destroy();
    }
  });
})(window, document, React);
```


## The Result

Putting this code together (You can view the completed [code on GitHub](https://github.com/patrickmarabeas/nrg.js), or [see it in action](http://patrickmarabeas.github.io/nrg.js/)) we end up with:

1. Two `input` elements, both of which update the model. Any changes in either our Angular application or our React view will be reflected in both.
2. A React component button which calls a function in our `MainController`, destroying the scope - also resulting in the unmounting of the component.


Pretty cool. But where is my perf increase!? This is obviously too small an application for anything to be gained by throwing your view over to React. To demonstrate just how much faster applications can be (by using React as the view), we'll throw a kitchen sink worth of randomly generated data at it. 5000 bits to be precise. 
 
 Now, it should be stated that you *probably* have a pretty questionable UI if you have this much data binding going on. Misko Hevery has a great response regarding Angular's performance on [StackOverflow](http://stackoverflow.com/a/9693933/1455709). In summary:
 
> Humans are:
>
> * slow - Anything faster than 50ms is imperceptible to humans and thus can be considered as "instant".
> * limited - You can't really show more than about 2000 pieces of information to a human on a single page. Anything more than that is really bad UI, and humans can't process this anyway.

Basically, know Angular's *and* your user's limits! 

That said, the following performance test was certainly accentuated on mobile devices. Though, on the flip side, UI should be simpler on mobile.


## Brute Force Performance Demonstration

```javascript
;(function(window, document, angular, undefined) {
  'use strict';

  angular.module('app')
    .controller('NumberController', ['$scope', function($scope) {
    
      $scope.numbers = [];
      ($scope.numGen = function(){
        for(var i = 0; i < 5000; i++) {
          $scope.numbers[i] = Math.floor(Math.random() * (999999999999999 - 1)) + 1;
        }
      })();
    }]);
    
})(window, document, angular);  
```

### Angular ng-repeat


```html
<div data-ng-controller="NumberController">
  <button ng-click="numGen()">Refresh Data</button>
  <table>
    <tr ng-repeat="number in numbers">
      <td>{{number}}</td>
    </tr>
  </table>
</div>
```

![Angular Performance](/images/angular-perf.jpg)

There was definitely lag felt as the numbers were loaded in and refreshed. From start to finish - around **1.5 seconds**.

### React Component


```html
<div data-ng-controller="NumberController">
  <button ng-click="numGen()">Refresh Data</button>
  <div data-component="numberComponent" data-ng-model="numbers"></div>
</div>
```

```javascript
;(function(window, document, React, undefined) {
  window.numberComponent = React.createClass({
    render: function() {
      var rows = this.props.scope.ngModel.map(function(number) {
        return (
          <tr>
            <td>{number}</td>
          </tr>
        );
      });

      return (
        <table>{rows}</table>
      );
    }
  });
})(window, document, React);
```

![React Performance](/images/react-perf.jpg)

So that just happened. **270 milliseconds** start to finish. Around **80% faster**!

## Conclusion

So, should you go rewrite all those Angular modules as React components? Probably not. It really comes down to the application you are developing and how dependent you are on OSS. It's definitely possible that a handful of complex modules could put your application in the realm of 'feeling a tad sluggish', but it should be remembered that *perceived performance* is all that matters to the user. Altering the manner in which content is loaded could end up being a better investment of time.

User's will definitely feel performance increases on mobile websites sooner, however, and is certainly something to keep in mind.

The nrg module and demo code can be found [over on my GitHub](https://github.com/patrickmarabeas/nrg.js).