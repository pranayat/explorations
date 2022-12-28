# bundle exports into one file
index.js
```
'use strict';

module.exports = require('./lib/queue');
module.exports.Job = require('./lib/job');
module.exports.utils = require('./lib/utils');
```

I commonly see this way of bundling module exports into an index file, the index file is then simply imported to make all these modules availiable wherever the index file is imported.

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
