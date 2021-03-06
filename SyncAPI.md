## nodejsdb
# Synchronous API Specification v1

The base structure that should be sufficient for most use cases is a key-value map with ordered key elements. In the context of synchronous API, this structure is fully contained in fast memory (RAM).

A properly implemented tree structure should not be slower (e.g. see "Performance" [here](https://github.com/pconstr/rawhash)) than an un-ordered (hash) map, and thus the necessity of multiple types of maps can be avoided.

The need for a pure list structure is currently unproven, as in many cases the FIFO/LIFO structures can be implemented using an ordered map with unique or weighted keys, thus for now a list structure is not included, although if real-life experience proves to require one, it will be added.

The fact that this version of the API is synchronous is allowing to make it very simple, e.g. atomic operations are not needed (because all DB calls are atomic), callback result functions are not necessary, and command queuing is not needed (because there is virtually zero latency to be avoided). In fact, the sheer simplicity of the synchronous API and its programming model may in the end prove more efficient than a multi-threaded, (possibly off-process), asynchronous version (both development and runtime efficiency considered). A supporting rationale for that case would be: *"At some point, you are going to have to shard anyway, so do it early and do it properly"*.

To maintain maximum implementation and usage flexibility, the API doesn't specify (external) persistence. (Intrinsic persistence is not considered to be feasible with a synchronous API). External persistence can easily be added via the 'set' event (see below). This allows for implementations which e.g. don't store data to hard-drive at all, but rather commit them in batches (where the buffered batch can be protected by multi-machine redundancy; /contrast this with full-data redundancy a.k.a mirroring/) to an external store, like S3.



## Usage

```js
// request the DB
var db = require('the-impl');

db.set('users', 1234, { fname: 'abc', lname: 'def' });
var user1234 = db.get('users', 1234);
var subset = db.range('users', true, 1200, 1300);
var sz = db.size('users');

```

## Operations

### Set

	db.set(name, key, value, event);

	Parameters
		name: name of the collection
		key: the key
		value: the value to create, overwrite, or delete
		event: optional; override the name of the callback; see "Set Event" below

	Spec
		1. both key and value support (and store the type of):
		    String, Number, Boolean, Object, Array, Buffer
		2. Object and Array are serialized without checking for cyclic dependencies
		3. everything except Number is compared (and stored) as binary string
		4. Number compared with binary string always compares lower (is ordered in front of it)
		5. setting `null` as value deletes the key (thus no explicit delete operation)
		6. `null` keys are not supported
		7. on insufficient memory, an exception is thrown with message 'out of memory'
		8. `true` is returned if the DB was changed:
		  8a. on delete, the key was existing and deleted
		  8b. on set, the key did not exist
		  8c. on set, the key did exist but had a different value
		9. collection is automatically created/deleted when the first key is added/last key removed
		
	Examples
		db.set('users', 'mykey', { fname: 'abc', lname: 'def', created: Date.now() });
		db.set('usersByScore', 123.456, 123);
		db.set('users', 'mykey'); // delete 'mykey'

### Set Event

	db.on.mycollection = function(name, key, value, previous) {
		// your handler code here
	};

	Parameters
		name: name of the collection that was changed
		key: the key
		value: the value that was set, `null` for deleted
		previous: the previous value that was replaced

	Spec
		1. after each `.set()` which changes the database state,
			an event is triggered if a handler for it is registered
		2. by default, handlers are registered under the collection name
		3. this collection name registration can be overriden using the `event` parameter of `.set()`
		4. since this is a synchronous API, you have to return from the handler synchronously (and fast)

	Examples
		// classic example:
		db.on.myCollection = function(name, key, value, previous) {
			// `name` is 'myCollection'
		};
		db.set('myCollection', 'somekey', 'somevalue');

		// example of event type overriding:
		db.on.myDynamicCollection = function(name, key, value, previous) {
			// `name` will be 'myDynamicCollection-12456' for the example `set` below
		};
		db.set('myDynamicCollection-12456', 'somekey', 'somevalue', 'myDynamicCollection');

### Get

	db.get(name, key);

	Parameters
		name: name of the collection
		key: the key
	
	Spec
		1. data is returned in the type it was stored under
		2. the key doesn't have to match the original type, but it has to match when binary-serialized
		3. null is returned for non-existent keys
		
	Example
		var val = db.get('users', 'userkey');

### Range


	db.range(name, descending, from, to, limit);

	Parameters
		name: name of the collection
		descending: optional; true if the traversal order is in descending order, false for ascending
		from: optional; what key to start at, inclusive; use `null` for start at the edge
		to: optional; what key to end at, inclusive; use `null` to end at the edge
		limit: optional; max number of results to return
		
	Spec
		1. result is aray of objects with properties key: and value:
		2. keys and values are decoded to their original types
		3. if the map is non-existent, returns an empty array
		4. called with null `name` operates on the root collection,
			which contains all the other named collections
	
	Examples
		var res = db.range('usersByName', true, 'A', 'Z', 10); // res = [ { key: , value: }, ... ]

		res.forEach(function(v) {
			myusers.push(db.get('users', v.value));
		});

		// list all collections in the database
		db.range();

		// same as above but with parameters
		db.range(null, false, 'mycolA', 'mycolZ', 100);

### Size

	db.size(name);

	Parameters
		name: name of the collection
		
	Spec
		1. returns the number of keys in the map
		2. called with null `name` returns the size of the root collection,
			which contains all the other named collections
	
	Examples
		// get number of users
		var sz = db.size('users');

		// get number of collections in this DB
		var all = db.size();

## Example

A simple schema with 2 models: __User__ and __Message__, where one sender can message multiple recipients. The storage API is hypothetical. This example uses `Array.forEach` for brevity, but in a high-performant code you probably want to use [something else](http://jsperf.com/fore-vs-for/2).

```js

// User: { login: }
// Message: { sender:, recipients:, text:, date: }

/*
	+----------------+              +------------------+
	|                | 1  Sender    |                  |
	|                |<-------------+                  |
	|      User      |              |     Message      |
	|                |<-------------+                  |
	|                | * Recipients |                  |
	+----------------+              +------------------+
*/

var db = require('nodejsdb-impl');
var storage = require('storage-impl');

db.on.users = function(name, key, value, previous) {
	if(!value) {

		// handle deletion; cascade to Message.sender
		db.range('messagesSentBy-' + key).forEach(function(v) {
			db.set('messages', v.value, null);
		});
	}
	if(!storage.loading)
		storage.store(name, key, value);
};

db.on.messages = function(name, key, value, previous) {
	if(!value) {
		
		// handle Message deletion, remove from senders
		db.set('messagesSentBy-' + previous.sender, key, null);
		
		// remove from recipients
		previous.recipients.forEach(function(v) {
			db.set('messagesReceivedBy-' + v, key, null);
		});
	}

	var r = value.recipients;
	for(var i = 0, len = r.length, x = rnd(); i < len; i++)
		db.set('messagesReceivedBy-' + r[i], value.date + '.' + (x + i), key);
		
	db.set('messagesSentBy-' + value.sender, value.date + '.' + rnd(), key);
	
	if(!storage.loading)
		storage.store(name, key, value);
};

db.on.messagesReceivedBy = function(name, key, value, previous) {
	// do something generic for the 'messagesReceivedBy-<userID>' collection class
	var userID = parseInt(/-(\d+)$/.exec(name)[1], 10);
	// ...
};

// get incoming messages for a user, ordered from newest to oldest
function getInboxForUser(userId) {
	var messages = [];
	db.range('messagesReceivedBy-' + userId, true).forEach(function(v) {
		messages.push(db.get('messages', v.value));
	});
	return messages;
}

// get sent items for a user, ordered from newest to oldest
function getSentItemsForUser(userId) {
	var messages = [];
	db.range('messagesSentBy-' + userId, true).forEach(function(v) {
		messages.push(db.get('messages', v.value));
	});
	return messages;
}

function rnd() {
	return Math.round(Math.random() * 10000);
}

storage.loading = true;
storage.read('./db').each(function(col, key, val) {
	db.set(col, key, val);
});
storage.loading = false;


// DB is ready here.


if(!db.size('users')) {
	console.log('Creating data...');
	
	db.set('users', 'user1', { login: 'user@one' });
	db.set('users', 'user2', { login: 'user@two' });
	db.set('users', 'user3', { login: 'user@three' });

	db.set('message', db.size('messages') + 1,
		{ text: 'hello 1', sender: 'user1', recipients: [ 'user2', 'user3' ], date: Date.now() });
	db.set('message', db.size('messages') + 1,
		{ text: 'hello 2', sender: 'user2', recipients: [ 'user1', 'user3' ], date: Date.now()});
}

console.log('Inbox for user1');
getInboxForUser('user1').forEach(function(v) {
	console.log(v.value);
});

console.log('Sent Items for user1');
getSentItemsForUser('user1').forEach(function(v) {
	console.log(v.value);
});

// yup, that's it!

```

Functional equivalent of the above schema and logic, implemented in SQL (the MySQL variant):

```sql
create table users (
	id varchar(255) primary key,
	login varchar(255));

create table messages (
	id serial primary key,
	sender varchar(255) references users(id) on delete cascade,
	text_ varchar(500),
	date timestamp default now, index(date));

create table recipients (
	message bigint unsigned references messaages(id) on delete cascade,
	user varchar(255) references users(id) on delete cascade);
	
delimiter $$;

create procedure create_message(in _sender varchar(255), in _text text, out _msgid bigint unsigned)
proc:begin
	insert into messages (sender, text_) value (_sender, _text);
	set _msgid = last_insert_id();
end;
$$

create procedure create_message_recipient(in _message bigint unsigned, in _recipient varchar(255))
proc:begin
	insert into recipient (message, user) value (_message, _recipient);
end;
$$

create procedure get_inbox_for_user(in _user varchar(255))
proc:begin
	select * from recipients r, messages m
		where user = _user, r.message = m.id order by m.date desc;
end;
$$

create procedure get_sent_items_for_user(in _user varchar(255))
proc:begin
	select * from messages m
		where m.sender = _user order by m.date desc;
end;
$$

delimiter ;

insert into users (id, login) value ('user1', 'user@one');
insert into users (id, login) value ('user2', 'user@two');
insert into users (id, login) value ('user3', 'user@three');

declare _msgid bigint unsigned;

call create_message('user1', 'hello 1', _msgid);

call create_message_recipient(_msgid, 'user2');
call create_message_recipient(_msgid, 'user3');
-- alternative:
-- insert into recipient (message, user) values (_msgid, 'user2'), (_msgid, 'user3');

call create_message('user2', 'hello 2', _msgid);

call create_message_recipient(_msgid, 'user1');
call create_message_recipient(_msgid, 'user3');
-- alternative:
-- insert into recipient (message, user) values (_msgid, 'user1'), (_msgid, 'user3');

select 'Inbox for user1' as msg;
call get_inbox_for_user('user1');

select 'Sent Items for user1' as msg;
call get_sent_items_for_user('user1');

```
