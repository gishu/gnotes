% What I know about CLOJURE
Gishu Pillai  

# Language

A Var is mapping of symbol to value. Use vars for *stable* global values.
Let blocks are clojure's local variables

```clojure
 (def sitcom "Seinfeld")	; 'user/sitcom
 (.get *1)   			;"Seinfeld"
 (.-sym *2)			;sitcom
```

## Functions

* compose
* threading : -> (first arg) and ->> (last arg)
* apply func args



## Namespaces

**Including ns**

```clojure
(require 'other-ns)
;; OR
(ns this-ns
    (:require other-ns))
```

**Alias - import and rename**

```clojure
(ns this-ns
    (:require [other-ns :as op]))

(op/the-func)
```

**Refer - pull in var**

```clojure
(require '[other-ns :refer [the-func]])	; seealso :refer :all
(the-func)
```

Use rarely because they can shadow existing VARs.

NOTE: `*ns*` refers to the current ns
 ```clojure
 (find-ns 'user)
 (ns-map 'user)
 ```

Keywords can be qualified :other-ns/the-func or ::the-func (use cur ns)

ns are only LOADED ONCE. You can get around this by
```clojure
(require :reload '[..])	     ; force reload ns

(defonce cannot-reload-me 10)  	     ;only once
(ns-unmap *ns* 'cannot-reload-me)    ; last recourse 
```


## Sequences

Adapter pattern. Most collections respond to a seq method, which returns a sequence.
4 basic operations on sequences

* first
* rest
* next - like rest but returns nil for empty
* cons - prepend item to list

```clojure  
(defn my-count [coll]
  (loop [count 0 s (seq coll)]
    (println count s)
    (if (seq s)
      (recur (inc count) (rest s))
      count)))

```

There are functions that accept seq and

* filter
  * (filter predicate seq)
* transform
  * (map func seq)
* reduction
  * (reduce func seed seq) 
* generate
  * (re-seq #"reg expr" "match-match-match")
  * (for [b '(1 2 3)] (* b b))


### LAZY Seq
realized only on demand. nil indicates end-of-seq.

```clojure
(take 10 (repeat 'abc'))
(repeatedly func)
(map #(* 2 %) (iterate func seed))

;; magic sauce
(defn faux-repeat [x]
      (cons x (lazy-seq (faux-repeat x))))

;; doseq is similar to for. Holds onto one item at a time. doall returns all.
;; 
```

## Destructuring

```clojure

(let [[_ _ third fourth] '(sa re ga ma pa)]
  (println third fourth))     	    ;; ga ma

(let [ {writer :larry} {:jeff "green" :larry "david"}]
  (println writer))    	      	    ;; "david"

(def authors [{:name "JA" :born 1775} {:name "CD" :born 1812}])
(let [ [{dob1 :born} {dob2 :born}]   authors]
   (println dob1 dob2))	   	     ;; 1775 1812

; {:keys [name age gender]} shorthand for {name :name, ...}
; {:as args/opts}   	   	    grab everything being destructured
; {:or defaults_map}		    fill in missing keys

```

## Types

### Records

NOTE: A class is generated with predefined accessors.

1. include all map-like functionality
2. more performant for specified attributes
3. more readable

```clojure
(defrecord Car [make model year])

; factory methods generated to create instances
(->Car "Fiesta" "B299" 2012)
(map->Car argsMap)`

; add arbitrary attribute - perf similar to maps 
(assoc myCar :color "Blue")

; test functions class? instance?
```

## Protocols

```clojure
(defprotocol Persist
  (load [])
  (save [obj]))

(extend-protocol Persist
  Buy
  (load [] (println "Loaded"))
  (save [obj] (println "SavedObj"))
  Sell
  ...)
  
```

## State/Identity

### Ref Types

implement IRef. Do not block when deref with @

Type | Create | Update | Setter
----|---|---|---|
Atom | atom | swap! | reset! |
Ref | ref | alter, commute | ref-set |
Var | def | alter-var-root | var-set |
Agent | agent | send, send-off | restart-agent |

* Atoms are sync updated, independent, will retry if item changed concurrently
* Refs - used to modify a group of things within a software transaction i.e. a `dosync` block. Retries if any item has changed concurrently.
  ** alter will retry. Use commute to not check previous value - no retries, performant.
* Vars - thread local. No need to manage state.
* Agents - updates are async. Updates are held till enclosing txn commits


**Pattern**

* (create-fn container)
* (update-fn container setter-fn & args)
* (setter-fn container new-value)

setter-fn is applied to the current value to yield the next state

Ref Types also support watchers (change-notific.) and validators

```clojure
(println "Watchers! - Change notif.")

(defn a-watcher
  [key identity old new]
  (println (format "event {Key=%s Id=%s %s=>%s}" key identity old new))
  )

(def atom1 (atom {:age 10}))
(add-watch atom1 :watch1 a-watcher)
  (compare-and-set! atom1 @atom1 {:age 50} )

(add-watch atom1 :watch2 a-watcher)
  (swap! atom1 update-in [:age] + 5)
(remove-watch atom1 :watch1)
; Note: handlers are called in reverse order

(println "Validators - integrity")

(def counter (atom 0))

(compare-and-set! counter 0 100)

; function should return true for valid - else false/throw
(set-validator! counter pos?)

(compare-and-set! counter 100  -10)

```


## Concurrency

All of the following

1. block if the result is not ready
2. result is cached, if deref/fetched multiple times

### Future
To move work onto a threadpool thread, fire-and-forget. Returns a ticket that can be deref (blocking) to get the value (cached)

```
(future [form])		;=> future object
; futureObj-[cancel/done?/cancelled?]
; deref/ @futureObj => result
```

### Promise
Promise transfers one value from one thread to another.
```
(let [p (promise)]
     (deliver p 42)
)
;	realized? non blocking completion check
;	@p => result
```

### Delays
Delays are tasks that are not immediately executed until they are `forced`. 

;; doesn't execute immediately until forced
(let [notify (delay (println "Delayed!"))]
  (force notify)
)
;; same as deref a future

### Misc

Clojure has a pmap (parallel execution via futures), when item processing outweighs the sync overhead

Better yet, are clojure.core.reducers (aliased as r here)
```(->> products
     (r/filter ground?) (r/map :weight) (r/fold +))
```
fold partitions the coll into groups, reduces in parallel and combines
r/func combine to a single reducer function (xducer) avoiding intermediate results. They themselves dont transform the coll.
i.e. utilize all available cores + less garbage created.
LIMIT: fold works only on persistent maps & vectors.

### core.async library

Need to add dep to core.async

```clojure
(def echo-chan (chan))
; consumer
(go (println (<! echo-chan)))

; prod
(>!! echo-chan "cake!")

(let [t (thread "3-layer-cake")]
  (<!! t))	
; => 3LC


;; alts!!
(let [ [result ch] (alts!! [c1 c2 (timeout 20)])  ]
```

* chan - creates channels, which communicate messages. Processes wait for completion of put/take. By default, unbuffered. (chan 2) creates a buffer of 2 msgs. when the limit is reached, the put/take blocks.
  * sliding FIFO dropping LIFO
  * <! >! used in go blocks - parks (doesn't block thread). <!! >!! to be used outside of go blocks - blocks.
* go - creates go blocks, that execute on thread-pool threads.
* thread - runs a process on a new thread. Use if your process is long-running to avoid consuming all thread pool threads. Returns a channel, on which the process' return value is put.
* alts!! - takes a vector of channels and return the result of first successful operation and winning channel





## Testing

### Specs
validate the shape of the data

```clojure
(require [clojure.spec.alpha :as s])

(s/valid? my-predicate value)

(def and-spec (spec/and number? even?))	; and conditions
(map  #(spec/valid? and-spec %) ["str" 7 [] 2] ) ;;=> f f f t

(def or-spec (spec/or :is-number number?, :is-string string?))
(map #(spec/valid? or-spec %) [ '() false "abc" 42 ] )
; => f f t t

;; check collections
(map #(spec/valid? (spec/coll-of string?) %)
     [ [] ["a" "g" "u"] '("e" "r" "o") {:kun "aguero"}])
     ;; spec/cat tight specs sx followed by sy and sz IN A COLL

   ; check shape of a map
(def book-spec (spec/keys :req-un [:user/stock :user/qty :user/price]))

(spec/valid? book-spec {:stock "hdfc", :qty 10})  	    ; => F
(spec/valid? book-spec {:stock "hdfc", :qty 10, :price 1900 :extra "ok"}) ; => T

```

#### Global spec registry
JVM wide registry of specs any code can use

```clojure
(spec/def ::book (spec/keys :req-un [:user/stock :user/qty :user/price]))

   ;; if keys are registered with spec registry, individual values of the book will also be verified
(spec/def ::title string?)
(spec/def ::book (spec/keys :req-un [::title ::pages]))

```

## Java Interop

java.lang is auto-imported usually.

```clojure
(def rect (java.awt.Rectangle. 0 0 10 20))	; instantiate

 (.-width rect)	       ; public field
 (.area rect)	       ; public method
 (File/separator)      ; static field/method


	 ;; including packages
(import java.io.File)
(ns my-ns
    (:import '(java.io File Inputstream)))

```

## Advanced

### Threads

- clojure data structures are immutable
- DO NOT use def from within a thread.. global var trampling
- dynamic variables binding is also threadsafe

1. Promises

```clojure
;; dont need to define the task - just care about the result
(def my-promise (promise))
;; fulfill/resolve the promise
(println "Sometime in a world far far away")
(deliver my-promise 42)
	 ; syntactic sugar for (deref my-promise timeout timeout-value)
(println "The promised answer is" @my-promise)
```

2. Future - spawn a thread and return a promise for the result.

Uses a thread pool internally
```clojure
;; Futures kick off a thread and return a ticket. 
;; ticket#realized? boolean to know if task is done
;; (deref ticket timeout value) - return result. Block till done. Result is cached once computed
;; You can also specify an optional timeout and a value to return.

(let [result (future (Thread/sleep 2000)
              (+ 1 2))
      timeout 5000
      timeoutValue 5]
  (println "We're off")
  (println "Are we done?" (realized? result))
  (println "Want the result " (deref result timeout timeoutValue ))
  (println "Realized =>" (realized? result))
  (println "Cached result " @result)
  (println "Are we done?" (realized? result))

)
```

3. Delays
```clojure
;; DELAYS
;; doesn't execute immediately until forced
(let [notify (delay (println "Delayed!"))]
  (force notify)
)
;; same as deref a future

```

### State

Guidelines:

- if value is stable with some thread-specific changes => VAR
- SYNC if multiple mutable values need to be updated atomically => REF
- ASYNC if update functions have side effects like notifications  => AGENT
- SYNC if only ONE mutable value => ATOM

Atoms and Refs retry till they succeed


#### Ref Types

;; All clojure ref can be deref (no block)
;; support watchers and validators

1. Atoms

compare-and-set semantics. If value has been modified parallelly, the set operation is retried. Will not return till value is applied
```clojure

(def counter (atom 0))

 (swap! counter + 10)	; => 10 mutate!
 @counter	  	; deref yields 10

  - change notifications

(defn a-watcher
  [key identity old new]
  (println (format "event {Key=%s Id=%s %s=>%s}" key identity old new))
  )

(add-watch counter :observer a-watcher)
;=> #object[clojure.lang.Atom 0x1b1e3611 {:status :ready, :val 20}]

(swap! counter + 10)
; event {Key=:observer Id=clojure.lang.Atom@1b1e3611 20=>30}
;=> 30

```

2. Refs
If you want to update multiple things in a transaction
```clojure

(dosync
  (alter ref1 updater-1)
  (alter ref2 updater-2))
```
alter has similar behavior as atom.swap! Updates will be retried. dosync block is atomic.

3. Agents

- ASYNC agents queue their update function to be executed later and return
- No retries - hence functions CAN have side-effects.
- Agents will enter a failed state and raise the error on subsequent send calls

`(send agent1 updater-with-side-effects)`

## Meta
- read reads from stdin
- readstring needs a string argument
- eval evaluates data structures. Invokes the compiler to generate code. Avoid using within code - eval also doesnt see local let bindings

You can hang metadata off any object. Objects are equal even if they differ in metadata.

```clojure
(def books (with-meta obj {:fav true})

(let [books ^:fav {:title "Babloo"}]	; shorthand for above
   (meta books))  	  		; => {:fav true}

```
## Macros

syntax quoted `
escaped by ~
spliced in by ~@coll
quoted '
gensymbol `(let [myvar# ~n]..`

```clojure
(defmacro arith-if [n pos zero neg]
  `(cond
	(pos? ~n) ~pos
	(zero? ~n) ~zero
	:else ~neg))

(arith-if 10 (println "P") (println "0") (println "Z"))		' => P

; macroexpand-1 '(call-macro) for debugging macros
```



# TIPS
## Functions
* conj vs cons -   adds items to collections. However it does not return a seq i.e. preserves type. conj also can prepend or append (type-specific) unlike prepend always cons
* reduce -If seed isn't given, first element is used
* reify - creates a one-off implementation of a protocol
