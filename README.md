# proper-lockfile [![Build Status](https://travis-ci.org/IndigoUnited/node-proper-lockfile.svg?branch=master)](https://travis-ci.org/IndigoUnited/node-proper-lockfile) [![Coverage Status](https://coveralls.io/repos/IndigoUnited/node-proper-lockfile/badge.png?branch=master)](https://coveralls.io/r/IndigoUnited/node-proper-lockfile?branch=master)

A inter-process and inter-machine lockfile utility that works on a local or network file system.


## Installation

`$ npm install proper-lockfile --save`


## Design

There are various ways to achieve [file locking](http://en.wikipedia.org/wiki/File_locking).

This library utilizes the `mkdir` strategy which works atomically on any kind of file system, even network based ones.
The lockfile path is based on the file path you are trying to lock by suffixing it with `.lock`.

When a lock is successfully acquired, the lockfile's `mtime` (modified time) is periodically updated to prevent staleness. This allows to effectively check if a lock is stale by checking its `mtime` against a stale threshold. If the update of the mtime fails several times, the lock might be compromised. The `mtime` is [supported](http://en.wikipedia.org/wiki/Comparison_of_file_systems) in almost every `filesystem`.


### Comparison

This library is similar to [lockfile](https://github.com/isaacs/lockfile) but the later has some drawbacks:

- It relies on `open` with `O_EXCL` flag which has problems in network file systems. `proper-lockfile` uses `mkdir` which doesn't have this issue.

> O_EXCL is broken on NFS file systems; programs which rely on it for performing locking tasks will contain a race condition.

- The lockfile staleness check is done via `ctime` (creation time) which is unsuitable for long running processes. `proper-lockfile` constantly updates lockfiles `mtime` to do proper staleness check.

- It does not check if the lockfile was compromised which can led to undesirable situations. `proper-lockfile` checks the lockfile when updating the `mtime`.


### Compromised

`proper-lockfile` does not detect cases in which:

- A `lockfile` is manually removed and someone else acquires the lock right after
- Different `stale`/`update` values are being used for the same file, possibly causing two locks to be acquired on the same file

`proper-lockfile` detects cases in which:

- Updates to the `lockfile` fail
- Updates take longer than expected, possibly causing the lock to became stale for a certain amount of time


As you see, the first two are a consequence of bad usage. Technically, it was possible to detect the first two but it would introduce complexity and eventual race conditions.


## Usage

### .lock(file, [options], [compromised], callback)

Tries to acquire a lock on `file`.

If the lock succeeds, a `release` function is provided that should be called when you want to release the lock.   
If the lock gets compromised, the `compromised` function will be called. The default `compromised` function is a simple `throw err` which will probably cause the process to die. Specify it to handle the way you desire.

Available options:

- `stale`: Duration in milliseconds in which the lock is considered stale, defaults to `10000` (minimum value is `5000`)
- `update`: The interval in milliseconds in which the lockfile's `mtime` will be updated, defaults to `stale/2` (minimum value is `1000`, maximum value is `stale/2`)
- `retries`: The number of retries or a [retry](https://www.npmjs.org/package/retry) options object, defaults to `0`
- `realpath`: Resolve symlinks using realpath, defaults to `true` (note that if `true`, the `file` must exist previously)
- `fs`: A custom fs to use, defaults to `graceful-fs`


```js
var lockfile = require('proper-lockfile');

lockfile.lock('some/file', function (err, release) {
    if (err) throw err;      // Lock failed

    // Do something while the file is locked

    // Call the provided release function when you're done
    release();

    // Note that you can optionally handle release errors
    // Though it's not mandatory since it will eventually stale
    /*release(function (err) {
        // At this point the lock was effectively released or an error
        // ocurred while removing it
        if (err) throw err;
    });*/
});
```


### .unlock(file, [options], [callback])

Releases a previously acquired lock on `file`.

Whenever possible you should use the `release` function instead (as exemplified above). Still there are cases in which its hard to keep a reference to it around code. In those cases `unlock()` might be handy.

The `callback` is optional because even if the removal of the lock failed, the lockfile's `mtime` will no longer be updated causing it to eventually stale.


Available options:

- `realpath`: Resolve symlinks using realpath, defaults to `true` (note that if `true`, the `file` must exist previously)
- `fs`: A custom fs to use, defaults to `graceful-fs`


```js
var lockfile = require('proper-lockfile');

lockfile.lock('some/file', function (err) {
    if (err) throw err;

    // Later..
    lockfile.unlock('some/file');

    // or..
    /*lockfile.unlock('some/file', function (err) {
        // At this point the lock was effectively released or an error
        // ocurred while removing it
        if (err) throw err;
    });*/
});
```


### .lockSync(file, [options], [compromised])

Sync version of `.lock()`.   
Returns the `release` function or throws on error.


### .unlockSync(file, [options])

Sync version of `.unlock()`.   
Throws on error.


## Graceful exit

`proper-lockfile` automatically remove locks if the process exists. Though, `SIGINT` and `SIGTERM` signals
are handled differently by `nodejs` in the sense that they do not fire a `exit` event on the `process`.
To avoid this common issue that `CLI` developers have, please do the following:

```js
// Map SIGINT & SIGTERM to process exit
// so that lockfile removes the lockfile automatically
process
.once('SIGINT', function () {
    process.exit(1);
})
.once('SIGTERM', function () {
    process.exit(1);
});
```


## Tests

Simply run the test suite with `$ npm test`

The test suite is very extensive. There's even a stress test to guarantee exclusiveness of locks.


## License

Released under the [MIT License](http://www.opensource.org/licenses/mit-license.php).
