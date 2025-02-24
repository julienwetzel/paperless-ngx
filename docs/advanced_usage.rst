***************
Advanced topics
***************

Paperless offers a couple features that automate certain tasks and make your life
easier.

.. _advanced-matching:

Matching tags, correspondents, document types, and storage paths
################################################################

Paperless will compare the matching algorithms defined by every tag, correspondent,
document type, and storage path in your database to see if they apply to the text
in a document. In other words, if you define a tag called ``Home Utility``
that had a ``match`` property of ``bc hydro`` and a ``matching_algorithm`` of
``literal``, Paperless will automatically tag your newly-consumed document with
your ``Home Utility`` tag so long as the text ``bc hydro`` appears in the body
of the document somewhere.

The matching logic is quite powerful. It supports searching the text of your
document with different algorithms, and as such, some experimentation may be
necessary to get things right.

In order to have a tag, correspondent, document type, or storage path assigned
automatically to newly consumed documents, assign a match and matching algorithm
using the web interface. These settings define when to assign tags, correspondents,
document types, and storage paths to documents.

The following algorithms are available:

* **Any:** Looks for any occurrence of any word provided in match in the PDF.
  If you define the match as ``Bank1 Bank2``, it will match documents containing
  either of these terms.
* **All:** Requires that every word provided appears in the PDF, albeit not in the
  order provided.
* **Literal:** Matches only if the match appears exactly as provided (i.e. preserve ordering) in the PDF.
* **Regular expression:** Parses the match as a regular expression and tries to
  find a match within the document.
* **Fuzzy match:** I don't know. Look at the source.
* **Auto:** Tries to automatically match new documents. This does not require you
  to set a match. See the notes below.

When using the *any* or *all* matching algorithms, you can search for terms
that consist of multiple words by enclosing them in double quotes. For example,
defining a match text of ``"Bank of America" BofA`` using the *any* algorithm,
will match documents that contain either "Bank of America" or "BofA", but will
not match documents containing "Bank of South America".

Then just save your tag, correspondent, document type, or storage path and run
another document through the consumer.  Once complete, you should see the
newly-created document, automatically tagged with the appropriate data.


.. _advanced-automatic_matching:

Automatic matching
==================

Paperless-ngx comes with a new matching algorithm called *Auto*. This matching
algorithm tries to assign tags, correspondents, document types, and storage paths
to your documents based on how you have already assigned these on existing documents.
It uses a neural network under the hood.

If, for example, all your bank statements of your account 123 at the Bank of
America are tagged with the tag "bofa_123" and the matching algorithm of this
tag is set to *Auto*, this neural network will examine your documents and
automatically learn when to assign this tag.

Paperless tries to hide much of the involved complexity with this approach.
However, there are a couple caveats you need to keep in mind when using this
feature:

* Changes to your documents are not immediately reflected by the matching
  algorithm. The neural network needs to be *trained* on your documents after
  changes. Paperless periodically (default: once each hour) checks for changes
  and does this automatically for you.
* The Auto matching algorithm only takes documents into account which are NOT
  placed in your inbox (i.e. have any inbox tags assigned to them). This ensures
  that the neural network only learns from documents which you have correctly
  tagged before.
* The matching algorithm can only work if there is a correlation between the
  tag, correspondent, document type, or storage path and the document itself.
  Your bank statements usually contain your bank account number and the name
  of the bank, so this works reasonably well, However, tags such as "TODO"
  cannot be automatically assigned.
* The matching algorithm needs a reasonable number of documents to identify when
  to assign tags, correspondents, storage paths, and types. If one out of a
  thousand documents has the correspondent "Very obscure web shop I bought
  something five years ago", it will probably not assign this correspondent
  automatically if you buy something from them again. The more documents, the better.
* Paperless also needs a reasonable amount of negative examples to decide when
  not to assign a certain tag, correspondent, document type, or storage path. This will
  usually be the case as you start filling up paperless with documents.
  Example: If all your documents are either from "Webshop" and "Bank", paperless
  will assign one of these correspondents to ANY new document, if both are set
  to automatic matching.

Hooking into the consumption process
####################################

Sometimes you may want to do something arbitrary whenever a document is
consumed.  Rather than try to predict what you may want to do, Paperless lets
you execute scripts of your own choosing just before or after a document is
consumed using a couple simple hooks.

Just write a script, put it somewhere that Paperless can read & execute, and
then put the path to that script in ``paperless.conf`` or ``docker-compose.env`` with the variable name
of either ``PAPERLESS_PRE_CONSUME_SCRIPT`` or
``PAPERLESS_POST_CONSUME_SCRIPT``.

.. important::

    These scripts are executed in a **blocking** process, which means that if
    a script takes a long time to run, it can significantly slow down your
    document consumption flow.  If you want things to run asynchronously,
    you'll have to fork the process in your script and exit.


Pre-consumption script
======================

Executed after the consumer sees a new document in the consumption folder, but
before any processing of the document is performed. This script can access the
following relevant environment variables set:

* ``DOCUMENT_SOURCE_PATH``

A simple but common example for this would be creating a simple script like
this:

``/usr/local/bin/ocr-pdf``

.. code:: bash

    #!/usr/bin/env bash
    pdf2pdfocr.py -i ${DOCUMENT_SOURCE_PATH}

``/etc/paperless.conf``

.. code:: bash

    ...
    PAPERLESS_PRE_CONSUME_SCRIPT="/usr/local/bin/ocr-pdf"
    ...

This will pass the path to the document about to be consumed to ``/usr/local/bin/ocr-pdf``,
which will in turn call `pdf2pdfocr.py`_ on your document, which will then
overwrite the file with an OCR'd version of the file and exit.  At which point,
the consumption process will begin with the newly modified file.

The script's stdout and stderr will be logged line by line to the webserver log, along
with the exit code of the script.

.. _pdf2pdfocr.py: https://github.com/LeoFCardoso/pdf2pdfocr

.. _advanced-post_consume_script:

Post-consumption script
=======================

Executed after the consumer has successfully processed a document and has moved it
into paperless. It receives the following environment variables:

* ``DOCUMENT_ID``
* ``DOCUMENT_FILE_NAME``
* ``DOCUMENT_CREATED``
* ``DOCUMENT_MODIFIED``
* ``DOCUMENT_ADDED``
* ``DOCUMENT_SOURCE_PATH``
* ``DOCUMENT_ARCHIVE_PATH``
* ``DOCUMENT_THUMBNAIL_PATH``
* ``DOCUMENT_DOWNLOAD_URL``
* ``DOCUMENT_THUMBNAIL_URL``
* ``DOCUMENT_CORRESPONDENT``
* ``DOCUMENT_TAGS``
* ``DOCUMENT_ORIGINAL_FILENAME``

The script can be in any language, but for a simple shell script
example, you can take a look at `post-consumption-example.sh`_ in this project.

The post consumption script cannot cancel the consumption process.

The script's stdout and stderr will be logged line by line to the webserver log, along
with the exit code of the script.


Docker
------
Assumed you have ``/home/foo/paperless-ngx/scripts/post-consumption-example.sh``.

You can pass that script into the consumer container via a host mount in your ``docker-compose.yml``.

.. code:: bash

  ...
  consumer:
    ...
    volumes:
      ...
      - /home/paperless-ngx/scripts:/path/in/container/scripts/
  ...

Example (docker-compose.yml): ``- /home/foo/paperless-ngx/scripts:/usr/src/paperless/scripts``

which in turn requires the variable ``PAPERLESS_POST_CONSUME_SCRIPT`` in ``docker-compose.env``  to point to ``/path/in/container/scripts/post-consumption-example.sh``.

Example (docker-compose.env): ``PAPERLESS_POST_CONSUME_SCRIPT=/usr/src/paperless/scripts/post-consumption-example.sh``

Troubleshooting:

- Monitor the docker-compose log ``cd ~/paperless-ngx; docker-compose logs -f``
- Check your script's permission e.g. in case of permission error ``sudo chmod 755 post-consumption-example.sh``
- Pipe your scripts's output to a log file e.g. ``echo "${DOCUMENT_ID}" | tee --append /usr/src/paperless/scripts/post-consumption-example.log``

.. _post-consumption-example.sh: https://github.com/paperless-ngx/paperless-ngx/blob/main/scripts/post-consumption-example.sh

.. _advanced-file_name_handling:

File name handling
##################

By default, paperless stores your documents in the media directory and renames them
using the identifier which it has assigned to each document. You will end up getting
files like ``0000123.pdf`` in your media directory. This isn't necessarily a bad
thing, because you normally don't have to access these files manually. However, if
you wish to name your files differently, you can do that by adjusting the
``PAPERLESS_FILENAME_FORMAT`` configuration option. Paperless adds the correct
file extension e.g. ``.pdf``, ``.jpg`` automatically.

This variable allows you to configure the filename (folders are allowed) using
placeholders. For example, configuring this to

.. code:: bash

    PAPERLESS_FILENAME_FORMAT={created_year}/{correspondent}/{title}

will create a directory structure as follows:

.. code::

    2019/
      My bank/
        Statement January.pdf
        Statement February.pdf
    2020/
      My bank/
        Statement January.pdf
        Letter.pdf
        Letter_01.pdf
      Shoe store/
        My new shoes.pdf

.. danger::

    Do not manually move your files in the media folder. Paperless remembers the
    last filename a document was stored as. If you do rename a file, paperless will
    report your files as missing and won't be able to find them.

Paperless provides the following placeholders within filenames:

* ``{asn}``: The archive serial number of the document, or "none".
* ``{correspondent}``: The name of the correspondent, or "none".
* ``{document_type}``: The name of the document type, or "none".
* ``{tag_list}``: A comma separated list of all tags assigned to the document.
* ``{title}``: The title of the document.
* ``{created}``: The full date (ISO format) the document was created.
* ``{created_year}``: Year created only, formatted as the year with century.
* ``{created_year_short}``: Year created only, formatted as the year without century, zero padded.
* ``{created_month}``: Month created only (number 01-12).
* ``{created_month_name}``: Month created name, as per locale
* ``{created_month_name_short}``: Month created abbreviated name, as per locale
* ``{created_day}``: Day created only (number 01-31).
* ``{added}``: The full date (ISO format) the document was added to paperless.
* ``{added_year}``: Year added only.
* ``{added_year_short}``: Year added only, formatted as the year without century, zero padded.
* ``{added_month}``: Month added only (number 01-12).
* ``{added_month_name}``: Month added name, as per locale
* ``{added_month_name_short}``: Month added abbreviated name, as per locale
* ``{added_day}``: Day added only (number 01-31).


Paperless will try to conserve the information from your database as much as possible.
However, some characters that you can use in document titles and correspondent names (such
as ``: \ /`` and a couple more) are not allowed in filenames and will be replaced with dashes.

If paperless detects that two documents share the same filename, paperless will automatically
append ``_01``, ``_02``, etc to the filename. This happens if all the placeholders in a filename
evaluate to the same value.

.. hint::
    You can affect how empty placeholders are treated by changing the following setting to
    `true`.

    .. code::

        PAPERLESS_FILENAME_FORMAT_REMOVE_NONE=True

    Doing this results in all empty placeholders resolving to "" instead of "none" as stated above.
    Spaces before empty placeholders are removed as well, empty directories are omitted.

.. hint::

    Paperless checks the filename of a document whenever it is saved. Therefore,
    you need to update the filenames of your documents and move them after altering
    this setting by invoking the :ref:`document renamer <utilities-renamer>`.

.. warning::

    Make absolutely sure you get the spelling of the placeholders right, or else
    paperless will use the default naming scheme instead.

.. caution::

    As of now, you could totally tell paperless to store your files anywhere outside
    the media directory by setting

    .. code::

        PAPERLESS_FILENAME_FORMAT=../../my/custom/location/{title}

    However, keep in mind that inside docker, if files get stored outside of the
    predefined volumes, they will be lost after a restart of paperless.


Storage paths
#############

One of the best things in Paperless is that you can not only access the documents via the
web interface, but also via the file system.

When as single storage layout is not sufficient for your use case, storage paths come to
the rescue. Storage paths allow you to configure more precisely where each document is stored
in the file system.

- Each storage path is a `PAPERLESS_FILENAME_FORMAT` and follows the rules described above
- Each document is assigned a storage path using the matching algorithms described above, but
  can be overwritten at any time

For example, you could define the following two storage paths:

1. Normal communications are put into a folder structure sorted by `year/correspondent`
2. Communications with insurance companies are stored in a flat structure with longer file names,
   but containing the full date of the correspondence.

.. code::

    By Year = {created_year}/{correspondent}/{title}
    Insurances = Insurances/{correspondent}/{created_year}-{created_month}-{created_day} {title}


If you then map these storage paths to the documents, you might get the following result.
For simplicity, `By Year` defines the same structure as in the previous example above.

.. code:: text

   2019/                                   # By Year
      My bank/
        Statement January.pdf
        Statement February.pdf

    Insurances/                           # Insurances
      Healthcare 123/
        2022-01-01 Statement January.pdf
        2022-02-02 Letter.pdf
        2022-02-03 Letter.pdf
      Dental 456/
        2021-12-01 New Conditions.pdf


.. hint::

    Defining a storage path is optional. If no storage path is defined for a document, the global
    `PAPERLESS_FILENAME_FORMAT` is applied.

.. caution::

    If you adjust the format of an existing storage path, old documents don't get relocated automatically.
    You need to run the :ref:`document renamer <utilities-renamer>` to adjust their pathes.

.. _advanced-celery-monitoring:

Celery Monitoring
#################

The monitoring tool `Flower <https://flower.readthedocs.io/en/latest/index.html>`_ can be used to view more
detailed information about the health of the celery workers used for asynchronous tasks.  This includes details
on currently running, queued and completed tasks, timing and more.  Flower can also be used with Prometheus, as it
exports metrics.  For details on its capabilities, refer to the Flower documentation.

To configure Flower further, create a `flowerconfig.py` and place it into the `src/paperless` directory.  For
a Docker installation, you can use volumes to accomplish this:

.. code:: yaml

    services:
      # ...
      webserver:
        # ...
        volumes:
          - /path/to/my/flowerconfig.py:/usr/src/paperless/src/paperless/flowerconfig.py:ro

Custom Container Initialization
###############################

The Docker image includes the ability to run custom user scripts during startup.  This could be
utilized for installing additional tools or Python packages, for example.

To utilize this, mount a folder containing your scripts to the custom initialization directory, `/custom-cont-init.d`
and place scripts you wish to run inside.  For security, the folder must be owned by `root` and should have permissions
of `a=rx`.  Additionally, scripts must only be writable by `root`.

Your scripts will be run directly before the webserver completes startup.  Scripts will be run by the `root` user.
If you would like to switch users, the utility `gosu` is available and preferred over `sudo`.

This is an advanced functionality with which you could break functionality or lose data.  If you experience issues,
please disable any custom scripts and try again before reporting an issue.

For example, using Docker Compose:


.. code:: yaml

    services:
      # ...
      webserver:
        # ...
        volumes:
          - /path/to/my/scripts:/custom-cont-init.d:ro


.. _advanced-mysql-caveats:

MySQL Caveats
#############

Case Sensitivity
================

The database interface does not provide a method to configure a MySQL database to
be case sensitive.  This would prevent a user from creating a tag ``Name`` and ``NAME``
as they are considered the same.

Per Django documentation, to enable this requires manual intervention.  To enable
case sensetive tables, you can execute the following command against each table:

``ALTER TABLE <table_name> CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;``

You can also set the default for new tables (this does NOT affect existing tables) with:

``ALTER DATABASE <db_name> CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;``
