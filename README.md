napp.alloy.adapter.restsql
==========================

## Description

SQL & RestAPI Sync Adapter for Titanium Alloy Framework. 
The idea is to combine Restful API with a local sql database. 


## How To Use

Simple add the following to your model in `PROJECT_FOLDER/app/models/`.

	exports.definition = {
		config : {
			"columns": {
				"id":"INTEGER PRIMARY KEY",
				"title":"text",
				"modified":text
			},
	       relations : { // optional
        		'photos' : {
        			method : 'fetchPhotos', // optional, if not set will be determined automatically
        			collection_id : 'album_id' // optional, if not set, determined automatically
        			type : 'one' // optional, 'one to many' or 'many to many'
        		}
	        },
			"URL": "http://urlPathToRestAPIServer.com/api/modelname",
			"debug": 1, //debug mode enabled
			"useStrictValidation":1, // validates each item if all columns are present
			"adapter" : {
				"type" : "sqlrest",
				"collection_name" : "modelname",
				"idAttribute" : "id",
				 "deletedAttribute" : "deleted_at"
				
				// optimise the amount of data transfer from remote server to app
				"addModifedToUrl": true,
				"lastModifiedColumn": "modified"
			},
			
			//optional
			"headers": { //your custom headers
	            "Accept": "application/vnd.stackmob+json; version=0",
		        "X-StackMob-API-Key": "your-stackmob-key"
	        },
	        
	        // delete all models on fetch
			"deleteAllOnFetch": true
		},
		extendModel : function(Model) {
			_.extend(Model.prototype, {});
			return Model;
		},
		extendCollection : function(Collection) {
			_.extend(Collection.prototype, {});
			return Collection;
		}
	}

Then add the `sqlrest.js` to `PROJECT_FOLDER/assets/alloy/sync/`. Create the folders if they dont exist. 

Use the `debug` property in the above example to get logs printed with sql statements and server response to debug your usage of the sqlrest adapter.

## New migration methods

### createIndex (@moshemarciano)

Adds more indexes to the database table, primary key is already an index.
both one column index and multiple column index are supported 

	migration.up = function(db) {
		db.createTable({
	        columns: {
	            id: 'INTEGER PRIMARY KEY',
	            time: 'TEXT'
	        }
		});
		db.createIndex({
			'firstIdx' : ['id', 'time'],   
		   	'timeIdx' : 'time'
		});
	};


## Special Properties

### URL Parameters (@moshemarciano)

Use a custom URL that only part of it changes from one HTTP call to the other

	exports.definition = {
		config : {
			.
			.
			'URL': 'http://urlPathToRestAPIServer.com/api/albums/{album_id}/modelname',
			.
			.

	photos.fetch({
		urlParams: {
			album_id : 34	
		},
		.
		.

parameters will be replaced *before* the URL is used to communicate with the server

### fetchTriggerOnChange (@moshemarciano)

Only trigger a fetch on the collection if anything was done to change it (create/delete/update)
	
	photos.fetch({
		fetchTriggerOnChange : true,
		.
		.

use this property to reduce the need to do data processing and view updating if not needed. default behavior is to trigger a fetch even if no changes are made(backward compatibility)

### Relationships (@moshemarciano)

Setup a relationship between models/collections, if your backend sends back JSON data which includes albums for example, and each album has a photos attribute which in itself is a sub collection, you can specify it like this and the adapter will detect it and begin a recursive SYNC operation on the sub-collection, so in one fetch, one HTTP request you can update multiple collections.

    relations : {
		'photos' : {
			method : 'fetchPhotos', // optional, if not set will be determined automatically
			collection_id : 'album_id', // optional, if not set, determined automatically
			type : 'one' // optional, 'one to many' or 'many to many'
		}
    },

consider creating one 'fake' collection called 'updates' which can issue a REST request to the server and include a timestamp of the last fetch, the server returns new data for multiple collections which are automatically handled by the adapter and synced to the local SQL database in one single fetch. This can greatly reduce server round trips, code complexity and overall app feel.

it will also create new methods for your model, named after the collection, for example if you specified a 'photos' collection, a new method will be added to the album model called 'fetchPhotos'. Also, if your albums collection id attribute is 'id', the adapter will automatically use 'album_id' to make the connection between the photos collection and the albums collection. The naming uses [underscore.inflections.js](https://github.com/jeremyruppel/underscore.inflection) code to get singular representations of collection and attributes names.

You can override these automatic naming methods if you like as seen above.

if you have a scenario in your app where you need to fetch the sub collection data for the user selected master collection, you can do this:

	albums.at(index).fetchPhotos({
		success : function  (collection, response, options) {
			Ti.API.info('photos for album : ' + JSON.stringify(photos, null, '\t'));
		}, 
		sql : { 
			orderBy:"title" 
		}	
	});

this way you only store in-memory the part of the data you need at any given point of time and not the whole data set. Can greatly reduce the amount of memory your app uses.

note that when you use fetch with a success and error callbacks, it will only fire once, regardless of the number of sub collections retrieved in the fetch. Only the invoking collection/model will fire. However you can monitor sub collection changes using Backbone event listeners, all sub collections are fetched with implicit fetchTriggerOnChange property ON

	photos.on('fetch', function(e) {
		Ti.API.debug('>>>>> business_types fetch');	
	});

### trackChangesInMemory (@moshemarciano)

Tells the adapter to track changes (create/delete/update) and collect the model id for each change. The changed models are collected into the *config.adapter.changed* array
	
	photos.fetch({
		fetchTriggerOnChange : true,
		trackChangesInMemory : true,
		.
		.

	photos.on('fetch', function(e) {
		_.each(photos.config.adapter.changed, function(model) {
			// only changed models are processed
			alert(photos.at(model).get('title'));
		});
	});

### trackChangesInModel (@moshemarciano)

Tells the adapter to track changes (create/delete/update) and collect the changed model id for each change. The changed models are collected into a database based model specified by the *trackModel* property or if not set, in the model name. Changes are tracked for the main collection as well as every related model as specified in the *relations* property

the model needs to follow a specific structure:
	
	exports.definition = {
	    config : {
	        'URL': baseURL + '/updates/{timestamp}',
	        'debug': debugMode,
	        columns : {
	        	id : 'INTEGER PRIMARY KEY AUTOINCREMENT',
				collection : 'TEXT', 
				changedID : 'INTEGER', 
				type : 'TEXT', 
				priority : 'INTEGER'
	        },
	 	    relations : {
				'albums' : {},
				'photos' : {},			
		    },        
	        'adapter' : {
	            'type' : 'sqlrest',
	            'collection_name' : 'updates',
	            'idAttribute' : 'id'
	        },
	    },
	    .
	    .
	    .

In this case it will use the *updates* table/model name since no specific *trackModel* was set.

we can then ask the adapter to fetch the data from the server while keeping tabs on each model that changed, so we can later process the changes and act upon them(e.g. download the photos that changed). 

	updates.fetch({
		urlParams: {
			timestamp : last_update_timestamp
		},
		fetchTriggerOnChange : true,
		trackChangesInModel : true,
		error : function (response, options) {
			// handle error
		},
		success : function (response) {
			// start processing changes saved 
			// in the updates collection
		}
	});	



### recievedHeaders (@moshemarciano)

used by the adapter to store the HTTP headers of the last fetch from the server, it is positioned under the config.adapter node of your model config. This is read only.

	updates.fetch({
		error : function (collection, response) {
			Ti.API.error('>>>>> fetch updates failed');		
		},
		success : function (collection, response) {
			Ti.App.Properties.setString('last_update_timestamp', updates.config.adapter.recievedHeaders.Date);
		}
	});	


### deletedAttribute (@moshemarciano)

support custom column naming to help with backend frameworks that use other naming conventions, such as PHP Laravel 4 'deleted_at'

	"adapter" : {
		...
		"deletedAttribute" : "deleted_at"
	}, 

NOTE for backward compatibility: value of deletedAttributed is checked and only processed and deleted if value is not null

### Last Modified

Save a timestamp for each model in the database. Use the `lastModifiedColumn` property in the adapter config to send the HTTP Header "Last-Modifed" with the newest timestamp. This is great for improving the amount of data send from the remote server to the app. 

	"adapter" : {
		...
		"lastModifiedColumn": "modified"
	},

This is tell the adapter which column to store a timestamp every time a model has been changed. 

### Custom Headers

Define your own custom headers. E.g. to add a BaaS API

	"headers": {
		"Accept": "application/vnd.stackmob+json; version=0",
		"X-StackMob-API-Key": "your-stackmob-key"
	}

### Nested Result Objects

Lets say you are a REST API where the results are nested. Like the Twitter search API. It has the found feeds in a results object. 
Use the `parentNode` to specify which child object you want to parse. 

	config: {
		...
		"parentNode" : "results"
	}
	
It has support for nested objects. 
	
	config: {
		...
		"parentNode" : "news.domestic"
	}

You can also specify this as a function instead to allow custom parsing the feed. Here is an example: 

*Feed:* 
http://www.google.com/calendar/feeds/developer-calendar@google.com/public/full?alt=json&orderby=starttime&max-results=15&singleevents=true&sortorder=ascending&futureevents=true

*Custom parsing:*

```javascript
parentNode: function (data) {
	var entries = [];

	_.each(data.feed.entry, function(_entry) {
		var entry = {};

		entry.id = _entry.id.$t;
		entry.startTime = _entry.gd$when[0].startTime;
		entry.endTime = _entry.gd$when[0].endTime;
		entry.title = _entry.title.$t;
		entry.content = _entry.content.$t;

		entries.push(entry);
	});

	return entries;
}
```


### useStrictValidation

Some times its important for the app to have some certainty that the data provided by the database is valid and does not contain null data. 
useStrictValidation runs through the fetch response data and only allows the items which have all columns in the obejct to be saved to the database.

	config: {
		...
		"useStrictValidation":1
	}
 

### localOnly

If you want to load/save data from the local SQL database, then add the `localOnly` property.

```javascript
collection.fetch({
	localOnly:true
});
```


### initFetchWithLocalData

Set this property to true, if you want to get the local data immediately, and get the remote data when the server returns it.

*Notice: This will trigger two fetch calls.* 


	config: {
		...
		"initFetchWithLocalData" : true
	}


### returnErrorResponse

Set this proerty to true if you want the error response object from the remote server. Default behaviour is not to return this, but return the data stored locally. 

	config: {
		...
		"returnErrorResponse" : true
	}


### Extended SQL interface

You can perform a local query, and use a bunch of SQL commands without having to write the actual query. The query is also support btw.
Use: *select, where, orderBy, limit, offset, union, unionAll, intersect, except, like, likeor*

```javascript
collection.fetch({  
	data: {  
		language: "English"
	},  
	sql: {  
		where:{  
			category_id: 2
		},
		orderBy:"title",
		offset:20,
		limit:20,
		like: {
			description: "search query"
		}
	},
	localOnly:true
});
```

## Changelog

**v0.1.36**
Bugfix for building the sql statement. If orderby, likeor and limit was used at the same time, the query was incorrect. This has been fixed. 

**v0.1.35**
Added support for deleting all models on fetch. Use `deleteAllOnFetch` in adapter config. 

**v0.1.34**
Added support for `addModifedToUrl` and `lastModifiedDateFormat` to better control over the outcome of `Last Modified`. 

**v0.1.33**
Bugfix for special case where collection has assigned an id and hence fetch will only return a single model. 

**v0.1.32**
Added `returnErrorResponse` enables the developer to get the error response object from the remote server. 

**v0.1.31**  
Bugfix for Last Modified Column.  

**v0.1.30**  
JSON.parse errors are now caught.  

**v0.1.29**  
Small bugfix. Issue when using `localOnly` and `initFetchWithLocalData` together.  

**v0.1.28**  
Added support for `returnExactServerResponse` which is useful for remote search. 

**v0.1.27**  
Bug fix for building sql queries. typeof array corrected to _.isArray

**v0.1.26** 
Added support for `parentNode` as a function for custom parsing. thanks @FokkeZB

**v0.1.25**  
More bugfixes

**v0.1.24**  
Bugfix: Auto ID's are not stored when idAttribute is not set  

**v0.1.23**  
Added `initFetchWithLocalData` to fetch params.   
Better logic for update/create. Now local db handles duplicate ids.   

**v0.1.22**  
Added `initFetchWithLocalData` for fetch local data before remote.  

**v0.1.21**  
Added `useStrictValidation` for fetch response data.   

**v0.1.20**  
Support for parsing nested result objects.

**v0.1.19**  
Support for custom headers.  
Fix for last modified column not existing on a new model.

**v0.1.18**  
Bugfix for select statements.

**v0.1.17**  
Added where statement that iterate an array. 
Bugfix for select statements. 
Optimised the localOnly request to only return data. 

**v0.1.16**  
Added combined data and sql.where queries. 
isCollection check. 
Bugfixes for `Last Modified`. 

**v0.1.15**  
Added check for model in read.

**v0.1.14**  
Added `Last Modified` for data transfer optimisation & debug mode

**v0.1.13**  
Updated for Alloy 1.0.0.GA

**v0.1.12**  
Added `urlparams` and improved `update` rest method

**v0.1.11**  
Advanced sql interface and localOnly added. 

**v0.1.10**  
Init 


## Author

**Mads Møller**  
web: http://www.napp.dk  
email: mm@napp.dk  
twitter: @nappdev  

## License

    Copyright (c) 2010-2013 Mads Møller

    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:

    The above copyright notice and this permission notice shall be included in
    all copies or substantial portions of the Software.

    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
    THE SOFTWARE.