---
layout: post
title:  "New Feature: Timeline on Cruise Pages"
date:   2016-01-11 15:21:00 -0800
categories: cchdo feature
author: "Andrew Barna"
---
Regular users of CCHDO probably noticed a redesigned cruise page.
The new page contains all the content of the old one, and some new
content to go along with it. The primary focus of the redesign was on
data and data history. Here are some of the highlights:

* The dataset files have been made the primary focus of the page. They
    are now right at the top of the page and are BIG and noticable. Also
    returning is the hierarchical layout from previous generations of
    the CCHDO website.
* The Big Green Download Data button has been made bigger and greener.
    OK, so it wasn't made greener, but was moved to be more obvious.
    Clicking that button immediately sends you all the files in the
    dataset of the cruise (more on how we do that in a later post).
* Descriptions of what "Unmerged Data" means. One of the biggest
    problems a data user can have is picking which data file they need.
    The previous cruise page, while meaningful to those of us at
    the CCHDO, was causing confusion.
* File events in the data history. This was the most requested feature
    we received after that major website overhaul in 2015-04. When we
    relaunched the CCHDO website in April last year, perhaps the largest
    feature missing was the ability to see when things were happening to
    the data files. Things like submission dates, when files were
    merged with the main dataset, all were absent from the website.

## File Events and Data History History

CCHDO has always maintained records of data processing notes. These are
the, often cryptic, blobs of text that has appeared under the links to
data since we've had a website. To make sense of these messages, you
usually needed direct access to the file system the data was stored in.
Things were simple, no duplicate file names to worry about, notes
appeared next to the file they were about, there were actually FILES
laying around a file system.

At some point, databases started running the backends of the CCHDO web
presence. History notes were still text, but not longer files. Instead,
the rows of a database table contained the history notes, this tied the
note to the cruise, but decoupled it slightly from the file the note was
about.

Soon, files themselves had entries in the database, not the binary data
itself, but metadata. We started to record when certain events were
happening to a file, submission date, merged into data set date, any
notes specific to some exact file. 

Several problems were noted with the maintainance of this database: 

* The file tables were fragemented by
    program, and file status. Seperate tables existed for files from
    submissions, files which were unprocessed, and files in the dataset.
    Even when the file itself was not changing.
* The database was being edited almost entirely by hand.
* Despite being a relational database, none of the integrity features
    offered were being used (e.g. no foreign key constraints).

The attempt to solve the above problems destablilized a few weeks after
it launched. Perhaps more on that story in a future post. What resulted
was the lean CCHDO site seen today. Due to the speedy nature in which it
was built, a minimum of the most important features was the focus. Now
that we are stable (and have had almost a year of stability), more
complicated features are starting to be added. Including the data
history timeline.

## How the Timeline Works

Cruise and File metadata at CCHDO are stored in a JSON format. In fact,
there are only two objects we keep track of:

  * The Cruise JSON Object (1 per cruise)
  * The File JSON Object (1 per file)

Everything you see on the cruise page is currently constructed from one
object representing the cruise metadata, and many objects representing
the files attached to that cruise.

The Cruise JSON object contains the data processing history notes, the
File JSON objects contained only the submission record for that file,
but only if it was submitted via the website. Deciding on how to store
the information related to file events was tricky. The first way
suggested was to have some optional keys inside the JSON which were for
specific events:

{% highlight json %}
{
  "merged": "2015-01-01",
  "submitted": "2014-12-10"
}
{% endhighlight %}

This was determined to be too restrictive, we couldn't think of
everything which might happen to a file so it was decided to have an
array of generic "event" objects within the File JSON.

{% highlight json %}
{
  "events": [
    {
      "type": "submission",
      "date": "2014-12-10T00:00:00Z",
      "name": "Andrew Barna",
      "notes": "some notes in this string"
    }
  ]
}
{% endhighlight %}

When the website loads all the file information, it combines all the
file events with the cruise history notes. These combined objects are
then sorted by date and passed to the view template for rendering.

## Reconstructing History

Having a place to put the event records is only part of the solution.
We also need to actually have some data to put there. Reconstructing
this history proved to be the more time consuming task.

There had been (at least) three databases since the website became
database powered. Reconstructing the history of any file meant going
through these old databases. The earliest database was easy, there was a
single table which contained the submit date, and merge date for most
files. However, the old table contained only a file path. To match the
old metadata with the current system was simply hashing the old file and
looking for that hash in the current system.

For the later events, since 2014-11, we had to get clever with the
metadata history. The CCHDO databsae maintains a record of every JSON
object which has ever existed within it. It also will provide a list of
timestamped [JSON Patches](https://tools.ietf.org/html/rfc6902) that
have been applied to an object (cruise or file). To reconstruct the
history of changes which should also be recorded as events, meant
looking through the list of patches for changes we cared about. For
example, if a patch was applied which changed the role of a file from
"unprocessed" to "merged", then that was a sure sign we should add a
merge event to the file.

After the various migration scripts were written and tested, it took
about 5 minutes to loop through all the file metadata records to update
them. We have about 12k files, so this meant about 40 files/second being
updated. Since the updates were done using an HTTP API, and multiple
round trips were involved, I'm pleased with the performance.

Oh... also we have an API. Soon to be public.

-Barna
