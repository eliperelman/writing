Ever wondered what your options are when it comes to storing data on the client in your web application? Browser support, API features, storage size - like so many DOM features, it can be difficult to know what's available, what browsers support it, and which option is the best fit for your needs. We'll take a look at the basics of each major type of client-side storage in this post, look at some of the pros and cons, and discuss some guidelines to keep in mind as you bring these tools to bear in your applications.

#First - Let's Fill in the Gaps
For a long time, the web limped along with one primary form of client side storage: cookies. Browsers typically limited cookies to a size of 4KB, allowing 20 per domain. Given today's standards, that's some tiny storage space - and it comes at the (potentially high) price of the cookies being sent with each HTTP request.

Thankfully, browsers didn't stop there:

* The ["Web Storage" spec](http://dev.w3.org/html5/webstorage/) – introduced in 2009 – provided APIs for `sessionStorage` & `localStorage`. The name `localStorage` has become nearly synonymous in usage with "Web Storage" - and you will often hear it used instead. Most implementations give you between 2MB and 5MB of key/value storage (with exceptions, of course).
* [Indexed DB](https://developer.mozilla.org/en-US/docs/IndexedDB) (called an "indexed table system") is a key/value store – not a relational database like Web SQL. You can persist JavaScript objects to an "object store" - but it does not require you to define a schema for your object structure (though you *do* define a key). In addition, you can define indexes to improve performance.
* Chrome 27+, Opera 15+ and Blackberry 10 have a [FileSystem API](http://www.w3.org/TR/file-system-api/) as well. While the opportunities to utilize the FileSystem API won't be as common as the other scenarios **yet**, this option is especially useful when local storage and Indexed DB fall short of your needs.
* If you're writing hybrid mobile applications – using a framework like [Apache Cordova](http://www.icenium.com/blog/icenium-team-blog/2013/03/26/demystifying-apache-cordova-and-phonegap) (a.k.a. - "PhoneGap") – in addition to having access to features like local storage (assuming the mobile webview supports it) you have a [File API](http://cordova.apache.org/docs/en/3.0.0/cordova_file_file.md.html#File) that allows you to read & write from the device's file system which follows the W3C spec that the FileSytsem API is based on.
* Though it's currently deprecated (as of 2010), you may still come across applications making use of a [Web SQL database](http://www.w3.org/TR/webdatabase/). Web SQL - supported primarily in Webkit browsers - provided a means to manipulate client-side data storage using a variant of SQL. It comes with all of the power and headaches of working with a relational database - sporting larger size limitations than local storage (Safari, for example, let Web SQL DBs get up to 50MB). The W3C abandoned Web SQL databases in favor of the [Indexed DB](https://developer.mozilla.org/en-US/docs/IndexedDB) specification.

#Web (Local & Session) Storage
Of the storage options we'll cover, Web Storage is the most widely supported option available to you as of today. The Web Storage spec gives us `sessionStorage` and `localStorage` – which share the same API (both implement the `Storage` prototype). `sessionStorage` is used to store data for the duration of the current browser session. It will persist between page reloads, but will be cleared when the browser is closed. Data stored in `localStorage` is persistent. It will only be cleared when the user specifically clears it via browser tools, or when your code removes items previously stored, etc.

Are there any real advantages of using `sessionStorage` over `localStorage`? In my opinion, no. The typical use-case for `sessionStorage` would be when you need to persist data in a form, for example, because the user might accidentally refresh the page. The problem, though, is that the user might also accidentally *close the browser*, too. If you were relying on `sessionStorage` in case of a refresh, and the user closes the browser instead…Oops. `localStorage` handles both issues well. Bottom line: you will probably find `localStorage` to be the best option of the two 99.9% of the time (*insert anti-well-actually-troll disclaimer here* :-)).

Let's look at an example.
<iframe width="100%" height="300" src="http://jsfiddle.net/ifandelse/NPKUJ/embedded/result,js,html,css/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

Now - the JavaScript:

	// cache our selectors
	var $name = $("#fullName");
	var $loc = $("#location");
	var store = {
		// store our input values as members of an object that gets
		// serialized, since local storage stores *strings*
    	saveToStorage: function() {
        	localStorage.setItem("example", JSON.stringify(this.getInputValues()));
	    },
	    // a bit naive, but we parse the JSON into an object
	    // and if we get something, we set our input values
    	loadFromStorage: function() {
        	var store = JSON.parse(localStorage.getItem("example"));
	        if(store) {
    	        $name.val(store.fullName);
        	    $loc.val(store.location); 
	        }
    	},
	    clearFields: function() {
    	    $name.val("");
        	$loc.val("");
	    },
    	clearStorage: function() {
        	localStorage.clear();
	    },
    	getInputValues: function() {
        	return {
            	fullName: $name.val(),
            	location: $loc.val()
	       }
    	}
	};
	// hook up event handlers
	$("#setStorage").on("click", $.proxy(store.saveToStorage, store));
	$("#loadStorage").on("click", $.proxy(store.loadFromStorage, store));
	$("#clearFields").on("click", store.clearFields);
	$("#clearStorage").on("click", store.clearStorage);
	
You can see from the above code that the `localStorage` API is quite simple. In our `saveToStorage` method we're calling `localStorage.setItem` and passing the key and value as arguments – saving our serialized object under the key "example" (we have to serialize our object because `localStorage` only stores string values.) Then, in our `loadFromStorage` method, we call `localStorage.getItem`, passing the key of the item we want. Of course, we get a string back, which we need to parse into an object. Further down, you'll notice we're calling `localStorage.clear`, which will remove *all* `localStorage` data for this origin. It's possible to remove specific keys, however, by calling `localStorage.removeItem(key)`.
	
##Important Details About Local Storage
* `localStorage` stores *strings*. This means that you will need to serialize your objects before storing them if you intend to persist something other than a string.
* The [Same Origin](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Same_origin_policy_for_JavaScript) policy rules are in effect. This means you cannot access the local storage from another domain. Keep in mind that same-origin rules apply even to different ports or [schemes](http://en.wikipedia.org/wiki/URI_scheme) from the *same* domain. In other words, http://my.domain.com *cannot* access local storage for https://my.domain.com AND http://my.domain.com:80 *cannot* access http://my.domain.com:8088. 
* Careful - *you can run out of space*. Many browsers allow for 2.5-5MB of local storage per domain. Bearing in mind that JavaScript strings are UTF-16, and thus take 2 bytes per character, a 5MB limit means you can store 2.5MB of text. [Here](http://dev-test.nemikor.com/web-storage/support-test/) is an informative test runner that will show you the amount of text data you can store per browser. From what I've researched, Opera and Firefox are the only browsers that currently allow you increase the `localStorage` size limit (though I believe Chrome allows this for apps downloaded via the Chrome Web Store).
* The [Storage Event](http://www.w3.org/TR/2011/WD-webstorage-20110208/#event-storage) isn't quite what you'd expect. Most browsers only fire the `storage` event if a *different* window changes local storage (IE 10 will fire the event for same-window changes, but Chrome and Safari do not, for example). So don't expect the event to be fired for changes within the same window.
* Simplicity comes with trade-offs:
	* It's easy to store and retrieve keys, but you're on your own when it comes to searching/filtering data.
	* The need to proactively serialize, and the fact that the API is synchronous *might* cause you performance headaches. I've never had an issue here, though.
	* Since you aren't required to define a schema, validating the stored data is up to you. 
	
Overall - `localStorage` can be a reliable workhorse - with older browser support often made possible via polyfills.  I've used [ampllify.store](http://amplifyjs.com/api/store/) for API normalization and fallbacks in many apps, for example. The **good** news is that current support looks good:

![](./WebStorageSupport.png)
*(taken from [http://caniuse.com/#feat=namevalue-storage](http://caniuse.com/#feat=namevalue-storage))*

The **bad** news is – as with ANY of these client-side storage options – you should *never* assume that the data will be truly persistent (after all, the user can nuke it from orbit if they desire), and you should be careful as to the kinds of data you store (it's not hard for any user to view stored data via browser tools).

#Indexed DB
The conventional wisdom is that `localStorage` works well for smaller amounts of data (especially when there's no heavy searching & filtering). When you need something more, that will most likely be IndexedDB (assuming you have browser support). Like `localStorage`, IndexedDB allows you to store data persistently on the client, and it follows the [same-origin policy](http://www.w3.org/Security/wiki/Same_Origin_Policy). Here are a few defining characteristics of IndexedDB:

* It's a key/value store. Values can be objects, and keys can be a property of those objects.
* It uses a transaction model - so each operation is tied to a transaction, including reads. (There are three transaction modes: `read-only`, `read/write`, and `versionchange`)
* You can create indexes (using any of the properties on the stored objects) for searching and sorting.
* The spec defines an asynchronous *and* synchronous API, though no major browser has implemented the synchronous version yet.

I put together another [jsFiddle](http://jsfiddle.net/ifandelse/f5mNW/) to demonstrate some of IndexedDB's features. In our fiddle, we have a `storageContainer` object that very lightly wraps IndexedDB. I've intentionally avoided bringing a UI framework into this example (though I did pull in an [event emitter](https://github.com/postaljs/monologue.js) and [message bus](https://github.com/postaljs/postal.js)). I think it's important to see IndexedDB's API - once you get the hang of it, it starts to make sense. However, it can be awkward even *after* you've gotten used to it.

Many API calls return a "request" (an [IDBRequest object](https://developer.mozilla.org/en-US/docs/Web/API/IDBRequest?redirectlocale=en-US&redirectslug=IndexedDB%2FIDBRequest)) - which typically has an `onsuccess` and `onerror` member which you can assign a handler to (and some requests have addition handler hooks besides these two). It might feel like you're stuck half way between pure-event emitting and promises. I personally prefer event-emitting style APIs, so I brought in a message bus ([postal.js](https://github.com/postaljs/postal.js)) to act as the communications bridge between the hand-rolled view models and our `storageContainer` instance. Our `storageContainer` instance listens for a couple of different messages and reacts appropriately when one arrives. If any of this is unfamiliar to you, don't worry. The code in the fiddle itself is focused only on the `storageContainer`.

You can take the example for a test drive here if your browser supports IndexedDB:
<iframe width="100%" height="300" src="http://jsfiddle.net/ifandelse/f5mNW/embedded/result,js,html,css/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

Let's break down some of the snippets a piece at a time:

##Opening and/or Creating a Database
The `init` function of our `storageContainer` is as follows:

	var storageContainer = {
		//other members...
		init: function () {
			// open our DB and create object stores if it's the
			// first time this DB has been created on this client
			var self = this;
			var request = indexedDB.open("BandPrefs", version);
			request.onerror = jimIsAnIdiot;
			request.onsuccess = function (e) {
				self.db = e.target.result;
				if (firstTime) {
					self.loadSeedData();
				}
				if (!firstTime && self.db.objectStoreNames.contains('prefs')) {
					self.loadExistingPrefs();
				}
			};
			// We can only create Object stores in a versionchange transaction.
			// We know that if this gets called, it's the first time
			// that the code has run on this client
			request.onupgradeneeded = function (e) {
				var prefStore;
				firstTime = true;
				self.db = e.target.result;
				prefStore = self.db.createObjectStore("prefs", {
					keyPath: "_id"
				});
				prefStore.createIndex("fullName", "fullName", {
					unique: false
				});
			};
		}
	};
 
One of the first steps we take in `init` is the call to `indexedDB.open`. The first argument is the name of the database, and the second is the version. This is one of the confusing aspects of the API if you're new to IndexedDB in general. Due to the transactional nature of IndexedDB, you can only modify the schema (e.g. - create object stores and/or indexes) in a version upgrade transaction. If the database has never been created before on the client, OR if you provide a version number higher than what exists, then you get the chance to modify the schema as part of the version upgrade by utilizing the `onupgradeneeded` hook. 

You can see that inside our `onupgradeneeded` handler, we're creating an object store named 'prefs' (for storing band/artist preferences). The key for the prefs object store will be the `_id` member of the object passed into the store. We're also creating a non-unique index on the prefs object store called "fullName" and the key of the index will be the `fullName` property of the pref object. Having this index will make it easy to retrieve a list of band preferences using the name of the person who liked the band - rather than needing to know the key of each one. It's also important to note that `onupgradeneeded` will fire *before* `onsuccess`.
 
##Storing Data

The `storePref` method on our `storageContainer` looks as follows:

	var storageContainer = {
		// stores one band preference record
		storePref: function (pref) {
		    var self = this;
		    var transaction = self.db.transaction(["prefs"], "readwrite");
		    transaction.oncomplete = function (event) {
		        self.emit("transaction.oncomplete");
		    };
		    transaction.onerror = jimIsAnIdiot; //self-deprecation FTW
		    var prefs = transaction.objectStore("prefs");
		    var addRequest = prefs.add(pref);
		    addRequest.onsuccess = function (event) {
		        self.emit("cursor.onsuccess", pref);
		    };
		}
		// other members, etc.
	};

As a user enters their name and a band/artist they like, the viewmodel for the form publishes a message that our `storageContainer` listens for. When that message arrives, the `storePref` method is invoked. Again, you'll notice we're dealing with transactions. We start a transaction via `self.db.transaction`. The first argument is an array of object store names that this transaction will involve. We only have one - "prefs". The second argument is the transaction mode. We're writing data, so we use `readwrite`. Notice that our transaction request has `oncomplete` and `onerror` hooks, but the `add` request also has `onsuccess` (and an `onerror` that I'm not using in this example). Our new pref object gets added to the store when we call `prefs.add(pref)` - that's all it takes.

## Retrieving Data
Our `storageContainer` supports two ways to retrieve data - everything, or all the preferences for a given name. The two methods involved are `loadExistingPrefs` and `filterPrefs`:

	var storageContainer = {
		// looks for existing band preferences and emits an
        // event containing the list if any exist.
        loadExistingPrefs: function () {
            var prefs = [],
                self = this;
            self.db.transaction(["prefs"])
                .objectStore("prefs")
                .openCursor().onsuccess = getIterator(self, prefs, "prefs.exist");
        },
        // filters using an index based on the name of the person
        // and emits an event containing the results
        filterPrefs: function (name) {
            var prefs = [],
                self = this;
            self.db.transaction(["prefs"])
                .objectStore("prefs")
                .index("fullName")
                .openCursor(IDBKeyRange.only(name)).onsuccess = getIterator(self, prefs, "prefs.filtered");
        }
        // other members, etc.
	};
	
For the most part, what these two methods do is *nearly* identical. We start a transaction (with `self.db.transaction(["prefs"])`). Then we indicate which object store we're going to work with (via `.objectStore("prefs")`). This is where the two diverge. On each one, we're opening a cursor that will allow us to iterate over results. However, the `filterPrefs` method (which allows us to find the band/artist preferences for a specific person) passes `IDBKeyRange.only(name)` to the cursor. This tells IndexedDB that we only want the items in this index that have the specified key. (The kinds of key constraints possible are very helpful - see [this](https://developer.mozilla.org/en-US/docs/IndexedDB/Using_IndexedDB#Specifying_the_range_and_direction_of_cursors) for more detail.) 

Both methods are using the same function to generate an onsuccess handler. (The `onsuccess` handler will be called for each record the cursor returns). The function returned from `getIterator` will be our `onsuccess` handler:

	var getIterator = function(container, prefs, topic) {
        return function (event) {
            var cursor = event.target.result;
            if (cursor) {
                prefs.push({
                    fullName: cursor.value.fullName,
                    bandName: cursor.value.bandName
                });
                cursor.
                continue ();
            } else {
                container.emit(topic, prefs);
            }
        };
    };
    
As long as our cursor is truthy, we continue compiling our results (by adding them to the `prefs` array). Once we've iterated over all the rows, the cursor will be falsy. At this point, we emit an event with the results we've compiled.

This example only shows you a few features of IndexedDB - but it shows enough to demonstrate not only the powerful capabilities, but (IMO) the need to wrap and abstract the API away from your application logic. The limited functionality in our `storageContainer` still has a decent amount of boilerplate code. Abstracting it into a separate infrastrucutre-focused component will help keep your application from buckling under the weight of confusing & noisy boilerplate.

##Support
Browser support for IndexedDB is improving - but it's still very new. Safari still doesn't have support, nor does <= IE 9.

![](./IndexedDBSupport.png)
*(taken from [http://caniuse.com/#feat=indexeddb](http://caniuse.com/#feat=indexeddb))*

It's possible to enable IndexedDB on browsers that don't support if they support Web SQL by using the [IndexedDB Polyfill](https://github.com/axemclion/IndexedDBShim).

Size limits for IndexedDB vary by browser:

* Firefox currently allows a site to use up to 50MB, after which it will prompt the user for permission. Firefox Mobile defaults to 5MB, and prompts for permission after that.
* IE 10 will allow up to a whopping 250MB per domain (but that appears to include both File API *and* IndexedDB size quotas).
* I haven't been able to find a hard limit in Chrome, though [this developer](http://productforums.google.com/forum/#!msg/chrome/-fgX6UPY-Q4/itG2KEHyZv4J) managed to get up to 400MB while testing the size limits before Chrome crashed. The [API docs](https://developers.google.com/chrome/whitepapers/storage) indicate that you have to request a size for Persistent Storage (which the user must approve), and that it can get "as large as the available space on the hard drive. It has no fixed pool of storage" (Chrome also has quote rules around "unlimited" and "temporary" storage).

#Using the File System
It's still early in the game for browser support of writing to the client file system. The only native support currently exists in Chrome 27+, Opera 15+ and Blackberry 10. [Burke Holland](https://twitter.com/burkeholland/) mentioned [idb.filesystem.js](https://github.com/ebidel/idb.filesystem.js), an IndexedDB-based polyfill that emulates FileSystem API support browsers that don't currently have it but *do* support IndexedDB. We'll focus on the native API in our example, but you could easily adapt it to use the polyfill if you wanted to have a fun evening project.

If you've worked with file system APIs in any language, then the FileSystem API won't seem foreign. In [this jsFiddle example](http://jsfiddle.net/ifandelse/T7Tpz/), I've adapted most of the earlier IndexedDB example to use FileSystem instead. 

You can take the example for a test drive here if you're in Chrome or Opera:
<iframe width="100%" height="300" src="http://jsfiddle.net/ifandelse/T7Tpz/embedded/result,js,html,css/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

## Requesting Access to the File System
Our FileSystem-based `storageContainer` has an init method that sets up initial access to the client file system. Here's the relevant snippet:

	var module = $.extend({
        fs: undefined,
        init: function () {
            var self = this;
            // we request 5MB of storage, though we won't use it all here
            navigator.webkitPersistentStorage.requestQuota(
                5 * 1024 * 1024,
                function (grantedBytes) {
                    // we request a handle to the file system
                    window.webkitRequestFileSystem(
                        PERSISTENT,
                        grantedBytes,
                        // the fs arg in this callback is our filesystem ref
                        function (fs) {
                            self.fs = fs;
                            self.loadExistingPrefs(function(contents) {
                                if(contents.length) {
                                    console.log("We already have content we'll use");
                                    self.emit("prefs.exist", contents);
                                } else {
                                    console.log("We need to load seed data....");
                                    self.loadSeedData(function() {
                                        self.loadExistingPrefs(); 
                                    });   
                                }
                            });
                        },
                        errorHandler);
                },
                errorHandler
            );
        }
        // other members, etc.
	}, Monologue.prototype);
	
First, we ask the user for permission to store persistent data (via `navigator.webkitPersistentStorage.requestQuota`). It's worth noting that most examples you'll find on the web today will use `window.webkitStorageInfo.requestQuota` for this. However, that has recently been deprecated in favor of the one used above. The first argument is the size we're requesting (5MB). Assuming the user is OK with this, our success handler (the second argument) will be invoked, with the size the user approved passed in as the callback arg.

Once we have the user's permission and the size they approved, we call `window.webkitRequestFileSystem` to request access to the file system. The first arg is the type of storage: `PERSISTENT` or `TEMPORARY`. `TEMPORARY` storage can be cleaned by the browser at any time, where `PERSISTENT` will only be cleared if the user explicitly does so. The third arg to `webkitRequestFileSystem` is the success callback. You can see that it receives a handle to the FileSystem via the `fs` argument. Inside our callback we're storing a reference to `fs` on our `storageContainer` and then we attempt to load any existing data for our example app. If no data exists, we populate the file system with our seed data and then load it up.

##Loading File System Data
Our `storageContainer` has a `loadExistingPrefs` method that will load any existing band/artist preferences from a `data.json` file, if it exists:

	var module = $.extend({
        loadExistingPrefs: function (cb) {
            this.fs.root.getFile('data.json', { create: true }, function (fileEntry) {
                fileEntry.file(function (file) {
                    var reader = new FileReader();
                    reader.onloadend = function (e) {
                        var contents = JSON.parse(this.result || "[]");
                        if(cb) {
                            cb(contents);
                        }
                    };
                    reader.readAsText(file);
                }, errorHandler);
            }, errorHandler);
        }
        // other members, etc.
	}, Monologue.prototype);
	
By calling `this.fs.root.getFile`, we're telling the file system that we want to load the `data.json` file (first arg) and create it if it doesnt already exist (`{ create: true }`, second arg). The third arg is the success callback that's invoked once the file is opened. Inside our callback we use the `fileEntry` instance passed to us to get a handle to the file and read it. We hook up an `onloadend` handler (which will fire once the file read has completed), in which we parse the JSON contents into the `contents` variable. The code that called `loadExistingPrefs` should have passed in a callback for us to invoke once we have the parsed contents - so we pass the `contents` variable to that callback. Finally, we kick the read off by invoked `reader.readAsText(file)`.

Overall reading isn't terribly complicated, but it could be a bit confusing if you're brand new to the asynchronous nature of JavaScript (and passing continuation callbacks around).

##Writing Data to the File System
Our `storageContainer` instance contains a `storePref` method, in which we append a new band/artist preference from the user to the existing file and save it.

	var module = $.extend({
        storePref: function (pref) {
            var currentContents, self = this;
            self.loadExistingPrefs(function(contents) {
                contents.push(pref);
                self.fs.root.getFile(
                    'data.json',
                    { create: false }, 
                    getWriter(
                        contents, 
                        function() { 
                            self.emit("prefs.exist", contents); 
                        }
                    ), 
                    errorHandler
                );
            }); 
        }
        // other members, etc.
	}, Monologue.prototype);

The first thing we do is load the existing prefs into memory. ()Technically, I could have held onto the already-parsed contents from when we loaded the preferences earlier, but I wanted to show these actions together.) Once we have the existing preferences loaded, they get passed to our callback (as the `contents` arg). We push the new band/artist preference into the existing `contents` array. Then we call `self.fs.root.getFile` to open the `data.json` file (first arg) for writing. I've specified that we're not creating the file, since it should already exist. The third arg to `getFile` is our success callback. We're calling `getWriter`, which returns a properly configured success callback:

	var getWriter = function(contents, cb) {
        return function (fileEntry) {
            // Create a FileWriter object for data.json.
            fileEntry.createWriter(function (fileWriter) {
                fileWriter.onwriteend = function (e) {
                    console.log('Write completed.');
                    if(cb) { cb(); }
                };
                fileWriter.onerror = function (e) {
                    alert('Write failed: ' + e.toString());
                };
                // Create a new Blob and write it to data.json.
                var blob = new Blob([JSON.stringify(contents)], {
                    type: 'text/plain'
                });
                fileWriter.write(blob);
            }, errorHandler);
        }    
    };
    
Our `getWriter` function takes the contents we want to save and a callback, and returns a function that handles writing the data. Inside it you can see that we're using the fileEntry handle passed in (`getFile` passes this arg into the success callback) to create a writer. Once we have the `fileWriter` instance (yet another level deep in nested callbacks, sigh), we hook up a `onwriteend` and `onerror` handler. Then we stored our serialized `contents` array in a BLOG and write it to our file. It's important to note that I'm overwriting the file in this case, not appending. You *can* append, though. If you simply wanted to add something to the end of the file, then you could include `fileWriter.seek(fileWriter.length);` before you call `fileWriter.write(blob)`.

##Mobile Device File System APIs
In my role as a Developer Advocate for [Icenium](http://www.icenium.com/), I definitely run into the need to store data in a mobile device's file system when building [hybrid mobile apps](http://tech.pro/blog/1355/when-to-go-native-mobile-web-or-cross-platformhybrid). Icenium, like PhoneGap, uses [Apache Cordova](http://cordova.apache.org/), which already supports file system access. The good news is the Apachae Cordova File API is based on the W3C File API spec - which means nearly everything you saw in the above examples will be relevant. The main exception is that you won't need to use webkit-prefixed methods (e.g. - use `window.requestFileSystem` instead of `window.webkitRequestFileSystem`). In fact, it's quite common to see developers normalizing the webkit-prefixed methods in web applications like this:

	window.requestFileSystem  = window.requestFileSystem || window.webkitRequestFileSystem;

##Support
Current browser support:

![](./FileSystemSupport.png)
*(taken from [http://caniuse.com/#feat=filesystem](http://caniuse.com/#feat=filesystem))*

However, you might improve on the above support if you use the [idb.filesystem.js](https://github.com/ebidel/idb.filesystem.js) polyfill mentioned earlier.

##Web SQL
As I mentioned earlier, Web SQL has been deprecated, so we're not going to spend much time on it. However, there are quite a number of applications in the wild using this feature, so you may run into one.

The [jsFiddle example](http://jsfiddle.net/ifandelse/rNbq5/) for this storage option adapts our earlier `localStorage` example to use Web SQL instead:


<iframe width="100%" height="300" src="http://jsfiddle.net/ifandelse/rNbq5/embedded/result,js,html,css/" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

##Creating/Opening a Database
At the top of the fiddle's code, you see this:

	var db = openDatabase('yaysql', '1.0', 'SQL in the web - my worst nightmare', 1 * 1024 * 1024);
    db.transaction(function (tx) {
        tx.executeSql(
            "create table if not exists " +
            "example(name string, location string)",
            [],
            function () {
                console.log("table created...maybe :-)");
            }
        );
    });
    
Thankfully, `openDatabase` will create one automatically if it doesn't already exist. The first arg is the name of the database, second is the version, third is the description of the database and the last arg is the estimated size.

Next I start a transaction by calling `db.transaction`. The callback receives a handle to the transaction via the `tx` arg. From here we call `executeSql` to create our "example" table if it doesn't already exist. The first argument to `executeSql` is the command text. The second arg is an optional array of parameters that can be mapped to placeholders in the command text (we'll see more of this in a moment). The third arg is the callback to be invoked after the command has executed.

##Saving Data to the Database
Our `saveToDB` method looks as follows:

	var store = {
        saveToDB: function () {
            var vals = this.getInputValues();
            db.transaction(function (tx) {
                tx.executeSql(
                    "INSERT INTO example (name, location) VALUES (?, ?)",
                    [vals.fullName, vals.location],
                    function () {
                        console.log("Saved");
                    }
                );
            });
        }
        // more members, etc.
    };
    
To save the data, we start a new transaction. Inside the transaction callback, we call `executeSql` again, this time passing an INSERT command. Note the `(?, ?)` placeholder for the value to be inserted. The items we pass in the second argument array will be mapped, in order, to the `?` placeholders. Our third arg, like before, is a callback to be invoked once the command has executed.

##Retrieving Data
Let's look at the `loadFromDB` method:

	var store = {
        loadFromDB: function (cb) {
            db.transaction(function (tx) {
                tx.executeSql(
                    'SELECT name, location FROM example',
                    [],
                    function (tx, results) {
                        if (!results.rows.length) {
                            alert("Nothing is stored in the DB currently.");
                            return;
                        }
                        // we're only using the first row...example cheating FTW
                        cb(results.rows.item(0).name, results.rows.item(0).location);
                    }
                );
            });
        }
        // more members, etc.
    };
    
I'm cheating a bit in this example, since I'm only concerned with the first row returned from our query - but this looks a lot like the other commands we've run, with the exception of the fact that we're actually making use of the arguments passed to our third argument callback. The second arg to that callback, `results` contains the records returned from our query. If we have rows, we grab the name and location from the first one and pass them into the `cb` callback, which was passed by the code that invoked `loadFromDB`. You can, of course, but more substantial SQL commands if necessary (specifying a WHERE clause, for example).

##Deleting Data
Just like the other commands, we start with a transaction, and pass in a DELETE command as our command text, removing all the rows currently in the "example" table.

	var store = {
		clearDB: function () {
            db.transaction(function (tx) {
                tx.executeSql(
                    'DELETE FROM example',
                    [],
                    function () {
                        console.log("Deleted data from DB");
                    }
                );
            });
        }
        // more members, etc.
    };

## Support

![](./WebSqlSupport.png)
*(taken from [http://caniuse.com/#feat=sql-storage](http://caniuse.com/#feat=sql-storage))*

#Wrapping Up
Thus far in my experience with client-side storage, two recurring themes have popped up:

* Do *NOT* assume the data will be there. Exceptions to this are understandable when you're dealing with hybrid mobile apps, or Chrome Apps - but if it's a standard web site, be sure your site can continue chugging along if the data it persisted there last time has since disappeared.
* Libraries that normalize APIs and handle fallback support can be extremely useful. Some of the ones I've run across are listed in the next section 

I've linked to (below) a few of the source articles I've referred to a lot in my own development, but I'm also very interested in hearing about your experiences using any of these storage options in your apps. If you have more information on size limits (or other features) you feel I should have mentioned or focused on - what would that be?

Finally - I *intentionally did not cover using cookies* as a client side storage option, though you certainly &lt;airquote&gt;**can**&lt;/airquote&gt;. Aside from (mostly marginalized) [stubborn ancient hold-outs](http://www.ie6countdown.com/), the web has matured well beyond being stuck with 4KB mini-key-value stores. However, if the need arises, there are many libraries out there to make dealing with cookies less painful - the most popular, perhaps, being [jquery-cookie](https://github.com/carhartl/jquery-cookie).


#Further Reading & Libs To Checkout

## Libraries

* [amplify.store](http://amplifyjs.com/api/store/) - I've used amplify.store on several projects. It definitely takes away the headaches of Web Storage support on legacy browsers.
* [lawnchair](http://brian.io/lawnchair/) - very slick library by Brian Leroux that enables key/value storage over several types of underlying storage implementations (you can write adapters for your storage medium of choice if one doesn't already exist)
* [pouchdb](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&ved=0CCsQFjAA&url=http%3A%2F%2Fpouchdb.com%2F&ei=kD0BUpeZJ4_s9AS7sYGwAQ&usg=AFQjCNGgF7SsYLGNlISRoB7WTqc39kXMSg&bvm=bv.50310824,d.eWU) - interesting wrapper around IndexedDB that allows it to synchronize with a CouchDB backend. If you're using CouchDB, and want to let someone else handle the heavy lifting of data synch - check this out.
* [IndexedDB Polyfill](https://github.com/axemclion/IndexedDBShim)
* [idb.filesystem.js](https://github.com/ebidel/idb.filesystem.js)

##Other Links

* [MDN - DOM Storage Guide](https://developer.mozilla.org/en-US/docs/Web/Guide/DOM/Storage)
* [W3C - Web Storage Recommendation](http://www.w3.org/TR/webstorage/)
* [MDN - IndexedDB](https://developer.mozilla.org/en-US/docs/IndexedDB)
* [MDN - Using IndexedDB](https://developer.mozilla.org/en-US/docs/IndexedDB/Using_IndexedDB)
* [W3C - IndexedDB Spec](http://www.w3.org/TR/IndexedDB/)
* [Real-world example of the HTML5 FileSystem API](http://www.adobe.com/devnet/html5/articles/real-world-example-html5-filesystem-api.html)
* [W3C - File API](http://www.w3.org/TR/file-system-api/)
* [5 Obscure Facts About HTML5 LocalStorage](http://htmlui.com/blog/2011-08-23-5-obscure-facts-about-html5-localstorage.html)
* [HTML5 Rocks - Client-Side Storage](http://www.html5rocks.com/en/tutorials/offline/storage/)
* [idb.filesystem.js - Bringing the HTML5 Filesystem API to More Browsers](http://ericbidelman.tumblr.com/post/21649963613/idb-filesystem-js-bringing-the-html5-filesystem-api)



