title: How to schedule jobs in your Node.js service
date: 2015-08-07 16:12:44
tags:
- Node.js
- node
---
When dealing with large applications there are a lot of cases where you need some sort of scheduled task or cron job. Modifying these and redeploying them can be a pain as they are separate from your service/code. In addition, making sure the correct version of your software and the cron jobs are running on the right servers can add headache to troubleshooting.

Since your NodeJS service is an always running process, and you have things like setTimeout and setInterval at your disposal, setting these up in your NodeJS service can solve all these problems. Additionally, since you already have all your models set up, doing cleanup tasks on your databases or caches are a breeze.

While developing these tasks, I realized that if and when I cluster these services, I will run into an issue where multiple tasks will run at the same time, and this was not a good thing. I searched for a while for a solution but I could not find anything, so I decided to write it myself.

<!-- more -->

## is-master

https://www.npmjs.com/package/is-master
https://github.com/mattpker/node-is-master

is-master uses an existing mongoose singleton as its "cache", since most projects I work on already have this running. It is very simple to use; all you need to do is require the module, start the worker process, and then you can check if the process is the master.

```
var mongoose = require('../node_modules/mongoose');
var im = require('../is-master.js');

// Start the mongoose db connection
mongoose.connect('mongodb://127.0.0.1:27017/im', function(err) {
    if (err) {
        console.error('\x1b[31m', 'Could not connect to MongoDB!');
        throw (err);
    }
});

// Start the is-master worker
im.start();

// Check if this current process is the master
if (im.master) {
    // Do stuff here
}
```

This works by inserting each process into your mongodb database, and then whichever process is the oldest is considered the master. Each entry in the mongodb is set to expire if they are not updated within 2 minutes, this way if a process goes offline a new process will be promoted to master.

To setup scheduled tasks with this, you can use setTimeout or setInterval. I highly recommend using setTimeout, setInterval may seem like an easier choice, but it can cause issues if the previous interval got stuck so then you have 2 scheduled tasks running on top of each other.

Here is an example of using setTimeout to scheduled tasks:

```
function cronJob() {

    function done = setTimeout(function() {
        cronJob();
    }, 120000);

    if (im.master) {

        // Do your work here

        someCallbackFunction(function(err, results) {
            // Work with your callback database

            // When you are done make sure to call done
            done();
        })

    } else {
        done();
    }

}

im.on('connected', cronjob);
```

In the example above, the cronJob function will run when is-master emits that is has connected and is ready. The cronJob function will then check if it is master and if it is you can proceed with doing whatever work is necessary. You just need to make sure to call the done() function in every case that it finishes so that it will run again.

If you are looking for more of a cron syntax and style for setting these up, check out the package https://www.npmjs.com/package/cron. Here is another example using that package:

```
var CronJob = require('cron').CronJob;
var job = new CronJob('0 * * * * *', function() {
    if (im.master) {

        // Do your work here

    }
}, null, false, "America/Los_Angeles");

im.on('connected', job.start);
```

I hope this helps, let me know in the comments if you have any issues or suggestions.
