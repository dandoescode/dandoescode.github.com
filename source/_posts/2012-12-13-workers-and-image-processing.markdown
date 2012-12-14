---
layout: post
title: "Workers and Image Processing"
date: 2012-12-13 20:24
comments: true
categories: [Python, Development Workers, Kapture]
---

## A Litte Background

In my second week of working at [Kapture](http://kaptu.re), I was tasked with moving our image processing over to a worker process to help speed up the process.  It wasn't THAT slow, but we noticed certain scenarios which caused Kaptures to take longer than they should.  We boiled it down to (mainly) the image processing.

So here is the basic flow of the way Kaptures USED to work so you can understand what is going on:

1. The user takes a photo, enters the info and it starts to upload the photo via an API request, then waits for a response from the server.
2. The API resizes it to the proper size.
3. Inserts the photo into the Merchant's Polaroid frame.
4. Makes a thumbnail.
5. Uploads the resulting image and thumbnail to S3.
6. Enter the new Kapture info into the database.
7. Send Response
8. User continues with Redemption

As you can see, there is a lot going on there.  This really isn't bad most of the time, but sometimes it can get bogged down with the image resizing (if the server is under load), or S3 can be going a bit slow.

So, what to do?  It is simple, offload it to a background process, right?  Well, yes, that would work, however, that still adds load to the API server and slows down other requests.  Our solution was to offload it to a worker process on a different server.  We already had a workers server, so it was no big deal for us (not going into setting up the Queue and such).

## Which Language?

We could write the worker in any language we wanted, so which did we choose?  Python.

We did this for two main reasons:

* Python is better than PHP for long running processes.
* Python has the awesome PIL library for image manipulation.

Now that we have that sorted, let's figure out how we want things to work.

## What Should the End Result Be?

It is always a good idea to plan things out before you just go Cowboy Coding.  Here is what we came up with:

1.  The user takes a photo, enters the info and the app sends the API the photo and other data, and waits for a response.
2.  Enter the new Kapture into the database.
3.  Send the photo (and some metadata) off to the Queue.
4.  Send Response
5.  User continues with Redemption

We took 3 huge steps out of the process.  Now it is time to code.

## Making it Work

### Get it to the Queue

The first thing to figure out is how to send the user's photo over to the worker.  We use RabbitMQ for our Queue server, so we could just send the raw image data in the message we send the Queue.  However, once you start to get into creating clusters of Queue servers, this will cause slow downs, and in some cases will cause some nodes in the cluster to incorrectly be taken out due to heartbeat timeouts (it is an Erlang thing).

We decided to shove the raw image data into our Redis server, then just send the Key along with our other data in the message.  This is done fairly simply:

``` php
<?php
$redis = new Predis\Client(array(
    'host' => Config::get('redis.host'),
    'port' => Config::get('redis.port'),
    'database' => 4
));

$redis->set('photo:'.$image_name, file_get_contents($image_file));
```

*Note: `$image_name` is guaranteed to be unique, so no chance of collisions here.*

What's next?  Yup, we need to send the message to the Queue server so the worker can, well, get to work.

``` php
<?php
Rabbit::enque('image_proc', 'polaroid', [
    // Some other data you can't see :P
    'image_name'    => $image_name,
]);
```

`Rabbit::enque` is a class and method we have written to make sending stuff to our Queue easy.  There are plenty of tutorials and code around to show you how to do this, so I won't bore you with that.  Just know that the first parameter is the Exchange name, and the second is the Queue name.

### The Worker

For the worker, like I said earlier, we are using Python.  The libraries we are using are as follows:

* kombu - The AMQP Library
* boto - S3 Stuff
* PIL - Image Library
* redis - Redis Library
* requests - To make Requests to the API

I am not going to post our exact code (obviously), but here is the basic setup we have to subscribe to the Queue.  This is a little more complicated than it needs to be because we will be adding more workers.

#### runner.py

``` python
from kombu import Connection
from kombu.utils.debug import setup_logging
from workers import PolaroidWorker

setup_logging(loglevel='INFO')

# Here is where we get our configuration and do a few other things

# Create the connection to the Queue server.
# We use a with statement here so the connection is auto-closed
with Connection(hostname=config['host'],
                port=config['port'],
                userid=config['user'],
                password=config['pass'],
                virtual_host=config['vhost']) as conn:
    try:
        PolaroidWorker(conn, config).run()
    except KeyboardInterrupt:
        print('Exiting')
```

I have omitted the getting of our config for security purposes, but just know it is a JSON file that we grab and parse.

This file is pretty simple, it just makes a Connection to the Queue server, then creates a new PolaroidWorker (more below), and runs it.

#### queues.py

``` python
from kombu import Exchange, Queue

task_exchange = Exchange('image_proc', type='direct')
queues = {
    'polaroid': Queue('polaroid', task_exchange)
}
```

This file defines our Exchange and then a Dictionary of Queues.  These are used later in our Workers.

#### workers.py

``` python
from kombu.mixins import ConsumerMixin
from kombu.log import get_logger
from queues import queues

# S3 Stuff
from boto.s3.connection import S3Connection
from boto.s3.key import Key

# Everything else we need
import Image, cStringIO, os, redis, urllib

logger = get_logger(__name__)

class BaseWorker(ConsumerMixin):

    # The name of the queue that this worker consumes (see queues.py)
    worker_queue = None

    def __init__(self, connection, config):
        print "Worker Init"
        self.connection = connection
        self.config = config

    def get_consumers(self, Consumer, channel):
        return [Consumer(queues=queues[self.worker_queue],
                         callbacks=[self.process])]

class PolaroidWorker(BaseWorker):

    worker_queue = 'polaroid'

    def process(self, body, message):
        try:
            # This is where you process the message from the Queue
        except Exception, e:
            print "[%s] Processing Raised Exception: %s" % (body['id'], exc)
        # Acknowledge the message to the Queue
        message.ack()
```

So, things get a bit more complicated here.  `BaseWorker` extends the `ConsumerMixin` which is provided by kombu to handle all of the "scaffolding" type things.  `get_consumers` is called automatically when the worker starts up (part of the `ConsumerMixin` class).  It returns an array of Consumers that will handle messages that this workers is supposed to consume.  As you can see we send it the Queue we defined in the `queues` Dictionary we defined in `queues.py`, then set the callback to be the `process` method of the worker class.

Next, we define the worker where we actually get things done.  All it defines is which `worker_queue` it is responsible for, then defines the `process` method.  We put all the processing in a try/catch so that we can continue on, and just dump out the exception if there is one.  Finally we acknowledge the message so that we don't re-process it.

## The Image Processing

There is a lot thet goes into this, and I can't show you all of the code, but here is a little tid-bit.

``` python
orig_tmp_file = '/tmp/orig_' + body['image_name'] + '.png'
tmp_file = '/tmp/' + body['image_name'] + '.png'

# Open the Redis connection
r = redis.StrictRedis(host=self.config['redis-host'], port=self.config['redis-port'], db=4)

# Gets then Deletes the photo from Redis
raw_photo = r.get('photo:' + body['image_name'])
r.delete('photo:' + body['image_name'])     

# Create a string buffer then open it as an image
# Note: cStringIO is much faster than StringIO, hence the usage
photo = Image.open(cStringIO.StringIO(raw_photo))

# Save the original so we can upload it later
photo.save(orig_tmp_file)

# Open the Polaroid Frame image, load it into PIL then close it.
f = urllib.urlopen(polaroid_url)
frame = Image.open(cStringIO.StringIO(f.read()))
f.close()

# Resize the image, and add the User Photo to the frame
# all in one go, then save it to the temp file
frame.paste(photo.resize((586, 497)), (13, 13))
frame.save(tmp_file)

# Get the images up to S3
framed_url = self.upload_image(s3_path_would_go_here, tmp_file)
image_url = self.upload_image(other_s3_path_would_go_here, orig_tmp_file)

# This is where we would send an API Request with all the info it needs to 
# update the records and such.

# Close the Redis connection, we don't want to leave open connections
r.connection_pool.disconnect()

# Delete the temp files
os.unlink(tmp_file)
os.unlink(orig_tmp_file)

# Some other stuff to free up memory and such that I won't bore you with
```

**Note: Some of the variable names and other stuff has been changed.**

It looks like a lot of code, but if you take out the comments, it isn't bad.

## Uploading to S3

In the above code we call `self.upload_image`, so here it is:

``` python
def upload_image(self, key_path, file_path):
    """Uploads the given image to S3 at the given key Path."""

    # Connect to S3 if we haven't already
    if self.s3 == None:
        self.s3 = S3Connection(self.config['aws']['key'], self.config['aws']['secret'])

    # Get the S3 bucket
    bucket = self.s3.get_bucket(self.config['aws']['s3_bucket'])

    # Determine the Key (object) Path and create it
    k = Key(bucket, key_path)

    # Get a file pointer to the temp file then send it to S3
    image = open(file_path)
    k.set_contents_from_file(image)
    image.close()

    # Make the image public
    k.make_public()

    return url #We build the URL here, but I can't show you that.
```

Pretty simple right?

## Did it Work?

Yes! Very well.  
