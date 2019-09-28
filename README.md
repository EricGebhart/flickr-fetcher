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
 
## initial thoughts.

The method used for resizing is not important. A webservice or one of many clojure libraries can be
used.  Likewise the server side does not seem to be so important, as it is possible to get an endpoint
with these simple requirements running quite quickly with less than 100 lines of code.    

For me, a project like this is meant for exploration and learning. I've done a lot of work with
[prismatic/plumatic schema](https://github.com/plumatic/schema) and I'm interested in learning more about 
[spec](https://clojure.org/guides/spec), so that seems like a good choice
if for only that reason.  I've done simple compojure servers before, I like compojure, and I like
using swagger for documentation purposes. I've not used [compojure-api](https://github.com/metosin/compojure-api) which has both and with the
the new [2.0 alpha](https://github.com/metosin/c2) version it is possible to use spec instead of schema, so why not.

Finally, I want this to be a system which can grow beyond it's simple beginnings. To that end it should
use either [component](https://github.com/stuartsierra/component) or [mount](https://github.com/tolitius/mount). I am agnostic about the two.  I used component in it's early days
and I like it just fine. It doesn't bother me to pass the world around.  I also like that components are
easily composable such that if I decide later that this should be a family of microservices running independently it would be easy to make that happen with some component modifications and set of different system 
definitions.  Component also makes it easier to test by allowing the creation of alternative systems for testing.

## Questioning the request and Enabling more flexibility

I would like for this to be project that is flexible enough to change when the realization that
the original request is not particularly useful. It's not that the request is horribly flawed, 
it really depends upon the goals of the project. For someone else, it may be they just want to 
see how to handle the resizing of multiple images using channels and threads. But the reality of
using this application might not be ideal. But a good core.async experiment at any rate.

It seems that the user experience for this might not be ideal.
The request is going to be slow if it has to fetch and resize the most recent images
every time someone asks for them.  Maybe there could be some cacheing system so it
doesn't have to repeat all the work, even though it might if someone asks for another size.
Why not just create a better architecture to answer a bigger question while answering this
simpler but actually more difficult question ?  It's only more difficult if I were to
restrict myself to answering this challenge exacly within the request.

If asked to resize all 100 of the most recent images to some odd size that will be extra slow
and also cause distortion as the aspect ratio of the variously sized images is lost.

The ability to continously fetch and create various sized images in the background, even if it
only does it once a day (ie. do we really want to replicate the flickr feed in it's entirety?) 
would provide better performance, similar results and a better user experience.

Add an option to update now, or just increase the update frequency should be sufficient.
And if we really want this thing to be a feed, turn the 3 components into micro-services
and let them run. Maybe with multiple workers sharing the resizing efforts.

And if not, or that we really don't care to collect all that, ok. Add a roll off feature
that removes older images that aren't flagged for saving in their meta-data files.
Another feature - save tags.

Granted the UI would become more complex, with the option to view different sized files or
generate new sizes.  But then we'll want filtering and searching and what not, so this project
might as well provide a solid foundation for future growth. The difference in code is
not great. The difference in architecture is.

 *The request is for this*
 * Fetch images when asked
 * Resize images only to the size asked for.
 
 *This design provides for this*
 * Fetch images by request or continually.
 * Automatically resize images after a fetch to any given variety or not.
 * Image sizes include :small, :medium and :large which preserve aspect ratio.
 * There is a :thumbnail image size of 150x150.
 * The fetcher, worker and server components can run idependantly as microservices or together
 as a system of components. If micro-services, the server might need a way to request new fetches.
 * The server can request a new fetch with new sizes 
 * The server can request new sizes for images already in the database.
 just by putting an images meta data file in the work queue folder with a resize value or values.
 * Image meta data is preserved so it can be used for a better UI in the future.


## A specification.

This project seems to need at least three components. A fetcher, a worker and a server.

### Fetcher
 The fetcher is a simple component which only fetches the image
 metadata and the original image that it points to.
 
 If the fetcher is running in a polling fashion, it should 
 start, on compnonent start, a thread which watches a channel for requests 
 and another channel for a *stop* for graceful shutdown.
 
 The image should use a filename like so. _<image-id>.original.<format-ext>_

 Get the list of recent image metadata from flicker and
 convert it to a list of image metadata records, then
 fetch and save the original image.
 If options were given for resizing merge those values
 into the resizes vector in each metadata record.
 Write the metadata record to the work directory.
 The naming should be _<image-id>.edn_

 The image metadata record format should be specified in spec
 and is a reflection of the entry record retrieved from Flickr, but with the
 following additional fields.
 
  * `:local-path nil` - this is the location of the image's folder.
  as determined by the CLI's _image-path_ option using the format _<path>/<image id>_
  * `:resizes []`     - Sizes that need to be created from this image.
  * `:sizes []`       - Sizes that currently exist for this image. ie. 
  
 Sizes and resizes are vectors and can have any of the following as values. 
 `:thumbnail, :small, :medium, :large, :original` or any number of `[x y]`.
  
 The fetcher will be told which sizes need to be created, either on the CLI or
 by the server when the server requests a new fetch.
  
 The fetcher can work in one of two modes. By request or in a continuous polling mode.
  
 It is possible, that the fetcher could use a channel and threads to retrieve the original images
 and write them to the local path. But that might not be nice etiquete for Flickr. If it does
 use a channel for the goal of spawning multiple fetching threads, it should also have a 
 secondary channel *stop* so that the component can be shutdown gracefully.
  
 Upon successful retrieval and writing of the original image file, the image entry should
 reflect the local path and the sizes vector should consist of _:original_  ie. `:sizes [:original]`
 If given on the command line or by the server's request , `:sizes []` may also have
 other entries.  These records should then be written as edn files to the systems work
 directory.  And the worker component should be asked to do work if it is not
 running as a micro service.

 It would also be possible to put them in a channel for the worker to
 recieve. Which is pretty nice, The server could even use the same channel
 to ask for more sizes.

 But channels imply that they are related components in the same executable. 
 So maybe not at this time. The file system is simple and allows for separate
 services. Filesystem folders are simple and it would be easy to replace them 
 with a durable queue later.


### Worker

 By the time we get here, we have the original image in it's folder and a nice 
 metadata record about the image.

 If the worker is running in a polling fashion, it should 
 start, on compnonent start, a thread which watches a channel for requests 
 and another channel for a *stop* for graceful shutdown.
 
 But that all depends on how it is decided to do the plumbing here. It may
 only need the channel for a graceful stop if all it does is watch the
 work folder/queue.

 This component only needs to create more images sizes as requested 
 by the image meta data records in the application's work directory. 
 If the resizes vector in an image metadata file is empty there is nothing to
 do but write the metadata file to the database directory, and remove it from
 the work directory.

 If running as a micro-service it should continually look in the application's 
 work folder to see if there is any work to be done.
 
 It should also be possible to tell the component to look for work.

 It should load the work records and place them on a channel for multiple resizing 
 threads to access. Each of these threads may spawn additional resizing threads as
 needed.  The return will be a image meta data record with updated sizes and resizes entries
 and the side effect is that those images will now reside in the image's _local-path_ folder
 inside the _images-path_ folder as specified by the CLI.
 Image names should be of the format _<id>.<size>.<format>_

 The worker will put the records on a channel and spawn multiple threads as needed
 to create the new resized images. The labeled sizes corresponding to the following.
 `:thumbnail = (150x150), :small = (1/4) :medium = (1/2) and :large = (3/4)`. 
 Each of these images should be written to the image's local path folder.
 The image's meta data record should be updated with the new sizes and if all is
 successful the _resizes_ vector will be empty.
 
 A secondary *stop* channel should be used for graceful shutdown of the worker component.
 
 The resulting meta data record should be written to the applications _image-db_ folder
 as indicated by the CLI option `database-path`. This is a point of future improvement either
 by replacement with a real DB or sub-division of folders by some date interval.

### Server

 This component can work independently of the other two, simply
 by loading the meta data records and providing an interface to the images.

This is pretty simple really.  Use compojure-api and spec to create a nice endpoint 
with a nice swagger api document.
The server should show the most recent images in the images directory. The server can 
get the image metadata files from the database folder and then do what it wishes. 
Sort, filter, display, etc.

Options to the end point are count and size. The possible
sizes are thumbnail, small, medium, large, original or an arbitrary [height width].  
if no quantity or size was given the server would simply get the 
last 100 metadata records from the database folder. 100
is the same default as Flickr. Flickr has a maximum of 500 images which is 
also probably a good limit.

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
 
Another method would be to let the fetcher and worker run in a polling mode,
possibly as separate micro-services.
 
In this situation the server only needs to know about the meta data records and show 
the most recent images. If a size does not exist, the server can write the meta-data records 
with the appropriate resize values to the work directory, and then wait
for them to reappear in the database with their new sizes.
 
The meta-data records have a lot of additional information which allow for
the creation of a nice interface including filtering. 

If needed other options could be added such as photographer/user etc. 
 
### CLI

I would use my own [clj-cli-ext](https://github.com/EricGebhart/clj-cli-ext) library to manage the command line interface.  It automates documentation
and makes it easy to make sub-commands which makes it easier to manage as a system grows in complexity.
It includes basic logging options which work well with any logging system someone might wish for.
Reasonable defaults should be used where appropriate.

#### Options

 * image-path
 * database-path
 * work-path
 * sizes <[:thumbnail :small :medium :large [x y] ...]>
 * log       - automatic with clj-cli-ext.
   * file
   * verbosity
 * help      - automatic with clj-cli-ext.
 * Fetcher
   * url 
   * timeout
   * polling on/off
    * interval <time in seconds between fetches>
 * Worker
   * polling on/off
     * interval <time in seconds between looking for work>
     * process-count ?
     
 
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
