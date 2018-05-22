# Tagtime Data

## The problem
Tagtime pings must be answered _at the instant they arise_ and so it is
important to have the system available on everywhere - on your phone,
tablet, laptop, desktop, or watch.

However we also would like the system to work offline - in case the
server fails, the user's network fails, or for some reason or the other
the user cannot connect to the network.

In this situation, we run into synchronization problems. As anyone who
has used any syncrhonization service (Apple Cloud, iTunes, Azure Active
Directory) has experienced, synchronization is a "hard problem" that
cannot simply be _tacked on_ to an existing architecture. Software
developers were forced into this realization early yet it took nearly
four decades until version control tools matured to the point where
synchronization became truly usable with the advent of
[git](https://git-scm.com/) and [github](https://github.com/).

### User Data
It is important to note that if either of these two requirements are not
present (i.e. we can guarantee network synchronization OR only single
entry point), this problem does not exist. However, we would like a
robust system hence we must deal with the issue for ping and tag data.

However the situation for user data (user profile, billing info, and
login) is different - here we do not need to support offline (or even
multi-device). It is sufficient for the user to create and manage their
account when they are connected and through a web interface.

User data will therefore be stored in a MySQL database in a traditional
user table with tenant info and so on. The design of this database
structure is described below:

    Table user
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(128) CHARSET UTF8 NOT NULL UNIQUE,
    password VARCHAR(128) CHARSET UTF8 NOT NULL,
    firstname VARCHAR(128) CHARSET UTF8,
    verification_key VARCHAR(128) CHARSET UTF8,
    last_login DATETIME,
    state SMALLINT UNSIGNED
    
    Table device
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    device_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    devicename VARCHAR(128) CHARSET UTF8,
    session_key VARCHAR(128) CHARSET UTF8
    

When the user signs-up we create a user in our system with state not verified 
simultaneously a verification code is sent in an email. The next time he 
logins he is asked to enter the verification code. If correct his state changes
to verified.

The hashed password is stored and used to verify his identity whenever he signs in. 

We ask him for the devicename and we store a non-expiring cookie that will 
prefill the devicename everytime he connects or signs in. If there is no cookie
he is provided with a list of devices to choose from or enter the name of a 
new device.

He will be also sent a cookie with session id that will be used to automatically
login the user from a specific device


He can ask to reset his password in which case all his session-ids are invalidate
A mail is sent with password change authorisation key. In Change password screen
he can either re-enter old password or change password authorisation key and then
enter new password.


We now turn our attention to the problem of pings & tags.

## Enter the ... Log
When we examine the way [git](https://git-scm.com/) and
[other](https://bitcoin.org/bitcoin.pdf)
[distributed](https://en.wikipedia.org/wiki/Blockchain)
[systems](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
systems work we find at their heart a very simple structure - the Log.

This structure gave rise to the [Kappa
Architecture](http://milinda.pathirage.org/kappa-architecture.com/)
which we will use as the basis for our design of the TagTime database.

## Log Structure Details
The tagtime ping logs will have the following characteristics:
1. Each user/device will have it's own log
2. Each log is an immutable list of records
3. Each record is sequenced one after another (monotonically increasing)
   by the [unix time](https://en.wikipedia.org/wiki/Unix_time) of it's
   entry.
4. The logs are processed sequentially - starting from the first and all
   the way to the last. As new logs come in they are processed to update
   the system view/snapshot.

Hence, conceptually the system looks like this:

![log view](log-view.png)


Now using the unix time of each entry we very easily interleave them to
form a combined log view:

![combined log view](combined-log-view.png)


We can see that if any view is offline for a while the combined log will
not contain it's data, but as soon as it comes back online it will
simply add it's entries into the entire stream.

## Log processing
In order to get the final state of data for processing, the logs are
processed sequentially - starting from the first and going all the way
to the last.

To 'update' data, new logs are simply appended which results in the
client/processor updating it's 'snapshot' of the data. Similarily when
updates are made to the data they are translated back into new log
messages that are appended to the log.

This means that _during startup_ the client/processor needs to go
through every log from the beginning of time. However, once loaded every
new update to be processed is going to be very quick as it only applies
the latest differential change to the data.

The startup speed should not be an issue because `Tagtime` starts up
with the computer and continues running from that point onward.


## Errors and Debug Support
We will also combine error logs and event logs into the same source
stream. This allows us to analyze problems and provide user support
by re-using the same mechanism making our lives easier.

## Device View
The overall view of a single device now looks like this:


Question : Will each non-local come seperately to be appended to specific log
and then combined? When does combining happen. 

My thought: on Startup and whenever data comes from server based on subscription
Can server send device specific logs as well as merged log so that combining is easy.

Will server flag out of sequence data so combine may have to truncate and rebuild.

This may be premature optimization. 

![overall view](device-view.png)

## Server View
The server simply accepts logs (subject to correct user authentication),
saves them, and pushes notifications to all connected devices that the
log has been updated. Each device can then request for updates from
whichever sequence point they last had (their last unix time for that
log).

## Types of Log Messages
Log messages can be of various types:
1. Data: Ping Data, Category data, User Config Data? etc etc
2. Error Logs
3. Event Logs
4. Commands: For example, "rename tag from X to Y" may be a command
   rather than actually updating all the data logs that contain that tag

Suggestion: When combining logs we can have commands make retrospective 
changes and disappear from combined log. 

### Interpreting tags
Having logs of various types brings up an interesting property of this
architecture: _as long as the client/processor_ ignores log messages it
does not understand we can send newer logs to old clients with no
problems - they will simply process the logs they understand and show
the resultant view. Once upgraded they will simply show the additional
features.

For example if we did not support tag renaming in an older version once
it started being supported all old clients would simply show the
un-renamed tags (as they do not have a way to rename) and then the
renamed tags once upgraded.


## Components Used
These are the components we will use for TagTime data management:
1. Maria Db
2. [Bolt DB](https://github.com/coreos/bbolt) for Golang Log Storage
3. [NeDB](https://github.com/louischatriot/nedb) for Electron Client
   Storage
4. https://github.com/tekartik/sqflite for Flutter App Storage
