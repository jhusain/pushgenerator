# Push Generator Proposal (ES2016)

Push Generators are an alternative to the Async Generator proposal currently in the strawman stage. They are proposed for ES2016.

## The Problem

JavaScript programs are single-threaded and therefore must steadfastly avoid blocking on IO operations. Today web developers must deal with a steadily-increasing number of push stream APIs including...

* Server sent events
* Web sockets
* DOM events

Unfortunately consuming these data sources in JavaScript is inconvenient.  Asynchronous functions (proposed for ES7) provide language support for functions that push a single result, allowing loops and try/catch to be used for control flow and error handling respectively. ES6 introduced language support for producing and consuming functions that return multiple results. However no language support is currently proposed for functions that push multiple values. The push generator proposal is intended to resolve this discrepancy, and add symmetrical support for push and pull functions in JavaScript. 

## Iteration and Observation

Iteration and Observation are two patterns which allow a consumer to progressively consume multiple values produced by a producer. In each of these patterns, the producer may produce one of three types of notifications:

a) a value
b) an error
c) a final value 

When the producer notifies the consumer of an error or a final value, no further values will be produced.  Furthermore in both patterns the producer or consumer should be able to short-circuit at any time and elect to produce or receive no further notifications.

### Iteration

Iteration puts the consumer in control. In iteration the producer waits for the consumer to *pull* a value. The consumer may synchronously or asynchronously *pull* values from the producer (the iterator), and the producer must synchronously produce values.

In ES6, language support was added for producing and consuming values via iteration.

```JavaScript
// data producer
function* nums() {
  yield 1;
  yield 2;
  return 3;
}

// data consumer
function consume() {
  for(var x of nums()) {
    console.log(x);
  }
}

consume();
// prints
// 1
// 2
```

Generator functions are considered *pull* functions, because the producer delivers notifications in the return position of function. This is evident in the desugared version of the for...of statement:

```JavaScript
function nums() {
  var state = 0;
  return {
    next(v) {
      switch(state) {
        case 0:
          state = 1;
          return {value: 1, done: false};
        case 1:
          state = 2;
          return {value: 2, done: false};
        case 2:
          state = 3;
          return {value: 3, done: true};
        caase 3:
          
    }
}

function consume() {
  var generator = nums(),
    pair;
  
  // value received in return position of next()
  while(!(pair = generator.next()).done) {
    console.log(pair.value);
  }
}

consume();
// prints
// 1
// 2
```

The producer may end the stream by either throwing when next is called or returning an IterationResult with a done value of true. The consumer may short-circuit the iteration process by invoking the return() method on the data producer (iterator).
```JavaScript
generator.return();
```
Note that there is no way for the producer to asynchronously notify the consumer that the stream has completed. Instead  the producer must wait until the consumer requests a new value to indicate stream completion.

Iteration works well for consuming streams of values that can be produced synchronously, such as in-memory collections or lazily-computed values  (ex. fibonnacci sequence).  However it is not possible to push streams such as DOM events or Websockets as iterators, because they produce their values asynchronously. For these data sources is necessary to use observation.

### Observation

Observation puts the producer in control. The producer may synchronously or asynchronouly *push* values to the consumer's sink (observer), but the consumer must handle each value synchronously.  In observation the consumer waits for the producer to *push* a value.

The push generator proposal would add syntactic support for producing and consuming push streams of data to ES2016:

```JavaScript
// data producer
function*> nums() {
  yield 1;
  yield 2;
  return 3;
}

// data consumer
function*> consume() {
  for(var x on nums()) {
    console.log(x);
  }
}

// note that two functions calls are required, one to create the Observable and another to being observation
consume()();
// prints
// 1
// 2
```

Push Generator functions are considered *push*, because the producer delivers notifications in the *argument* position of the function. This is evident in the desugared version of the code above:

```JavaScript
function nums() {
  return {
    [Symbol.observer](generator = { next() {}, throw() {}, return() {} }) {
      var iterationResult = generator.next(1);
      if (iterationResult.done) break;
      iterationResult = generator.next(2);
      if (iterationResult.done) break;
      iterationResult = generator.return(3);
      
      return generator;
    }
  }
};

function consume() {
  return {
    [Symbol.observer](generator = { next() {}, throw() {}, return() {} }) {
      return nums()[Symbol.observer]({
        next(v) { console.log(v); }
      });
    }
  };
};

// note that two functions calls are required, one to create the Observable and another to being observation
consume()();
// prints
// 1
// 2
```

The push generator proposal would add symmetrical support for Iteration and Observation to JavaScript. Like generator functions, *push generator functions* allow functions to return multiple values. However push generator functions send values to consumers via *Observation* rather than Iteration. The for..._on_ loop is also introduced to enable values to be consumed via observation. The process of Observation is standardized using a new interface: Observable.

## Introducing Observable

ES6 introduces the Generator interface, which is a combination of two different interfaces:

1. Iterator
2. Observer

The Iterator is a data source that can return a value, an error (via throw), or a final value (value where IterationResult::done).

```JavaScript
interface Iterator {
  IterationResult next();
}

type IterationResult = {done: boolean, value: any}

interface Iterable {
  Iterator iterator();
}
```

The Observer is a data _sink_ which can be pushed a value, an error (via throw()), or a final value (return()):

```JavaScript
interface Observer {
  void next(value);
  void return(returnValue);
  void throw(error);
}
```

These two data types mixed together forms a Generator:

```JavaScript
interface Generator {
  IterationResult next(value);
  IterationResult return(returnValue);
  IterationResult throw(error);
}
```

Iteration and Observation both enable a consumer to progressively retrieve 0...N values from a producer. _The only difference between Iteration and Observation is the party in control._ In iteration the consumer is in control because the consumer initiates the request for a value, and the producer must synchronously respond. 

In this example a consumer requests an Iterator from an Array, and progressively requests the next value until the stream is exhausted.

```JavaScript
function printNums(arr) {
  // requesting an iterator from the Array, which is an Iterable
  var iterator = arr[Symbol.iterator](),
    pair;
  // consumer (this function)
  while(!(pair = iterator.next()).done) {
    console.log(pair.value);
  }
}
```

This code relies on the fact that in ES6, all collections implement the Iterable interface. ES6 also added special support for...of syntax, the program above can be rewritten like this:

```JavaScript
function printNums(arr) {
  for(var value of arr) {
    console.log(value);
  }
}
```

ES6 added great support for Iteration, but currently there is no equivalent of the Iterable type for Observation. How would we design such a type? By taking the dual of the Iterable type.

```JavaScript
interface Iterable {
  Generator [Symbol.iterator]()
}
```

The dual of a type is derived by swapping the argument and return types of its methods, and taking the dual of each term. The dual of a Generator is a Generator, because it is symmetrical. The generator can both accept and return the same three types of notifications:

1. data
2. error
3. final value

Therefore all that is left to do is swap the arguments and return type of the Iterator's iterator method and then we have an Observable.

```JavaScript
interface Observable {
  void [Symbol.observer](Generator observer)
}
```

This interface is too simple. If iteration and observation can be thought of as long running functions, the party that is not in control needs a way to short-circuit the operation. In the case of observation, the producer is in control. As a result the consumer needs a way of terminating observation. If we use the terminology of events, we would say the consumer needs a way to _unsubscribe_. To allow for this, we make the following modification to the Observable interface:

```JavaScript
interface Observation {
  void unobserve();
}

interface Observable {
  Observation [Symbol.observer](Generator observer)
}
```

This version of the Observable interface accepts a Generator and returns an Observation. The consumer can short-circuit observation (unsubscribe) by invoking the return() method on the Generator object returned for the Observable @@observer method. To demonstrate how this works, let's take a look at how we can adapt a common push stream API (DOM event) to an Observable.

```JavaScript
// The decorate method accepts a generator and dynamically inherits a new generator from it
// using Object.create. The new generator wraps the next, return, and throw methods, 
// intercepts any terminating operations, and invokes an onDone callback.
// This includes calls to return, throw, or next calls that return a pair with done === true
function decorate(generator, onDone) {
  var decoratedGenerator = Object.create(generator);
  decoratedGenerator.next = function(v) {
    var pair = generator.next(v);
    // if generator returns done = true, invoke onDone callback
    if (pair && pair.done) {
      onDone();
    }
    
    return pair;
  };
  
  ["throw","return"].forEach(method => {
    var superMethod = generator[method];
    decoratedGenerator[method] = function(v) {
      // if either throw or return invoked, invoke onDone callback
      onDone();
      superMethod.call(generator, v);
    };
  });
}

// Convert any DOM event into an push generator
Observable.fromEvent = function(dom, eventName) {
  // An Observable is created by passing the defn of its observer method to its constructor
  return new Observable(function observer(generator) {
      var handler,
        decoratedGenerator = 
          decorate(
              generator,
              // callback to invoke if generator is terminated
              function onDone() {
                   dom.removeEventListener(eventName, handler);
              });
        handler = function(e) {
          decoratedGenerator.next(e);
        };
      
      dom.addEventListener(eventName, handler);
      
      return decoratedGenerator;
  });
};
 
// Adapt a DOM element's mousemoves to an Observable
var mouseMoves = Observable.fromEvent(document.createElement('div'), "mousemove");

// subscribe to Observable stream of mouse moves
var decoratedGenerator = mouseMoves[@@observer]({
  next(e) {
    console.log(e);
  }
});

// unsubscribe 2 seconds later
setTimeout(function() {
  // short-circuit the observation/unsubscribe
  decoratedGenerator.return();
}, 2000);
```

Observable is the data type that a function modified by both * and async returns, because it _pushes_ multiple values.

|               | Sync          | Async         |
| ------------- |:-------------:|:-------------:|
| function      | T             | Promise<T>    |
| function*     | Iterator<T>   | Observable<T> |

 An Observable accepts a generator and pushes it 0...N values and optionally terminates by either pushing an error or a return value. The consumer can also short-circuit by calling return() on the Generator object returned from the Observable's Symbol.observer method. 

In ES7, any collection that is Iterable should also be Observable. Here is an implementation for Array.

```
Array.prototype[Symbol.observer] = function(observer) {
  var done,
    decoratedObserver = decorate(observer, () => done = true);
    
  for(var x of this) {
    decoratedObserver.next(v);
    if (done) {
      return;
    }
  }
  decoratedObserver.return();
  
  return decoratedObserver;
};
```

## Adapting existing push APIs to Observable

It's easy to adapt the web's many push stream APIs to Observable.

### Adapting DOM events to Observable
```JavaScript
Observable.fromEvent = function(dom, eventName) {
  // An Observable is created by passing the defn of its observer method to its constructor
  return new Observable(function observer(generator) {
      var handler,
        decoratedGenerator = 
          decorate(
              generator,
              // callback to invoke if generator is terminated
              function onDone() {
                   dom.removeEventListener(eventName, handler);
              });
        handler = function(e) {
          decoratedGenerator.next(e);
        };
      
      dom.addEventListener(eventName, handler);
      
      return decoratedGenerator;
  });
};
```

### Adapting Object.observe to Observable

```JavaScript
Observable.fromEventPattern = function(add, remove) {
  // An Observable is created by passing the defn of its observer method to its constructor
  return new Observable(function observer(generator) {
    var handler,
      decoratedGenerator =
        decorate(
            generator, 
            function() {
                remove(handler);
            });

    handler = decoratedGenerator.next.bind(decoratedGenerator);
    
    add(handler);

    return decoratedGenerator;
  });
};

Object.observations = function(obj) {
    return Observable.fromEventPattern(
        Object.observe.bind(Object, obj), 
        Object.unobserve.bind(Object, obj));
};
```

### Adapting WebSocket to Observable

```JavaScript

Observable.fromWebSocket = function(ws) {
  // An Observable is created by passing the defn of its observer method to its constructor
  return new Observable(function observer(generator) {
    var done = false,
      decoratedGenerator = 
        decorate(
          generator,
          () => {
            if (!done) {
              done = true;
              ws.close();
              ws.onmessage = null;
              ws.onclose = null;
              ws.onerror = null;
            }
          });
    
    ws.onmessage = function(m) {
      decoratedGenerator.next(m);
    };
    
    ws.onclose = function() {
      done = true;
      decoratedGenerator.return();
    };
    
    ws.onerror = function(e) {
      done = true;
      decoratedGenerator.throw(e);
    };
    
    return decoratedGenerator;
  });
}
```

### Adapting setInterval to Observable

```JavaScript
Observable.interval = function(time) {
  return new Observable(function observer(generator) {
      var handle,
          decoratedGenerator = decorate(generator, function() { clearInterval(handle); });

      handle = setInterval(function() {
          decoratedGenerator.next();
      }, time);

      return decoratedGenerator;
  });
};
```
## Observable Composition

The Observable type is composable. Once the various push stream APIs have been adapted to the Observable interface, it becomes possible to build complex asynchronous applications via composition instead of state machines. Third party libraries (a la Underscore) can easily be written which allow developers to build complex asynchronous applications using a declarative API similar to that of JavaScript's Array. Examples of such methods defined for Observable are included in this repo, but are _not_ proposed for standardization.

Let's take the following three Array methods:
```JavaScript
[1,2,3].map(x => x + 1) // [2,3,4]
[1,2,3].filter(x => x > 1) // [2,3]
```
Now let's also imagine that Array had the following method:
```JavaScript
[1,2,3].concatMap(x => [x + 1, x + 2]) // [2,3,3,4,4,5]
```
The concatMap method is a slight variation on map. The function passed to concatMap _must_ return an Array for each value it receives. This creates a tree. Then concatMap concatenates each inner array together left-to-right and flattens the tree by one dimension.
```JavaScript
[1,2,3].map(x => [x + 1, x + 2]) // [[2,3],[3,4],[4,5]]
[1,2,3].concatMap(x => [x + 1, x + 2]) // [2,3,3,4,4,5]
```
Note: Some may know concatMap by the name "flatMap", but I use the name concatMap deliberately and the reasons will soon become obvious.

These three methods are surprisingly versatile. Here's an example of some code that retrieves your favorite Netflix titles.

```JavaScript
var user = {
  genreLists: [
    {
      name: "Drama",
      titles: [
        { id: 66, name: "House of Cards", rating: 5 },
        { id: 22, name: "Orange is the New Black", rating: 5 },
        // more titles snipped
      ]
    },
    {
      name: "Comedy",
      titles: [
        { id: 55, name: "Arrested Development", rating: 5 },
        { id: 22, name: "Orange is the New Black", rating: 5 },
        // more titles snipped
      ]
    },
    // more genre lists snipped
  ]
}

// for each genreList, the map fn returns an array of all titles with
// a rating of 5.0.  These title arrays are then concatenated together 
// to create a flat list of the user's favorite titles.
function getFavTitles(user) {
  return user.genreLists.concatMap(genreList =>
    genreList.titles.filter(title => title.rating === 5));
}

// we consume the titles and write the to the console
getFavTitles(user).forEach(title => console.log(title.rating));

```
Using nearly the same code, we can build a drag and drop event. Observables are streams of values that arrive over time. They can be composed using the same Array methods we used in the example above (and a few more).
In this example we compose Observables together to create a mouse drag event for a DOM element.

```JavaScript
// for each mouse down event, the map fn returns the stream
// of all the mouse move events that will occur until the
// next mouse up event. This creates a stream of streams,
// each of which is concatenated together to form a flat
// stream of all the mouse drag events there ever will be.
function getMouseDrags(elmt) {
  var mouseDowns = Observable.fromEvent(elmt, "mousedown"),
  var documentMouseMoves = Observable.fromEvent(document.body, "mousemove"),
  var documentMouseUps = Observable.fromEvent(document.body, "mouseup");
  
  return mouseDowns.concatMap(mouseDown =>
    documentMouseMoves.takeUntil(documentMouseUps));
};

var image = document.createElement("img");
document.body.appendChild(image);

getMouseDrags(image).forEach(dragEvent => {
  image.style.left = dragEvent.clientX;
  image.style.top = dragEvent.clientY;
});
```

### Asynchronous Observation

Observation puts control in the hands of the producer. The producer may asynchronously send values, but the consumer must handle those values synchronously to avoid receiving interleaving next() calls. In some situations the consumer is unable to handle a value synchronously and must prevent the producer from sending more values until it has asynchronously handled a value. This pattern is known as _asynchronous observation_.  

One example of asynchronous observation is asynchronous I/O, in which both the sink and source are asynchronous.  The read stream must wait until the write stream has asynchronously handled a value. This is sometimes referred to as_back pressure_.

Asynchronous observation arises from the composition of for...on and await.  Here's an example:

```JavaScript
async function testFn() {
  var writer = new AsyncStreamWriter("/...");
  
  for(var x on new AsyncStreamReader("/...")) {
    await writer.write(x);
  }
}
```

Note that in the example above, the promise created by the write operation is being awaited within the body of the for…on loop. To avoid concurrent write operations, the Data source must wait until the data sink has asynchronously finished handling each value. How is this accomplished?

###The Asynchronous Observable

An asynchronous observable waits until a consumer has finished handling a value before sending more values.  This is accomplished by waiting on promises returned from the generator's next(), throw(), and return() methods.   Here’s an example of an asynchronous generator function that returns an asynchronous observable.

```JavaScript
function^^ getStocks() {
  var reader = new AsyncFileReader(“stocks.txt”);
  try { 
    while(!reader.eof) {
      var line = await reader.readLine();
      // If the yield expression is replaced by a promise, 
      // the loop is paused until the promise is fullfilled.
      await yield JSON.parse(line);
    }
  }
  finally {
    await reader.close();
  }
}
```

Note that the asynchronous observable returned by the function above awaits promises returned by the generator before sending more values. We can desugar the code above to this:

```JavaScript
function spawn(genF) {
  return new Promise(function(resolve, reject) {
    var gen = genF();
    function step(nextF) {
      var next;
      try {
        next = nextF();
      } catch(e) {
        // finished with failure, reject the promise
        reject(e); 
          return;
      }
      if(next.done) {
        // finished with success, resolve the promise
        resolve(next.value);
        return;
      } 
      else if (next.value && next.value.then) {
        // not finished, chain off the yielded promise and `step` again
        Promise.cast(next.value).then(function(v) {
          step(function() { return gen.next(v); });      
        }, function(e) {
          step(function() { return gen.throw(e); });
        });
      }
      else {
        // ES6 tail recursion prevents stack growth
        step(function() { return gen.next(next.value)});
      }
    }
    step(function() { return gen.next(undefined); });
  });
}

function() {
  return new Observable(function observer(generator) {
    var done,
        decoratedGenerator = Object.create(generator);
        
    ["return", "throw"].forEach(method => {
      decoratedGenerator[method] = function (arg) {
        done = true;
        generator[method].call(this, arg);
      };
    });

    decoratedGenerator.next = function(v) {
      var pair = generator.next.call(this, v);
      done = pair.done;
      return pair;
    };

    spawn(function*() {
      var reader,
        line,
        pair;

      // generator.return() could've been invoked before this function ran
      if (done) { return; }
          
      reader = new AsyncFileReader("stocks.txt"),
      try {
        while(!reader.eof) {
          // send promise to spawn fn to be resolved
          line = yield reader.readLine();
          // generator.return() could've been invoked while this promise was resolving
          if (done) { return; }
          // Send value to generator
          pair = decoratedGenerator.next(JSON.parse(line));
          if (done) {
            return;
          }
          else {
            // send promise (or regular value) to spawn fn to be resolved
            yield pair.value;
            // generator.return() could've been invoked while this promise was resolving
            if (done) { return; }
          }
        }
      }
      finally {
        // send promise (or regular value) to spawn fn to be resolved
        yield reader.close();
        // generator.return() could've been invoked while this promise was resolving
        if (done) { return; }                
      }
    }).then(
      v => {
        if (!done) {
          decoratedGenerator.return(v);
        }
      },
      e => {
        if (!done) {
          decoratedGenerator.throw(e);
        }
      });

    return decoratedGenerator;
  });
}
```

## A Note on Iterable and Observable Duality

The fact that the Observable and Iterable interface are not strict duals is a smell. If Observation and Iteration are truly dual, the correct definition of Iterable should be this:

```JavaScript
interface Iterable {
  Generator iterator(Generator);
}
```
In fact this definition is more useful than the current ES6 definition. In iteration, the party not in control is the producer. Using the same decorator pattern, the producer can now short-circuit the iterator without waiting for the consumer to call next. All the producer must do is invoke return() on the Generator passed to it, and the consumer will be notified. Now we have achieved duality, and given the party that is not in control the ability to short-circuit. I contend that collections should implement this new Iterable contract in ES7.

# Transpilation

Async generators can be transpiled into Async functions. A transpiler is in the works. Here's an example of the expected output.

The following code...

```JavaScript
function^^ getStockPrices(stockNames, nameServiceUrl) {
    var stockPriceServiceUrl = await getStockPriceServiceUrl();

    // observable.buffer() -> AsyncObservable that supports backpressure by buffering
    for(var name on stockNames.buffer()) {
        // accessing arguments array instead of named paramater to demonstrate necessary renaming
        var price = await getPrice(await getSymbol(name, arguments[1]), stockPriceServiceUrl),
            topStories = [];

        for(var topStory on getStories(symbol).take(10)) {
            topStories.push(topStory);

            if (topStory.length === 2000) {
                break;
            }
        }

        // grab the last value in getStories - technically it's actually the return value, not the last next() value.
        var firstEverStory = await* getStories();

        // grab all similar stock prices and return them in the stream immediately
        // short-hand for: for(var x on obs) { yield x }
        yield* getStockPrices(getSimilarStocks(symbol), nameServiceUrl);

        // This is just here to demonstrate that you shouldn't replace yields inside a function
        // scope that was present in the unexpanded source. Note that this is a regular 
        // generator function, not an async one.
        var noop = function*() {
            yield somePromise;
        };

        yield {name: name, price: price, topStories: topStories, firstEverStory: firstEverStory };
    }
}
```

...can be transpiled into the [async/await](https://github.com/lukehoban/ecmascript-asyncawait) feature proposed for ES7:

```JavaScript
function DecoratedGenerator(generator) {
  this.generator = generator;
  this.done = false;
}

DecoratedGenerator.prototype = {};
["next","throw","return"].forEach(function(method) {
  DecoratedGenerator.prototype[method] = function(v) {
    if (this.done) break;
    var iterationResult = this.generator[method](v);
    this.done = iterationResult.done;
    return iterationResult;
  };
});

function Observable(observer) {
  this.observer = observer;
}

Observable.of = function(v) {
  return new Observable(function(observer) {
    var iterResult = observer.next(v);
    if (iterResult.done) break;
    observer.return();
    return { dispose: function() };
  })
}

Observable.fromGenerator = function(genFn) {
  return new Observable(function(generator) {
    var subscription,
      iter = genFn(),
      process = function(v) {
        var result = iter.next(v);
        if (result instanceof Observable) {
          subscription = result.observer({
            next: function(v) {
            },
            throw: function(e) {
              return observer.throw(e);
            },
            return: function(v) {
              process(v);
            }
          });
        }
        else {
          return 
        }
      };
      generator = new DecoratedGenerator(generator);
      process();
      return { 
        dispose: function() {
          iter.return();            
          if (subscription) {
            subscription.dispose();
          }
        }
      };
  })
}
```

