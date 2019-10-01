# flickr-fetcher

Flickr Fetcher is is a simple example program which fetches the most recent flicker photos
and then resizes them so that an endpoint can then serve them.

## The request

Implement and push to a github repo a small http service that has one endpoint which downloads,
resizes and stores images from the flicker feed
[documented here.](https://www.flickr.com/services/feeds/docs/photos_public/).
Using the [flickr API](https://www.flickr.com/services/developer/api/) is not necessary for this project.

 * Able to specify the number of images, if not specified then all feed images.
 * Able to specify the resize width and height for images.
 * it is sufficient to save the resized images on the local disk.
 * Use clojure
 * It is fine to use libraries that provide http server implementations and image resizing, but
 not in one solution.

I was told that this project is actually an exercise in how to queue
the resizing jobs for the images.  Otherwise it would be better to just
request the sizes from flickr using the API.  Resizing is not actually
necessary as Flickr has alread done that work.
 
## initial thoughts and information.

It feels a little strange to compose a design document in as a solitary
designer. I would prefer some interaction with other developers. Although 
I like this idea much more than writitng some code in that at least you will
see some of how I think.  

This has also turned into somewhat of a monster, each morning I wake up and I
have more idea's or a thought to add. Or some thing that I shuld remove because
it's completely unecessary if I do X over there.

I don't like documents that say to do something one way or another unless
there is good reason, which would usually mean there was a consensus of the
team that some method or feature was very important. But because we don't know
each other, as I wrote this I thought of it as documentation for things to try,
things to research that might be a good fit here and there. I gave links to examples, 
or possible librariees, things I have used or things that seem interesting that could 
be used. It's a sort of substitute for the conversation about tooling and libraries
that we might have as a team.

So this is not just a design document. I tried to give some idea of my thinking and at
the same time resources for later in case I or someone else decides to
code this thing which seems pretty interesting at this point.

The method used for resizing is not important. A webservice or one of many
clojure libraries can be used.  Likewise the server side does not seem to
be so important, as it is possible to get an endpoint with these simple
requirements running quite quickly with less than 100 lines of code.

For me, a project like this is meant for exploration and learning. I've done a lot of work with
[prismatic/plumatic schema](https://github.com/plumatic/schema) and I'm
interested in learning more about [spec](https://clojure.org/guides/spec),
so that seems like a good choice if for only that reason.
I've done simple compojure servers before, I like compojure, and
I like using swagger for documentation purposes. I've not used
[compojure-api](https://github.com/metosin/compojure-api) which has
both and with the the new [2.0 alpha](https://github.com/metosin/c2)
version it is possible to use spec or schema, so why not.

Finally, I want this to be a system which can grow beyond
it's simple beginnings. It would be easy to make an endpoint that
fetched images and resized them and was not usable in any other way,
with no ability to expand or be reused.

To that end it should have composoble sub-systems. I could use either 
[component](https://github.com/stuartsierra/component) or
[mount](https://github.com/tolitius/mount). I am agnostic about the two.
I used component in it's early days and I like it just fine. It doesn't
bother me to pass the world around. Also there is enough other stuff
to play with that it seems component is a good choice. I also like
that components are easily composable such that a component could be
easily replaced by a different implementation or the component system
could be rearranged to create a family of microservices. 
Component also makes it easier to test by allowing the creation of
alternative systems for testing.

All that said, I wonder if components or mount are a good fit. Maybe not.
This system doesn't have a lot of state.

Communication between the pieces / components can be simple functions or channels. My
thought is that the system state will not change once it is running. I could be wrong.
Component/Mount are mostly about maintaining system state. But they are also a nice 
compartmentalization of the parts, so in that way it did help with the design thinking.
Even if they are not actually necessary.

## Questioning the request and Enabling more flexibility

I would like for this to be project that is flexible enough to change and grow beyond 
the original request. 

It seems that the user experience for this might not be ideal. But it is a fun project
to experiment with core.async, threading, queues, file blobs, a database, spec/schema
a server endpoint and even a UI using clojurescript and react if desired.

The request is going to be slow if it has to fetch and resize the most recent images
every time someone asks for them. Displaying what we have while new things come in is going
to be important. It would also be nice if there was a way to not repeat work. That idea alone
breaks the original request and makes it complicated unless the back backend is decoupled
from the endpoint.

Requesting a size of height and width before knowing the size of each image is also going
to introduce distortion as the aspect ratio of the variously sized images is lost.
Maybe there should be an algorithm which chooses a size close to the requested size but preserves
the aspect ratio. Percentages would be better. 

So really it's a more interesting problem than "just fetch a number of images from the recent 
feed, convert them to the size I ask for, then show them to me." 

It would be better to ask "how can I retrieve the current feed and store the images, potentially
with some common sizes, and give those to the server when it asks while also allowing the server
to spawn a new fetch or ask for some different sizes of images we already have." 

And maybe, we give just the image metadata to the server, and then it decides which 
images and sizes it wants based on that most recent list of images. Choices to keep in mind.
Potentially the server is just going to give the response to front-end ui, unless it _is_
the front-end UI. Future choices to think about.

The ability to continously fetch and create various sized images in the background, even if it
only does it once a day (ie. do we really want to replicate the flickr feed in it's entirety?) 
would provide better performance, similar results and a better user experience. And an architecture
capable of that should also be capable of fulfilling the original request without adding 
complexity.

Add an option to update now, or just an increase in the update frequency should be sufficient.
And if we really want this thing to be a feed, turn the fetcher, worker and server components 
into micro-services and let them run. Maybe with multiple workers sharing the resizing efforts.

And if not, or that we really don't care to collect all that, ok. Perhaps we add a roll off 
feature that removes older images that aren't flagged for saving in their meta-data files.
Another feature - save tags.  Let the database component take care of that.

Granted the UI would become more complex, with the option to view different sized files or
generate new sizes.  But then we'll want filtering and searching and what not, so this project
might as well provide a solid foundation for future growth. The difference in code is
not great. The difference in architecture from a server that just fetchs and displays images is.

 *The request is for this*
 * Fetch images when asked
 * Resize images only to the size asked for.
 * display the number of images in the size asked for.
 
 *This design provides for this*
 * Fetch images by request or continually.
 * Automatically resize images after a fetch to any given variety or not.
 * Image sizes include :small, :medium and :large which preserve aspect ratio.
 * There is a :thumbnail image size of 150x150.
 * Arbitrary sizes of height and width can also be specified.
 * The components could potentially be configured to run microservices. 
 * The server can request a new fetch with new sizes 
 * The server can request new sizes for images already in the database.
 * Image metadata is used as a database which provides many potential possibilities.
 * The system of 5 components is composable for easier testing, component replacement,
 microservices or other improvements 


## Overview

This project seems to need five components. A fetcher, a worker and a
server, plus two more components which all of those potentially depend
upon. A queue and a database.

The queue is used by the fetcher to provide jobs to the worker. And also potentially by
the server if the feature of resizing existing images is added.
The queue could be just a folder of edn files and a bit of code. But it's probably easier
and better to use something like [factual's durable queue](https://github.com/Factual/durable-queue).
There is a [ready made component here.](http://github.com/danielsz/system)
There are many other choices for queue implemntations available.
The queue is used by the fetcher, worker and server, if the server is given the feature of
submitting resizes of images in the database.

The database should isolate the side effects of writing the images and the
metadata, this will make testing of everything else much easier as all
functions elsewhere should be pure. The database is used by the worker and the server
The worker gives it the newly resized images. The server requests metadata and images 
from the database.

The implementation of the database component can start as two folders, one for metadata edn
files and one with folders named by image id, one for each image.
There is a lot of space for additional features here.  Archiving, automatic roll off (ie. delete 
old images), flagging for saving, filtering, searching, a real database, etc. 

For future growth of the project the queue and database are the keys. 
If done well they can allow decoupling of the each of the services 
into micro-services.  And certainly the ability to swap out one component for
a new and improved component is a nice way to grow things.

Ultimately each component can be it's own project and this project would simply
be a component system configuration with a CLI, and new systems with similar needs could
use these components as needed.

_I had questions as to where the original image file should be fetched._

 * Fetcher - Sort of makes sense. it can send all the initial data to the database then put the work on the queue.
 * Database - also makes some sense, if given a new metadata record, go fetch the image and save it.
 * Worker   - again, when a worker gets a job, if _:original_ is not in the sizes, fetch it, if it is
 available, get it from the database then continue with creating the other sizes,
 wrap it all up with a save to the database. *I think this is the winner*
 
 If in the database or the worker, resize threads will have to wait on the arrival of the original. If the fetcher does it, the original should be there by the time the worker needs it. _ok. I still like the worker the best. It's a good fit of functionality._
 
 It feels like more consistent behavior to have it in the worker, since what it is doing is creating new
 images. _:original_ is just a special case that all others depend on.
 
 This makes the fetcher and the database much simpler both in function and in testing.
 The fetcher actually becomes simpler because it doesn't need to talk to the database at
 all.  Only the queue. The complexity of the worker is not overly different nor is testing. 

### Fetcher

 The job of the fetcher is to get the list of recent image metadata from
 the flickr feed and convert to clojure data which consists of a list
 of image metadata records, update them with any desired _resizes_ which may
 have been passed in a request from the server.

 If a count is given limit the retrieval to that count. 
 The default, like the flickr api should be 100 with a maximum of 500. 

 Send the image records to the queue.
 
 If the fetcher is running in a polling fashion, it should 
 start, on compnonent start, a process (go loop) which watches a 
 channel for requests and another channel for a *stop* for graceful 
 shutdown. Writing nice 
 [core.async components](https://cb.codes/a-tutorial-of-stuart-sierras-component-for-clojure/) 
 is a well known pattern.

 The image metadata record format should be specified in spec
 and is a reflection of the image entry record retrieved from Flickr,
 like this one. 
 
```
<entry>
    <title>Rainbow</title>
    <link rel="alternate" type="text/html" href="https://www.flickr.com/photos/amancalledalex/48812423833/"/>
    <id>tag:flickr.com,2005:/photo/48812423833</id>
    <published>2019-09-29T07:35:34Z</published>
    <updated>2019-09-29T07:35:34Z</updated>
    <flickr:date_taken>2019-09-27T18:10:07-08:00</flickr:date_taken>
    <dc:date.Taken>2019-09-27T18:10:07-08:00</dc:date.Taken>
    <content type="html">			&lt;p&gt;&lt;a href=&quot;https://www.flickr.com/people/amancalledalex/&quot;&gt;amancalledalex&lt;/a&gt; posted a photo:&lt;/p&gt;
	
&lt;p&gt;&lt;a href=&quot;https://www.flickr.com/photos/amancalledalex/48812423833/&quot; title=&quot;Rainbow&quot;&gt;&lt;img src=&quot;https://live.staticflickr.com/65535/48812423833_d54f886cbb_m.jpg&quot; width=&quot;240&quot; height=&quot;180&quot; alt=&quot;Rainbow&quot; /&gt;&lt;/a&gt;&lt;/p&gt;

</content>
    <author>
      <name>amancalledalex</name>
      <uri>https://www.flickr.com/people/amancalledalex/</uri>
      <flickr:nsid>39829646@N02</flickr:nsid>
      <flickr:buddyicon>https://combo.staticflickr.com/pw/images/buddyicon10.png#39829646@N02</flickr:buddyicon>
    </author>
    <link rel="enclosure" type="image/jpeg" href="https://live.staticflickr.com/65535/48812423833_d54f886cbb_b.jpg" />
    <category term="" scheme="https://www.flickr.com/photos/tags/" />
    <displaycategories>
            </displaycategories>
    </entry>
```

 The following additional fields are added to the record.

  * `:resizes []`     - Sizes that need to be created from this image.
  * `:sizes []`       - Sizes that currently exist for this image. 
  
 Sizes and resizes are vectors and can have any of the following as values. 
 `:thumbnail, :small, :medium, :large, :original` or any number of `[<height> <width>]`.
 
 The sizes _:small, :medium, and :large_ are percentage based, aspect ratio preserving.
  
 The fetcher may be told, by the server, which sizes need to be created.
 in which case it should put them into the resizes vector before giving the image
 records to the queue.
 
 Otherwise sizes will come from the CLI which the worker has direct access to.
  
 The fetcher should be able to work in one of two modes. By request or in a continuous polling mode
 which may also need to handle a request.
  
 If running continuously as a service the fetcher should also have a 
 secondary channel *stop* so that the component can be shutdown gracefully.
  

### Queue

A factual durable queue component. Although it is not complicated, 
[A pre-built component can be found here](http://github.com/danielsz/system) 
This component only needs to do it's job as queue.  Take image metadata files and
give image meta data files in a way that is resiliant to worker failures.

The fetcher and possible the server will give it image metadata records and the
worker will take them.


### Database

This component only saves or sends image metadata files and the
images that go with them.  Since an the database needs the size and the image itself,
it is probably easist if the worker sends them as a map with keys _:size_ and _:image_.
Or is it better to simply put them in the metadata file as a list of image maps.
`[{:image ... :size ...}, ...]`

When giving an image to save,
the database component should write the image to it's folder which is named after it's id in
_image-path_ from the CLI. The image should use a filename like so. _<image-id>.<size>.<format-ext>_

_If there is a possiblity that different image-paths may be used simultaneously the image's
path should be saved in it's metadata record._ otherwise the database can make the path
automatically from the image id as it needs it.

The simpest implementation would be a file (edn) based database of image metadata files. 
A simple component which can take image meta data files for storage and give the most recent 
image metadata entries and/or images in response to a request.

Something that might be fun would be to load the metadata files into a datascript
db at component initialization. That could provide some fun things to play with.

The final database folder should consist of edn files named after the image
id.  All of the image's sizes will resized in an image's folder. Image names should be 
of the format _<id>.<size>.<format>_

The resulting meta data record should be written to the applications _database-path_ folder
as indicated by the CLI option of the same name..

Unless a count is given a request for recent images should result in the
last 100 images in the database. This is the same default as Flickr. The
database can maintain a list of the 500 most recent images for quick
response. The Flickr API has a maximum of 500 images which is probably
a good limit.

Additional features might be the ability to archive, or delete images after a certain
time. or to retain or tag images.

### Worker

Everything is still fresh at this point.  We have the CLI resizes options and we have
an image metadata record potentially with some resize values. Those should be merged with
the resize values from the CLI. If there is no _:original_ size in the _sizes vector_ it 
will need to be fetched, if there is an _:original_ entry then the database can give it 
to us.

This seems, at this moment, to make the most sense over having the the fetcher or the database
retrieve the original image. It fits nicely into the mechanism that the worker must have anyway.
This original image could be provided as a _promise_ to threads to do the resizing,
or maybe just go get it and after that do a simple pmap for the rest. This will be called
from a go-loop so it's already it's own process.  Keep it simple until it can't be.
 
The worker should probably just be a go-loop which takes from the queue
component.  It is probably sufficient to call a function for each image which
then does a pmap to get the resized images. Then send the images along
with an updated metadata record to the database component. An effort
should be made to keep functions pure in order to isolate the side
effects and improve testability.
 
Like the fetcher in polling mode, the worker will need a channel for a *stop* 
signal to enable a graceful shutdown.
 
This component only needs to create more images size as requested 
by the CLI and resizes vector in the metadata entries in the queue component. 
It should filter the merged resizes by the available sizes to prevent repeating work. 
If the resulting resizes vector is empty and there is an _:original_ 
size, then there is nothing to do. 
 
On error, the worker should do the appropriate things in signaling the queue
that the job failed for that entry.
 
In addition to arbitrary `[height width]` possible resize values are the following.
`:thumbnail = (150x150), :small = (1/4) :medium = (1/2) and :large = (3/4)`. 
 
Before sending the metadata entry and images to the database component, 
the image's meta data record should be updated with the new sizes and if all is
successful the _resizes_ vector will be empty.

### Server

This is pretty simple really.  Use compojure-api and spec to create a nice endpoint 
with a nice swagger api document.
The server should show the most recent images in the images directory. The server can 
get the image metadata entries from database component and then do what it wishes. 

*Questions*
 * Does the server always request a new fetch ?
 * Is the server happy with what it can get from DB without a new fetch ?
 * Does the server need to ask the fetcher to do work ?
 * Should the data base provide a channel for incoming images for the server to watch ?
 
*Possibilities*s
 * The server doesn't worry about how _recent_ the images are. 
 * The server could request a new size for a group of photos by putting their records in the queue.
 * The server requests quantities and sizes that are known to be available.
 * After a request and display of recent images, the server might request new sizes for those images.
 * The server might want to request a refresh of the recents if the fetcher isn't going full tilt.
 but it is happy to show the most recent currently in the database.
*OR*
 * The server always requests recent images and possibly a count and sizes.
 
My preference would be to decouple the server from the backend, with the ability to
spawn new fetches which would be undectable to the user. 
 
Options to the end point are count and size. The possible
sizes are _thumbnail, small, medium, large, original_ or an arbitrary _[height width]_.  
if no quantity or size was given the server would simply get the 
last 100 metadata records from the database component. 

It might be nice to render the meta data along with the images
so that the links can be followed to the photographer among other things.
The meta-data records have a lot of additional information which allow for
the creation of a nice interface including filtering. 

If needed other endpoint options could be added such as photographer/user, filtering on size, etc. 
 
### CLI

I would use my own [clj-cli-ext](https://github.com/EricGebhart/clj-cli-ext) library to manage the command line interface.  It automates documentation
and makes it easy to make sub-commands which makes it easier to manage as a system grows in complexity.
It includes basic logging options which work well with any logging system someone might wish for.
Reasonable defaults should be used where appropriate.

It is probably not necessary to create component level options for everything, but as the project 
grows it does make good sense.

#### Options

 * help      - automatic with clj-cli-ext.
 * log       - automatic with clj-cli-ext.
   * file
   * verbosity
 * Fetcher
   * url 
   * timeout
   * polling on/off
    * interval <time in seconds between fetches>
 * queue
   * work-path
 * database
   * database-path
   * image-path
 * Worker
    * sizes <[:thumbnail :small :medium :large [x y] ...]>  with a wA minimum of [:original]
 * server
     
 
### Logging.

I've used [Timbre](https://github.com/ptaoussanis/timbre) in the past and it works well with care.

### component initialization

In main, the cli parser would be called, and the result of
the validated cli tree would be passed to the initialization
of each component in the system. 
 
### Testing

#### General

My usual work flow is to be in the REPL while working on both code and tests.
As the code takes form so do the tests. I don't do strict TDD. But an interplay.
For me this avoids the tedium writing a bunch of unit tests for code that is done.

With components testing can be made much easier by creating component systems which
consist only of the components required of the test. The CLI can mocked, to create
settings which would only be used for testing. Such as using a fake URL.

A sample database and image folders to match should be created in test-resources
to test the server and the worker.

Making a snapshots of the data in test-resources, and pointing the components 
to alternative folders or empty destinations should be done as needed.
This will make it easier to keep the tests idempotent.

In the case of the worker, a temporary snapshot would allow testing of a submittal
of resize work for existing images.

For testing of new work from a skeleton set of meta data files and their 
matching image folders/images it would only be necessary to point the worker at some new
database folder that would be removed before each test run.

For testing of filesystem problems it might be nice to a have a very small partition
mounted somewhere so that out of disk space errors would be easy to create.  This would
be easy to do with a VM.

#### Testing the fetcher
Testing  the fetcher is not too difficult as it only has to download an XML feed and
convert it to a list of records. 
 
We just need a simple component system which consists only of the fetcher
the component.  

Using a fake XML file with  `file://` url for retrieval should be
sufficient to fake it out and get some meta data entries for some
images also located in test-resources.
 
The use of fake data and a fake url avoids querying flicker for the xml 
which would be changing continuously, it can also be reused to test the rest
of the system. 

We also have spec, which can help with this.

Maybe we need one test to make sure the interface with flickr hasn't changed.

#### Testing the queue

There probably won't be much to do here if we are using a queuing library.
Make sure any additional interfaces work.
 
#### Testing the worker

Again, here we have our meta-data file to validate, to make sure the files are
created, and that the updates to the metadata file are correct.

A testing system that consists only of the worker and queue components with a fake metadata
file in the work directory using a static image from test-resources should be
sufficient to construct reasonable tests. Until it gets to the end
where integration with the database needs to be tested as well.

### testing the server

Probably the easist way to test this is to create a fake database and image folder
in test resources. After testing the fetcher, queue and worker, 
we should have the code to set this up from our test-resources snapshot.
Or just have a database snapshot sitting there.

Send the database some records and images, and then query them back out.

Using components it is also possible to create fake systems, so we can have a test-fetcher component which feeds the worker component, or a test-worker component which only validates what the fetcher component creates. Or a test-worker and test-server which can give the database a workout.

## Fault tolerance

### The Fetcher

 Here my guess is that the most likely cause of problems is timeouts. 
 Log them and adjust the cli default accordingly. Maybe it would be nice
 to have a status file that the server could read and display when it serves
 the images.  Last successful fetch, last failure, success ratio, something
 not too alarming, but informative.

[resiliance4clj](https://github.com/resilience4clj) looks like a nice way to handle this kind of thing.

### The Worker

If a worker thread fails, it is likely a timeout from the fetch of the image or 
is likely a malformed image in which case all the resizing will fail. I have
no idea if this is a real problem or not.
There are probably scenarios of which I have no idea.

In any case the job should end up back waiting in the queue to be tried again.

### The database

It seems the biggest problem here is file I/O.  I could be wrong. But an out of
disk space error seems the most likely thing.

### A monolith, or micro-services, and other improvements.

Using components would allow for this executable to behave as a monolithic
executable, each component with their own processes, but also as independent micro-services with
a recomposition of the component system definition.  

Using a durable queue allows for decoupling of the fetcher and the worker.

Using [factual's durable queue](https://github.com/Factual/durable-queue)
is one choice but there are others which use databases
like [mysql] (https://github.com/wildbit/mysql-queue)
or [postgres](https://github.com/layerware/pgqueue) or
[stockpile](https://github.com/puppetlabs/stockpile) durable queues. 

[Onyx](https://www.onyxplatform.org), [kafka](https://kafka.apache.org) or
another network based queue seems like overkill, unless it is desirable
to learn those here. Why not?  Except that it's more than I want to do 
for a code challenge.

There are numerous [pre-built components here](http://github.com/danielsz/system) 
which might be fun to play with.

I think one step at a time is a good practice. A set of folders for a database of
edn files and images is a good start, But a real database could be fun. Even
just datascript or datomic, or even hadoop and cascalog, which might be a bit slow,
but fun.

Decoupling the fetcher and worker would allow for multiple workers to be 
deployed if the work becomes overwhelming. 

To that end, I think to start I would define a nice command line interface
with clj-cli-ext keeping in mind the options that would be needed
later to grow this application in these directions. Component sub options
are easy to configure in the CLI.

 
## Conclusion

I am sure I have missed some things here. I prefer to work in a space where bouncing ideas
off of other people is the norm. I ame sure that others would have different and sometimes
better ideas. Often the best results are born of a synergy of different ideas. When I see
the design simplifying that is a good sign. As I wrote this. That is what happened. Still
I am sure with other people's input it could be better.
 



## Installation

Download from http://example.com/FIXME.

## Usage

FIXME: explanation

    $ java -jar flickr-fetcher-0.1.0-standalone.jar [args]

## Options

FIXME: listing of options this app accepts.

## Examples

...

### Bugs

...

### Any Other Sections
### That You Think
### Might be Useful

## License

Copyright © 2019 FIXME

This program and the accompanying materials are made available under the
terms of the Eclipse Public License 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0.

This Source Code may also be made available under the following Secondary
Licenses when the conditions for such availability set forth in the Eclipse
Public License, v. 2.0 are satisfied: GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or (at your
option) any later version, with the GNU Classpath Exception which is available
at https://www.gnu.org/software/classpath/license.html.
