# bundle exports into one file
index.js
```
'use strict';

module.exports = require('./lib/queue');
module.exports.Job = require('./lib/job');
module.exports.utils = require('./lib/utils');
```

Bundling module exports into an index file, the index file is then simply imported to make all these modules availiable wherever the index file is imported. Watch our for circular deps.

# hanlde bad constructor function invocations
lib/queue.js
```
const Queue = function Queue(name, url, opts) {
  if (!(this instanceof Queue)) {
    return new Queue(name, url, opts);
  }
  
  this.name = name;
  this.token = uuid.v4();
}
```
this `!(this.instanceof Queue))` handles the case where `Queue` may be accidentally called without the new keyword.

# storing this in a closure

```
var sampleObject = function() {
 this.foo = 123;
}

sampleObject.prototype.getFoo = function() {
 var nested = function() {
  // this is global obj as this function does not belong to any object, the closure hence refers to the global object
  return this.foo;
 }
 return nested();
}

var test = new sampleObject();
test.getFoo() // returns undefined
```

```
var sampleObject = function() {
 this.foo = 123;
}

sampleObject.prototype.getFoo = function() {
 // this is the sampleObject parent object that owns this getFoo method
 var _this = this;
 var nested = function() {
  // store it inside a closure here
  return _this.foo;
 }
 return nested();
}

var test = new sampleObject();
```

can also achieve this by binding the nested function to the this value
```
sampleObject.prototype.getFoo = function() {
 var nested = function() {
  return this.foo;
 }.bind(this); // here this is the parent sampleObject object that owns the getFoo method
 return nested();
}
```
OR by using call
```
SampleObject.prototype.getFoo = function() {
 var nested = function() {
  return this.foo;
 };
 return nested.call(this);
}
```
