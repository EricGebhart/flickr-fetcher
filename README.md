# flickr-fetcher

Flickr Fetcher is is a simple example program which fetches the most recent flicker photos
and then resizes them so that an endpoint can then serve them.

## The challenge

Implement and push to a github repo a small http service that has one endpoint which downloads,
resizes and stores images from the flicker feed
[documented here.](https://www.flickr.com/services/feeds/docs/photos_public/).

 * Able to specify the number of images, if not specified then all feed images.
 * Able to specify the resize width and height for images.
 * it is sufficient to save the resized images on the local disk.
 * Use clojure
 * It is fine to use libraries that provide http server implementations and image resizing, but
 not in one solution.
 
 *given all that, I realize I took a wrong turn and designed something more elaborate* that collects 
 images but without the ability of custom sizes.

## redesign from the design below 

 *this design differs in that*
 * It doesn't continually fetch images - _only fetch images when asked_
 * If asked to resize images it only resizes the images to the size asked for.
 * The worker component creates up to N threads and processes each image from the 
 channel until they are gone.  Each thread handles one image to resize.
 * There is no need to think of micro-services
 * There is no need to think of alternative durable queuing, a channel is sufficient.
 * The future flexibility of the system is not worth thinking about much.
 
_In reality, the ability to resize at request time could be built on top
 of the system defined below without issue_
 
 * Instead of continually fetching, the system below could be changed, perhaps with an option, 
 to only fetch on command and still do all the resizing, adding one more thread for the 
 requested size if asked for.

 * it's all configurable I suppose.

## The request

The request to the endpoint can specify a number of photos and a size.  If no number or size is given
all recent photos are given in the original size.

Using the [flickr API](https://www.flickr.com/services/developer/api/) is not necessary for this project.  A list of recent images can with the url 
[documented here.](https://www.flickr.com/services/feeds/docs/photos_public/).

This project is actually an exercise in how to queue the resizing jobs for the images.  Otherwise it
would be better to just request the sizes from flickr using the API.  Resizing is not actually necessary
as Flickr has alread done that work.


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

## A specification.

At first thought, this project needs three components. A fetcher, a worker and a server.

 * Fetcher  -  Get the list of recent image metadata from flicker and
 convert it to a list of clojure maps for the worker component.

 * Worker   -  This component needs to fetch the original image from
 the url in the job metadata. Resize the image into 4 additional sizes,
 thumbnail (150x150), small (1/4) medium (1/2) and large (3/4).  Each of
 these images should be written to a folder named after the image id,
 along with a file of the edn metadata.

 * Server   -  This component can be completely independent of the other
 two. It would simply get the last 100 images from the images folder. 100
 is the same default as Flickr, from the final images directory.
 Flickr has a maximum of 500 images which is also probably a good limit.
 If needed other options could be added such as photographer/user etc. For
 speed and versatility, the server could keep a list, ordered by date
 of the metadata files for filtering.
 
### CLI

I would use my own [clj-cli-ext](https://github.com/EricGebhart/clj-cli-ext) library to manage the command line interface.  It automates documentation
and makes it easy to make sub-commands which makes it easier to manage as a system grows in complexity.
It includes basic logging options which work well with any logging system someone might wish for.
 
### Logging.

I've used [Timbre](https://github.com/ptaoussanis/timbre) in the past and it works well with care.
 
### A monolith, or micro-services.

Using components could allow for this executable to behave as a monolithic
executable, each with their own threads, but also as independent micro-services with
a recomposition of the component system definition.  

In the monolithic configuration the fetcher and worker would use a
channel together and the server would be serving whatever happens to be
there in the filesystem when a request comes in.

But it would also be possible to run this executable
as separate executables provided that a reasonable queue between the
fetcher and worker were used.  Something like [factual's durable queue](https://github.com/Factual/durable-queue),
or one of the other [mysql] (https://github.com/wildbit/mysql-queue), [postgres](https://github.com/layerware/pgqueue) or [stockpile](https://github.com/puppetlabs/stockpile)
durable queues. [Onyx](https://www.onyxplatform.org), [kafka](https://kafka.apache.org) or
another network based queue seems like overkill, unless it is desirable
to learn those here. Why not?  Except that it's more than I want to do 
for a code challenge.

I think one step at a time is a good practice. A channel is a good start,
and rearranging the components to optionally use a durable queueing system
that would allow for micro-services and better/easier fault tolerance
later on would be easy to factor in. Decoupling the fetcher and worker
would allow for multiple workers to be deployed if the work becomes
overwhelming. 

Another choice is maybe one worker component which deploys multiple actual
worker processes each with it's 5 threads.  All based on some CLI option.

To that end, I think to start I would define a nice command line interface
with clj-cli-ext keeping in mind the options that would be needed
later to grow this application in these directions. Component sub options
are easy to configure in the CLI.


### component initialization

In main, the cli parser would be called, and the result of
the validated cli tree would be passed to the initialization
of each component in the system. 

### The Fetcher component

 My initial thought, for simplicity is to map the list of image entries
 to a channel to be picked up by the worker. That implies that they are
 related components in the same executable. Testing could be done by
 having a fake worker that could verify that the entries are as expected
 by validating against a spec.  The test could also use a fake url to
 retrieve a local file instead of querying flicker for the xml which
 would be changing continuously, which may not matter that much if all
 we care about is it's conformance to the spec. But then, idempotence,
 predictability. Maybe one test to make sure the interface with flickr
 hasn't changed.
 
 As an extension, use a durable queue like factual's durable queue or
 one of the durable queues based on postgress or redis.
 
 Those would allow the fetcher and worker(s) to be split into separate
 micro-services. Which would also be easier to test.  Otherwise we'll
 need a fake worker to test the fetcher, and a fake fetcher to test the
 worker. Durable queues add complexity in some ways, but fault tolerance
 is much easier and better.
 
 *Fault tolerance:*  Here the most likely thing is timeouts. 
 Log them and adjust the cli default accordingly. Maybe it would be nice
 to have a status file that the server could read and display when it serves
 the images.  Last successful fetch, last failure, success ratio, something
 not too alarming, but informative.

[resiliance4clj](https://github.com/resilience4clj) looks like a nice way to handle this kind of thing.


### The worker component

 I currently think it would be best to make a promise out of the request for
 the original image which is then given to each of 4 threads created for the resizing.
 The resizing threads would create a thumbnail (150x150), small (1/4), 
 medium (1/2) and large (3/4) sized images.  The original image could be 
 specified as a promise for the other 4 threads. These would be written 
 to a folder in the designated images folder using the image id as the 
 folder name.  The image metadata edn would be written along with the images.
 
 I am not sure of the wisdom of just adding these images as data to the metadata
 file, verses writing them directly to the filesystem with a name something
 like this. _<image-id>.<size>.jpg_ 
 
 
#### fault tolerance

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
 
### The server component 

This is pretty simple really.  Use compojure-api and spec to create a nice endpoint which
returns the most recent images in images directory.  Options are count and size. The possible
sizes are thumbnail, small, medium, large and original.  The default count is 100, the max is 500,
and the default size is original.  It might be nice to render the meta data along with the images
so that the links can be followed to the photographer among other things.

### CLI options

 * General - *Path* to images.  *Log file*, *log verbosity* (builtin to clj-cli-ext). 
 Path to write failed edn jobs.
 * Fetcher - Request *timeout* and *time between fetching*.
 * worker  - Number of processes ? recovery ?
 * server  - 

### testing

This can be simplified some by using the CLI interface.  By providing default urls and behaviors in the
cli tree, which the components use for their startup, we can overide those values for testing.
The first thing that comes to mind is using a static file url which points to an XML image entry list in test resources.

Using components it is also possible to create fake systems, so we can have a test-fetcher component which feeds the worker component, or a test-worker component which only validates what the fetcher component creates.

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
