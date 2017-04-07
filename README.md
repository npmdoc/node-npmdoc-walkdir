# api documentation for  [walkdir (v0.0.11)](http://github.com/soldair/node-walkdir)  [![npm package](https://img.shields.io/npm/v/npmdoc-walkdir.svg?style=flat-square)](https://www.npmjs.org/package/npmdoc-walkdir) [![travis-ci.org build-status](https://api.travis-ci.org/npmdoc/node-npmdoc-walkdir.svg)](https://travis-ci.org/npmdoc/node-npmdoc-walkdir)
#### Find files simply. Walks a directory tree emitting events based on what it finds. Presents a familiar callback/emitter/a+sync interface. Walk a tree of any depth.

[![NPM](https://nodei.co/npm/walkdir.png?downloads=true)](https://www.npmjs.com/package/walkdir)

[![apidoc](https://npmdoc.github.io/node-npmdoc-walkdir/build/screenCapture.buildNpmdoc.browser._2Fhome_2Ftravis_2Fbuild_2Fnpmdoc_2Fnode-npmdoc-walkdir_2Ftmp_2Fbuild_2Fapidoc.html.png)](https://npmdoc.github.io/node-npmdoc-walkdir/build/apidoc.html)

![npmPackageListing](https://npmdoc.github.io/node-npmdoc-walkdir/build/screenCapture.npmPackageListing.svg)

![npmPackageDependencyTree](https://npmdoc.github.io/node-npmdoc-walkdir/build/screenCapture.npmPackageDependencyTree.svg)



# package.json

```json

{
    "author": {
        "name": "Ryan Day",
        "email": "soldair@gmail.com"
    },
    "bugs": {
        "url": "https://github.com/soldair/node-walkdir/issues"
    },
    "contributors": [
        {
            "name": "tjfontaine"
        }
    ],
    "dependencies": {},
    "description": "Find files simply. Walks a directory tree emitting events based on what it finds. Presents a familiar callback/emitter/a+sync interface. Walk a tree of any depth.",
    "devDependencies": {
        "tape": "^4.0.0"
    },
    "directories": {},
    "dist": {
        "shasum": "a16d025eb931bd03b52f308caed0f40fcebe9532",
        "tarball": "https://registry.npmjs.org/walkdir/-/walkdir-0.0.11.tgz"
    },
    "engines": {
        "node": ">=0.6.0"
    },
    "gitHead": "63bd65dc8fba903634f84f818dad3e7f39032927",
    "homepage": "http://github.com/soldair/node-walkdir",
    "keywords": [
        "find",
        "walk",
        "tree",
        "files",
        "fs"
    ],
    "license": "MIT",
    "main": "./walkdir.js",
    "maintainers": [
        {
            "name": "soldair",
            "email": "soldair@gmail.com"
        }
    ],
    "name": "walkdir",
    "optionalDependencies": {},
    "readme": "ERROR: No README data found!",
    "repository": {
        "type": "git",
        "url": "git://github.com/soldair/node-walkdir.git"
    },
    "scripts": {
        "test": "tape test/*.js"
    },
    "version": "0.0.11"
}
```



# <a name="apidoc.tableOfContents"></a>[table of contents](#apidoc.tableOfContents)

#### [module walkdir](#apidoc.module.walkdir)
1.  [function <span class="apidocSignatureSpan">walkdir.</span>find (path, options, cb)](#apidoc.element.walkdir.find)
1.  [function <span class="apidocSignatureSpan">walkdir.</span>sync (path, options, cb)](#apidoc.element.walkdir.sync)
1.  [function <span class="apidocSignatureSpan">walkdir.</span>walk (path, options, cb)](#apidoc.element.walkdir.walk)



# <a name="apidoc.module.walkdir"></a>[module walkdir](#apidoc.module.walkdir)

#### <a name="apidoc.element.walkdir.find"></a>[function <span class="apidocSignatureSpan">walkdir.</span>find (path, options, cb)](#apidoc.element.walkdir.find)
- description and source-code
```javascript
function walkdir(path, options, cb){

  if(typeof options == 'function') cb = options;

  options = options || {};

  var fs = options.fs || _fs;

  var emitter = new EventEmitter(),
  dontTraverse = [],
  allPaths = (options.return_object?{}:[]),
  resolved = false,
  inos = {},
  stop = 0,
  pause = null,
  ended = 0,
  jobs=0,
  job = function(value) {
    jobs += value;
    if(value < 1 && !tick) {
      tick = 1;
      process.nextTick(function(){
        tick = 0;
        if(jobs <= 0 && !ended) {
          ended = 1;
          emitter.emit('end');
        }
      });
    }
  }, tick = 0;

  emitter.ignore = function(path){
    if(Array.isArray(path)) dontTraverse.push.apply(dontTraverse,path)
    else dontTraverse.push(path)
    return this
  }

  //mapping is stat functions to event names.	
  var statIs = [['isFile','file'],['isDirectory','directory'],['isSymbolicLink','link'],['isSocket','socket'],['isFIFO','fifo'],['
isBlockDevice','blockdevice'],['isCharacterDevice','characterdevice']];

  var statter = function (path,first,depth) {
    job(1);
    var statAction = function fn(err,stat,data) {

      job(-1);
      if(stop) return;

      // in sync mode i found that node will sometimes return a null stat and no error =(
      // this is reproduceable in file descriptors that no longer exist from this process
      // after a readdir on /proc/3321/task/3321/ for example. Where 3321 is this pid
      // node @ v0.6.10
      if(err || !stat) {
        emitter.emit('fail',path,err);
        return;
      }


      //if i have evented this inode already dont again.
      var fileName = _path.basename(path);
      var fileKey = stat.dev + '-' + stat.ino + '-' + fileName;
      if(inos[fileKey] && stat.ino) return;
      inos[fileKey] = 1;

      if (first && stat.isDirectory()) {
        emitter.emit('targetdirectory',path,stat,depth);
        return;
      }

      emitter.emit('path', path, stat,depth);

      var i,name;

      for(var j=0,k=statIs.length;j<k;j++) {
        if(stat[statIs[j][0]]()) {
          emitter.emit(statIs[j][1],path,stat,depth);
          break;
        }
      }
    };

    if(options.sync) {
      var stat,ex;
      try{
        stat = fs.lstatSync(path);
      } catch (e) {
        ex = e;
      }

      statAction(ex,stat);
    } else {
        fs.lstat(path,statAction);
    }
  },readdir = function(path,stat,depth){
    if(!resolved) {
      path = _path.resolve(path);
      resolved = 1;
    }

    if(options.max_depth && depth >= options.max_depth){
      emitter.emit('maxdepth',path,stat,depth);
      return;
    }

    if(dontTraverse.length){
      for(var i=0;i<dontTraverse.length;++i){
        if(dontTraverse[i] == path) {
          dontTraverse.splice(i,1)
          return;
        }
      }
    }

    job(1);
    var readdirAction = function(err,files) {
      job(-1);
      if (err || !files) {
        //permissions error or invalid files
        emitter.emit('fail',path,err);
        return;
      }

      if(!files.length) {
        // empty directory event.
        emitter.emit('empty',path,stat,depth);
        return;
      }

      if(path == sep) path='';
      for(var i=0,j=files.length;i<j;i++){
        statter(path+sep+files[i],false,(depth||0)+1);
      }

    };


    //use same pattern for sync as async api
    if(options.sync) {
      var e,files;
      try {
          files = fs.readdirSync(path);
      } catch (e) { }

      readdirAction(e,files);
    } else {
      fs.readdir(path,readdirAction);
    }
  };

  if (options.follow_symlinks) {
    var linkAction = function(err,path,depth){
      job(-1);
      //TODO should fail event here on error?
      statter(path,false,depth);
    };

    emitter.on('link',function(path,stat,depth){
      job(1);
      if(options.sync) {
        var lpath,ex;
        try {
          lpath = fs.readlinkSync(path);
        } catch(e) {
          ex = e;
        }
        linkAction(ex,_path.resolve(_path.dirname(path),lpath),depth);

      } else {
        fs.readlink(path,function(err,lpath){
          linkAc ...
```
- example usage
```shell
n/a
```

#### <a name="apidoc.element.walkdir.sync"></a>[function <span class="apidocSignatureSpan">walkdir.</span>sync (path, options, cb)](#apidoc.element.walkdir.sync)
- description and source-code
```javascript
sync = function (path, options, cb){
  if(typeof options == 'function') cb = options;
  options = options || {};
  options.sync = true;
  return walkdir(path,options,cb);

}
```
- example usage
```shell
...
emitter.on('file',function(filename,stat){
  console.log('file from emitter: ', filename);
});


//sync with callback

walk.sync('../',function(path,stat){
  console.log('found sync:',path);
});

//sync just need paths

var paths = walk.sync('../');
console.log('found paths sync: ',paths);
...
```

#### <a name="apidoc.element.walkdir.walk"></a>[function <span class="apidocSignatureSpan">walkdir.</span>walk (path, options, cb)](#apidoc.element.walkdir.walk)
- description and source-code
```javascript
function walkdir(path, options, cb){

  if(typeof options == 'function') cb = options;

  options = options || {};

  var fs = options.fs || _fs;

  var emitter = new EventEmitter(),
  dontTraverse = [],
  allPaths = (options.return_object?{}:[]),
  resolved = false,
  inos = {},
  stop = 0,
  pause = null,
  ended = 0,
  jobs=0,
  job = function(value) {
    jobs += value;
    if(value < 1 && !tick) {
      tick = 1;
      process.nextTick(function(){
        tick = 0;
        if(jobs <= 0 && !ended) {
          ended = 1;
          emitter.emit('end');
        }
      });
    }
  }, tick = 0;

  emitter.ignore = function(path){
    if(Array.isArray(path)) dontTraverse.push.apply(dontTraverse,path)
    else dontTraverse.push(path)
    return this
  }

  //mapping is stat functions to event names.	
  var statIs = [['isFile','file'],['isDirectory','directory'],['isSymbolicLink','link'],['isSocket','socket'],['isFIFO','fifo'],['
isBlockDevice','blockdevice'],['isCharacterDevice','characterdevice']];

  var statter = function (path,first,depth) {
    job(1);
    var statAction = function fn(err,stat,data) {

      job(-1);
      if(stop) return;

      // in sync mode i found that node will sometimes return a null stat and no error =(
      // this is reproduceable in file descriptors that no longer exist from this process
      // after a readdir on /proc/3321/task/3321/ for example. Where 3321 is this pid
      // node @ v0.6.10
      if(err || !stat) {
        emitter.emit('fail',path,err);
        return;
      }


      //if i have evented this inode already dont again.
      var fileName = _path.basename(path);
      var fileKey = stat.dev + '-' + stat.ino + '-' + fileName;
      if(inos[fileKey] && stat.ino) return;
      inos[fileKey] = 1;

      if (first && stat.isDirectory()) {
        emitter.emit('targetdirectory',path,stat,depth);
        return;
      }

      emitter.emit('path', path, stat,depth);

      var i,name;

      for(var j=0,k=statIs.length;j<k;j++) {
        if(stat[statIs[j][0]]()) {
          emitter.emit(statIs[j][1],path,stat,depth);
          break;
        }
      }
    };

    if(options.sync) {
      var stat,ex;
      try{
        stat = fs.lstatSync(path);
      } catch (e) {
        ex = e;
      }

      statAction(ex,stat);
    } else {
        fs.lstat(path,statAction);
    }
  },readdir = function(path,stat,depth){
    if(!resolved) {
      path = _path.resolve(path);
      resolved = 1;
    }

    if(options.max_depth && depth >= options.max_depth){
      emitter.emit('maxdepth',path,stat,depth);
      return;
    }

    if(dontTraverse.length){
      for(var i=0;i<dontTraverse.length;++i){
        if(dontTraverse[i] == path) {
          dontTraverse.splice(i,1)
          return;
        }
      }
    }

    job(1);
    var readdirAction = function(err,files) {
      job(-1);
      if (err || !files) {
        //permissions error or invalid files
        emitter.emit('fail',path,err);
        return;
      }

      if(!files.length) {
        // empty directory event.
        emitter.emit('empty',path,stat,depth);
        return;
      }

      if(path == sep) path='';
      for(var i=0,j=files.length;i<j;i++){
        statter(path+sep+files[i],false,(depth||0)+1);
      }

    };


    //use same pattern for sync as async api
    if(options.sync) {
      var e,files;
      try {
          files = fs.readdirSync(path);
      } catch (e) { }

      readdirAction(e,files);
    } else {
      fs.readdir(path,readdirAction);
    }
  };

  if (options.follow_symlinks) {
    var linkAction = function(err,path,depth){
      job(-1);
      //TODO should fail event here on error?
      statter(path,false,depth);
    };

    emitter.on('link',function(path,stat,depth){
      job(1);
      if(options.sync) {
        var lpath,ex;
        try {
          lpath = fs.readlinkSync(path);
        } catch(e) {
          ex = e;
        }
        linkAction(ex,_path.resolve(_path.dirname(path),lpath),depth);

      } else {
        fs.readlink(path,function(err,lpath){
          linkAc ...
```
- example usage
```shell
n/a
```



# misc
- this document was created with [utility2](https://github.com/kaizhu256/node-utility2)
