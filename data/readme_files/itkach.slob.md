* Slob
  Slob (sorted list of blobs) is a read-only, compressed data store
  with dictionary-like interface to look up content by text keys. Keys
  are sorted according to [[http://www.unicode.org/reports/tr10/][Unicode Collation Algorithm]]. This allows to
  perform punctuation, case and diacritics insensitive
  lookups. /slob.py/ is a reference implementation of slob format
  reader and writer in [[http://python.org][Python 3]].

** Installation

   /slob.py/ depends on the following components:

   - [[http://python.org][Python]] >= 3.3
   - [[http://icu-project.org][ICU]] >= 4.8
   - [[https://pypi.python.org/pypi/PyICU][PyICU]] >= 1.5

   In addition, the following components are needed to set up
   slob environment:

   - [[http://git-scm.com/][git]]
   - [[https://virtualenv.pypa.io/][virtualenv]]

   Consult your operating system documentation and these component's
   websites for installation instructions.

   For example, on Ubuntu 14.04, the following command installs
   required packages:

   #+BEGIN_SRC sh
   sudo apt-get install python3 python3-icu python-virtualenv git
   #+END_SRC

   Create new Python virtual environment:

   #+BEGIN_SRC sh
   virtualenv env-slob -p python3 --system-site-packages
   #+END_SRC

   Activate it:

   #+BEGIN_SRC sh
   source env-slob/bin/activate
   #+END_SRC

   Install from source code repository:

   #+BEGIN_SRC sh
   pip install git+https://github.com/itkach/slob.git
   #+END_SRC

   or, download source code manually:

   #+BEGIN_SRC sh
   wget https://github.com/itkach/slob/archive/master.zip
   pip install master.zip
   #+END_SRC

   Run tests:

   #+BEGIN_SRC sh
   python -m unittest slob
   #+END_SRC

** Command line interface

   /slob.py/ provides basic command line interface to inspect
   and modify slob content.

   #+BEGIN_SRC
   usage: slob [-h] {find,get,info,tag} ...

   positional arguments:
     {find,get,info,tag}  sub-command
       find               Find keys
       get                Retrieve blob content
       info               Inspect slob and print basic information about it
       tag                List tags, view or edit tag value
       convert            Create new slob with the same convent but different
                          encoding and compression parameters
                          or split into multiple slobs

   optional arguments:
     -h, --help           show this help message and exit
   #+END_SRC

   To see basic slob info such as text encoding, compression and tags:
   #+BEGIN_SRC
   slob info my.slob
   #+END_SRC

   To see value of a tag, for example /label/:
   #+BEGIN_SRC
   slob tag -n label my.slob
   #+END_SRC

   To set tag value:
   #+BEGIN_SRC
   slob tag -n label -v "A Fine Dictionary" my.slob
   #+END_SRC

   To look up a key, for example /abc/:
   #+BEGIN_SRC
   slob find wordnet-3.0.slob abc
   #+END_SRC

   The output should like something like
   #+BEGIN_SRC
   465 text/html; charset=utf-8 ABC
   466 text/html; charset=utf-8 abcoulomb
   472 text/html; charset=utf-8 ABC's
   468 text/html; charset=utf-8 ABCs
   #+END_SRC

   First column in the output is blob id. It can be used to retrieve
   blob content (content bytes are written to stdout):
   #+BEGIN_SRC
   slob get wordnet-3.0.slob 465
   #+END_SRC

   To re-encode or re-compress slob content with different
   parameters:
   #+BEGIN_SRC
   slob convert -c lzma2 -b 256 simplewiki-20140209.zlib.384k.slob simplewiki-20140209.lzma2.256k.slob
   #+END_SRC

   To split into multiple slobs:

   #+BEGIN_SRC
   slob convert --split 4096 enwiki-20150406.slob enwiki-20150406-vol.slob
   #+END_SRC

   Output name /enwiki-20150406-vol.slob/ is the name of the
   directory where resulting .slob files will be created.

   This is useful for crippled systems that can't use normal
   filesystems and have file size limits, such as SD cards on
   vanilla Android. Note that this command doesn't duplicate any
   content, so clients must search all these slobs when looking for
   shared resources such as stylesheets, fonts, javascript or
   images.


** Examples

*** Basic Usage

    Create a slob:

    #+BEGIN_SRC python
      import slob
      with slob.create('test.slob') as w:
          w.add(b'Hello A', 'a')
          w.add(b'Hello B', 'b')
    #+END_SRC

    Read content:

    #+BEGIN_SRC python
      import slob
      with slob.open('test.slob') as r:
          d = r.as_dict()
          for key in ('a', 'b'):
              result = next(d[key])
              print(result.content)

    #+END_SRC

    will print

    #+BEGIN_SRC
b'Hello A'
b'Hello B'
    #+END_SRC


    Slob we created in this example certainly works, but it is not
    ideal: we neglected to specify content type for the content we
    are adding. Lets consider a slightly more involved example:

    #+BEGIN_SRC python
      import slob
      PLAIN_TEXT = 'text/plain; charset=utf-8'
      with slob.create('test1.slob') as w:
          w.add('Hello, Earth!'.encode('utf-8'),
                'earth', 'terra', content_type=PLAIN_TEXT)
          w.add_alias('земля', 'earth')
          w.add('Hello, Mars!'.encode('utf-8'), 'mars',
                content_type=PLAIN_TEXT)
    #+END_SRC

    Here we specify MIME type of the content we are adding so that
    consumers of this content can display or process it
    properly. Note that the same content may be associated with
    multiple keys, either when it is added or later with /add_alias/.

    This

    #+BEGIN_SRC python
      with slob.open('test1.slob') as r:

          def p(blob):
              print(blob.id, blob.content_type, blob.content)

          for key in ('earth', 'земля', 'terra'):
              blob = next(r.as_dict()[key])
              p(blob)

          p(next(r.as_dict()['mars']))

    #+END_SRC

    will print

    #+BEGIN_SRC
0 text/plain; charset=utf-8 b'Hello, Earth!'
0 text/plain; charset=utf-8 b'Hello, Earth!'
0 text/plain; charset=utf-8 b'Hello, Earth!'
1 text/plain; charset=utf-8 b'Hello, Mars!'
    #+END_SRC

    Note that blob id for the first three keys is the same, they all
    point to the same content item.

    Take a look at tests in /slob.py/ for more examples.


*** Slobber - Minimalistic Web UI (Java)

    See http://github.com/itkach/slobber/


*** Slobby - Minimalistic Web UI (Python)

    See http://github.com/itkach/slobby/

*** Create from MediaWiki sites:

    See https://github.com/itkach/mwscrape2slob

*** Convert XDXF

    See http://github.com/itkach/xdxf2slob/

*** Convert TEI
    See https://github.com/itkach/tei2slob/

*** Convert Aard Dictionary's dictionaries

    See http://github.com/itkach/aar2slob/

*** Convert WordNet database

    See http://github.com/itkach/wordnet2slob/

*** pyglossary - convert various formats

    See https://github.com/ilius/pyglossary

*** Aard 2 - Dictionary for Android

    See http://github.com/itkach/aard2-android/

*** Download Content in Slob Format

    See https://github.com/itkach/slob/wiki/Dictionaries


** Slob File Format

*** Slob

| Element       | Type                                 | Description                                                                                                                                                                        |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| magic         | fixed size sequence of 8 bytes       | Bytes ~21 2d 31 53 4c 4f 42 1f~: string ~!-1SLOB~ followed by ascii unit separator (ascii hex code ~1f~) identifying slob format                                                   |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| uuid          | fixed size sequence of 16 bytes      | Unique slob identifier ([[https://tools.ietf.org/html/rfc4122][RFC 4122]] UUID)                                                                                                                                             |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| encoding      | tiny text (utf8)                     | Name of text encoding used for all other text elements: tag names and values, content types, keys, fragments                                                                       |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| compression   | tiny text                            | Name of compression algorithm used to compress storage bins.                                                                                                                       |
|               |                                      | slob.py understands following names: /bz2/, /zlib/ which correspond to Python module names, and /lzma2/ which refers to raw lzma2 compression with LZMA2 filter (this is default). |
|               |                                      | Empty value means bins are not compressed.                                                                                                                                         |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| tags          | char-sized sequence of tags          | Tags are text key-value pairs that may provide additional information about slob or its data.                                                                                      |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| content types | char-sized sequence of content types | MIME content types. Content items refer to content types by id.                                                                                                                    |
|               |                                      | Content type id is 0-based position of content type in this sequence.                                                                                                              |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| blob count    | int                                  | Number of content items stored in the slob                                                                                                                                         |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| store offset  | long                                 | File position at which store data begins                                                                                                                                           |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| size          | long                                 | Total file byte size (or sum of all files if slob is split into multiple files)                                                                                                    |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| refs          | list of long-positioned refs         | References to content                                                                                                                                                              |
|---------------+--------------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| store         | list of long-positioned store items  | Store item contains number of items stored, content type id for each item and storage bin with each item's content                                                                 |



*** tiny text

    char-sized sequence of encoded text bytes


*** text

    short-sized sequence of encoded text bytes


*** large byte string

    int-sized sequence of bytes

*** /size type/-sized sequence of /items/

     | Element | Type                      |
     |---------+---------------------------|
     | count   | /size type/               |
     | items   | sequence of /count/ items |


*** tag

     | Element | Type                        |
     |---------+-----------------------------|
     | name    | tiny text                   |
     | value   | tiny text padded to maximum |
     |         | length with null bytes      |

     Tag values are tiny text of length 255, starting with encoded
     text bytes followed by null bytes. This allowes modifying tag
     values without having to recompile the whole slob. Null bytes
     must be stripped before decoding value text.

*** content type

    text


*** ref

     | Element    | Type      | Description                                           |
     |------------+-----------+-------------------------------------------------------|
     | key        | text      | Text key associated with content                      |
     | bin index  | int       | Index of compressed bin containing content            |
     | item index | short     | Index of content item inside uncompressed bin         |
     | fragment   | tiny text | Text identifier of a specific location inside content |


*** store item
     | Element          | Type                                                    | Description                                       |
     |------------------+---------------------------------------------------------+---------------------------------------------------|
     | content type ids | int-sized sequence of bytes                             | Each byte is a char representing content type id. |
     | storage bin      | list of int-positioned large byte strings without count | Content                                           |

Storage bin doesn't include leading int that would represent item
count - item count equals the length of content type ids. Items in the
storage bin are large byte strings - actual content bytes.

*** list of /position type/-positioned /items/

     | Element   | Type                                                        | Description                                                                                         |
     |-----------+-------------------------------------------------------------+-----------------------------------------------------------------------------------------------------|
     | positions | int-sized sequence of item offsets of type /position type/. | Item offset specifies position in file where item data starts, relative to the end of position data |
     | items     | sequence of /items/                                         |                                                                                                     |

*** char
    unsigned char (1 byte)

*** short
    big endian unsigned short (2 bytes)

*** int
    big endian unsigned int (4 bytes)

*** long
    big endian unsigned long long (8 bytes)


** Design Considerations

   Slob format design is influenced by [[http://aarddict.org/][Aard Dictionary]]'s aard and [[http://openzim.org/][ZIM]]
   file formats. Similar to Aard Dictionary, it allows to perform
   non-exact lookups based on UCA's notion of collation
   strength. Similar to ZIM, it groups and compresses multiple
   content items to achieve high compression ratio and can combine
   several physical files into one logical container. Both aard and
   ZIM contain vestigial elements of predecessor formats as well
   as elements specific to a particular use case (such as
   implementing offline Wikipedia content access). Slob aims to
   provide a minimal framework to allow building such applications
   while remaining a simple, generic, read-only data store.

*** No Format Version
    Slob header doesn't contain explicit file format version
    number. Any incompatible changes after the format is finalized
    will be introduced in a new file format which will get its own
    identifying magic bytes.

*** No Content Checksum
    Unlike aard and ZIM file formats, slob doesn't contain
    content checksum. File integrity can be easily verified by
    employing standard tools to calculate content hash. Inclusion of
    pre-calculated hash into the file itself prevents using most
    standard tools and puts burden of implementing hash calculation
    on every slob reader implementation.
