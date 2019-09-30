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
it's simple beginnings. To that end it should use either
[component](https://github.com/stuartsierra/component) or
[mount](https://github.com/tolitius/mount). I am agnostic about the two.
I used component in it's early days and I like it just fine. It doesn't
bother me to pass the world around. Also there is enough other stuff
to play with that it seems component is a good choice. I also like
that components are easily composable such that a component could be
easily replaced by a different implementation or the component system
could be rearranged to create a family of microservices. definitions.
Component also makes it easier to test by allowing the creation of
alternative systems for testing.

## Questioning the request and Enabling more flexibility

I would like for this to be project that is flexible enough to change and grow beyond 
the original request. 

It seems that the user experience for this might not be ideal. But it is a fun project
to experiment with core.async, threading, queues, file blobs, a database, spec/schema
a server endpoint and even a UI using clojurescript and react if desired.

The request is going to be slow if it has to fetch and resize the most recent images
every time someone asks for them. Displaying what we have while new things come in is going
to be important. it would also be nice if there was a way to not repeat work, although the
doesn't have to repeat all the work, even though it might if someone asks for another size.

Requesting a size of height and width before knowing the size of each image is also going
to introduce distortion as the aspect ratio of the variously sized images is lost.
Maybe there should be an algorithm which chooses a size close to the requested size but preserves
the aspect ratio. If that is a desire.

So really it's a more interesting problem than "just fetch a number of images from the recent 
feed and convert them to the size I ask for, then show them to me." 

It would be better to ask "how can I retrieve the current feed and store the images, potentially
with some common sizes, and give those to the server when it asks while also the allowing the server
to spawn a new fetch or ask for some different sizes of images we already have." 

And maybe, we give just the image metadata to the server, and then it decides which 
images and sizes it wants based on that most recent list of images. Choices to keep in mind.
Potentially the server just gives the response to front-end ui. Future choices to think about.

The ability to continously fetch and create various sized images in the background, even if it
only does it once a day (ie. do we really want to replicate the flickr feed in it's entirety?) 
would provide better performance, similar results and a better user experience. And an architecture
capable of that should also be capable of fulfilling the original request without adding 
complexity.

Add an option to update now, or just increase the update frequency should be sufficient.
And if we really want this thing to be a feed, turn the fetcher, worker and server components 
into micro-services and let them run. Maybe with multiple workers sharing the resizing efforts.

And if not, or that we really don't care to collect all that, ok. Add a roll off feature
that removes older images that aren't flagged for saving in their meta-data files.
Another feature - save tags.  Let the database component take care of that.

Granted the UI would become more complex, with the option to view different sized files or
generate new sizes.  But then we'll want filtering and searching and what not, so this project
might as well provide a solid foundation for future growth. The difference in code is
not great. The difference in architecture from a server that fetchs and displays is.

 *The request is for this*
 * Fetch images when asked
 * Resize images only to the size asked for.
 
 *This design provides for this*
 * Fetch images by request or continually.
 * Automatically resize images after a fetch to any given variety or not.
 * Image sizes include :small, :medium and :large which preserve aspect ratio.
 * There is a :thumbnail image size of 150x150.
 * Arbitrary sizes of height and width can also be specified.
 * The fetcher, worker and server components could potentially be configured as microservices 
 as a system of components. If micro-services, the server might need a way to request new fetches.
 * The server can request a new fetch with new sizes 
 * The server can request new sizes for images already in the database.
 * Image metadata is used as a database which provides many potential possibilities.
 * The system of 5 components is composable for easier testing, component replacement,
 microservices or other improvements 


## Overview

This project seems to need at five components. Three main compontents A fetcher, a worker
and a server, and two components which all of those potentially depend upon. A queue and a database. 

The queue is used by the fetcher to provide jobs to the worker. And also potentially by
the server if the feature of resizing existing images is added.
The queue could be just a folder of edn files and a bit of code. But it's probably easier
and better to use something like [factual's durable queue](https://github.com/Factual/durable-queue).
There is a [ready made component here.](http://github.com/danielsz/system)
There are many other choices for queue implemntations available.

The database should isolate the side effects of writing the images and the
metadata, this will make testing of everything else much as easier as all
functions elsewhere should be pure. It is used by everyone. The fetcher
gives it the initial image and metadata. The worker gives it the newly
resized images. The server requests metadata and images from the database.

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

### Fetcher
 The fetcher is a simple component which fetches the recent feed, converts it
 to clojuere data, then retrieves the original images.  If a count is given
 limit the retrieval to that count. The default, like the flickr api should
 be 100 with a maximum of 500. It should give the metadata and the images
 to the database component. 
 
 If the fetcher is running in a polling fashion, it should 
 start, on compnonent start, a process (go loop) which watches a 
 channel for requests and another channel for a *stop* for graceful 
 shutdown. Writing nice 
 [core.async components](https://cb.codes/a-tutorial-of-stuart-sierras-component-for-clojure/) 
 is a well known pattern.

 The job of the fetcher is to get the list of recent image metadata from
 the flickr feed and convert it to a list of image metadata records, then fetch
 and give the metadata and original sized image to the database component to be saved.
 
 If options were given for resizing those values
 should be put into the resizes vector in each metadata record.
 Then give each metadata record to the queue component.

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
  * `:sizes []`       - Sizes that currently exist for this image. ie. 
  
 Sizes and resizes are vectors and can have any of the following as values. 
 `:thumbnail, :small, :medium, :large, :original` or any number of `[<height> <width>]`.
  
 The fetcher will be told which sizes need to be created, either by the CLI or
 by the server when the server requests a new fetch.
  
 The fetcher should be able to work in one of two modes. By request or in a continuous polling mode
 which may also need to handle a request.
  
 It is possible, that the fetcher could use a go-loop to retrieve the original images
 and send them to the database. But that might not be nice etiquete for Flickr. If it does
 use a go-loop for the goal of spawning multiple fetching processes, it should also have a 
 secondary channel *stop* so that the component can be shutdown gracefully.
 Maybe 8 processes is not so bad...
  
 Upon successful retrieval and writing of the original image file, 
 image records should have a populated sizes vector of _:original_  
 ie. `:sizes [:original]` If given on the command line or by the server's request , 
 `:resizes []` may also have other entries. These metadata records should then
 be given to the queue component for the worker to create the resized images.
 
 ### A different division of labor.

 Another possible way to divide responsibilities is to let the database do the fetching
 of the original file when it is requested.  I Like that as it simplifies the fetcher
 and puts the responsibility of the saving and delivering the original image all in one
 place.  ---  *Think on this, and Rewrite*

### Queue

A factual durable queue component. Although it is not complicated, 
[A pre-built component can be found here](http://github.com/danielsz/system) 
This component only needs to do it's job as queue.  Take image metadata files and
give image meta data files in a way that is resiliant to worker failures.

The fetcher and possible the server will give it image metadata records and the
worker will take them.


# Stopped here.........................................................

### Database

I am not sure that managing both the metadata files and the image files
should be the responsibility of this component. But at least now, this 
makes some sense and it is simple enough.

When giving an image to save,
the database component should write the image to it's _local-path_ in the
_images-path_ The image should use a filename like so. _<image-id>.original.<format-ext>_

A file based database of image metadata files. I simple component which can
take image meta data files for storage and give the most recent image metadata entries
in response to a request.

Something that might be fun would be to load the metadata files into a datascript
db at component initialization. That could provide some fun things to play with.

Optionally I suppose we could give it the responsibility of returning images when
asked for.

 The final database folder should consist of edn files named after the image
 id.  All images will resized in an images _local-path_. Image names should be 
 of the format _<id>.<size>.<format>_

 The resulting meta data record should be written to the applications _image-db_ folder
 as indicated by the CLI option `database-path`. This is a point of future improvement either
 by replacement with a real DB or sub-division of folders by some date interval.

A request for recent images should result in the last 100 images in the database.
this is the same default as Flickr. Flickr has a maximum of 500 images which is 
also probably a good limit.

### Worker

 By the time we get here, we have the original image in it's folder and a nice 
 metadata record about the image.

 If the worker should probably just be a go-loop which takes from the queue
 component.  It is probably sufficient to call a function for each image which
 then does a pmap to get the resized images. I would keep these pure. and then
 write the images and update the metadata record and write it to the database
 all in the same function, to isolate the side effects and improve testability.
 
 Like the fetcher in polling mode, the worker will need a channel for a *stop* 
 signal to enable a graceful shutdown.
 
 This component only needs to create more images sizes as requested 
 by the metadata entries in the queue component.
 If the resizes vector in an image metadata file is empty there is nothing to
 do but write the metadata file to the database directory.
 
 On error, the worker should do the appropriate things in signaling the queue
 that the job failed for that entry.
 
 In addition to arbitrary `[height width]` Possible resize values are the following.
 `:thumbnail = (150x150), :small = (1/4) :medium = (1/2) and :large = (3/4)`. 
 Each of these images should be written to the image's local path folder.
 
 Before sending the metadata entry to the database component, 
 the image's meta data record should be updated with the new sizes and if all is
 successful the _resizes_ vector will be empty.

### Server

This is pretty simple really.  Use compojure-api and spec to create a nice endpoint 
with a nice swagger api document.
The server should show the most recent images in the images directory. The server can 
get the image metadata entries from database component and then do what it wishes. 

Options to the end point are count and size. The possible
sizes are thumbnail, small, medium, large, original or an arbitrary [height width].  
if no quantity or size was given the server would simply get the 
last 100 metadata records from the database component. 

It might be nice to render the meta data along with the images
so that the links can be followed to the photographer among other things.
 
The request for this project was that the when asked, the server would cause a retrieval of
the most recent records and deliver the quantity requested in the size requested.

As designed, these components can do much more. 
For this simple behavior it is only necessary for the server to ask the fetcher 
to initiate a fetch for the given size of images. As the image metadata records 
appear in the database folder the server could then render the quantity asked for.
 
I think this behavior will be slow. I also wonder about the true desire of
being able to resize to any height or width. This will cause the loss of
aspect ratio in a somewhat random way, which the _:thumbnail_ size certainly does.
I've added the small, medium, large sizes in order to allow resizing which honors
the original aspect ratio of the images.
 
Another method would be to let the fetcher run in a polling mode such that images are
constantly being retrieved over a time interval. In this way, everything is much simpler
and server only needs to worry about serving whatever is already there. 

In this situation the server only needs to know about the meta data records and show 
the most recent images. If a size does not exist, the server can give the meta-data records 
with the appropriate resize values to the queue component, and then wait
for them to reappear in the database with their new sizes.
 
The meta-data records have a lot of additional information which allow for
the creation of a nice interface including filtering. 

If needed other options could be added such as photographer/user etc. 
 
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
    * sizes <[:thumbnail :small :medium :large [x y] ...]>
 * server
     
 
### Logging.

I've used [Timbre](https://github.com/ptaoussanis/timbre) in the past and it works well with care.

### component initialization

In main, the cli parser would be called, and the result of
the validated cli tree would be passed to the initialization
of each component in the system. 
 
### Testing

#### General

With components testing can be made much easier by creating component systems which
consist only of the components required of the test. The CLI can mocked, to create
settings which would only be used for testing. Such as using a fake URL.

A sample database and image folders to match should be created in test-resources
to test the server and the worker.

Making a snapshots of the data in test-resources, and pointing the the components 
to alternative folders or empty destinations should be done as needed.

In the case of the worker, a temporary snapshot would allow testing of a submittal
of resize work for existing images.

For testing of new work from a skeleton set of meta data files and their 
matching image folders/images it would only be necessary to point the worker at some new
database folder that would be removed before each test run.

#### Testing the fetcher
 Testing  the fetcher is not too difficult as it only has to download an image
 and create an edn file that we can validate with spec.
 
 We just need a simple component system which consists only of the fetcher
 component.  

 Using a fake XML file with  `file://` url for retrieval should be
 sufficient to fake it out and get some meta data entries for some
 images also located in test-resources.
 
 The use of fake data and a fake url avoids querying flicker for the xml 
 which would be changing continuously,
 Maybe we need one test to make sure the interface with flickr hasn't changed.
 
#### Testing the worker

Again, here we have our meta-data file to validate, to make sure the files are
created, and that the updates to the metadata file are correct.

A testing system that consists only of the worker component with a fake metadata
file in the work directory using a static image from test-resources should be
sufficient to construct reasonable tests.

### testing the server

Probably the easist way to test this is to create a fake database and image folder
in test res
This can be simplified some by using the CLI interface.  By providing default urls and behaviors in the
cli tree, which the components use for their startup, we can overide those values for testing.
The first thing that comes to mind is using a static file url which points to an XML image entry list in test resources.

Using components it is also possible to create fake systems, so we can have a test-fetcher component which feeds the worker component, or a test-worker component which only validates what the fetcher component creates.

## Fault tolerance

### The Fetcher
 Here the most likely problem is timeouts. 
 Log them and adjust the cli default accordingly. Maybe it would be nice
 to have a status file that the server could read and display when it serves
 the images.  Last successful fetch, last failure, success ratio, something
 not too alarming, but informative.

### The Worker

If a worker thread fails, it is likely a malformed image in which case
all the threads will fail, or a filesystem error. In the case of resizing
problem, the job should be re-submitted for a retry since it is likely
a malformed image. If this route is taken, keeping a count of retries in
the metadata file might be a good idea to limit the attempts. 
Also, maybe there is a checksum that we can apply to check the validity of the
image before we try to do any processing.

If the threads do the writing, which seems reasonable, the failure could be disk space
or another filesystem error.
In this case, for this simple project, it seems most reasonable to try to write the 
original edn file to a log or failure directory or both. Without a more fault tolerant queuing 
system in place that seems reasonable.  A recovery option on the CLI could recover
those edn files and submit them for reprocessing once the problems are resolved.

On the other hand, for a toy project, how much do we care if we miss a few images as
long as it doesn't continue to be chronic problem ?

Testing: The biggest problem here is likely to be malformed images or some sort of
file system error.

[resiliance4clj](https://github.com/resilience4clj) looks like a nice way to handle this kind of thing.

### A monolith, or micro-services, and other improvements.

Using components would allow for this executable to behave as a monolithic
executable, each with their own threads, but also as independent micro-services with
a recomposition of the component system definition.  

This design uses folders as a work queue which works,
But it might also be nice to use something like [factual's durable queue](https://github.com/Factual/durable-queue), or one of the other [mysql] (https://github.com/wildbit/mysql-queue), [postgres](https://github.com/layerware/pgqueue) or [stockpile](https://github.com/puppetlabs/stockpile)
durable queues. [Onyx](https://www.onyxplatform.org), [kafka](https://kafka.apache.org) or
another network based queue seems like overkill, unless it is desirable
to learn those here. Why not?  Except that it's more than I want to do 
for a code challenge.

There are numerous [pre-built components here](http://github.com/danielsz/system) 
which might be fun to play with.

I think one step at a time is a good practice. A set of folders for a database of
edn files and images is a good start,

Rearranging the components to optionally use a durable queueing system
that would allow for micro-services and better/easier fault tolerance
later on would be easy to factor in. 

Decoupling the fetcher and worker would allow for multiple workers to be 
deployed if the work becomes overwhelming. 

To that end, I think to start I would define a nice command line interface
with clj-cli-ext keeping in mind the options that would be needed
later to grow this application in these directions. Component sub options
are easy to configure in the CLI.



 
 


 
 
 



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
