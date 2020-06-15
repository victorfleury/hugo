---
title: "Grab a nice wallpaper every day"
date: 2018-02-25T18:45:26+01:00
tags: ['python']
---

This is a small tool I developed to grab the latest image from the [National Geographic's](https://www.nationalgeographic.com/photography/photo-of-the-day/) website and build a nice photo library of wallpapers.

It is always nice to start the day with a new nice wallpaper.

With a small `cron job`, it is then ran every day automatically.

There is no API is provided, so we must scrap the webpage for the information we want.

But with a simple bit of Python it is quite easy to grab these images.

Let's inspect the page to find the interesting part.
```html
 <div data-pestle-module="infiniteGallery">
    <script type="text/json" data-pestle-options>
        {"id":"OwFPp5uL","firstImage":"https://www.nationalgeographic.com/photography/photo-of-the-day/2018/02/pakistan-coal-miner-portrait","endpoint":"https://www.nationalgeographic.com/photography/photo-of-the-day/_jcr_content/.gallery.2018-02.json",......
```

Here what we are interested in, is the JSON file.
Inspecting it in Firefox gives us a good overview of the content.
We can see that there is one file per month of the year based on the naming convention : `.gallery.<YEAR-MONTH>.json`

The file contains a list of `items`. The useful ones are described as follow : *(The descriptions are only wild guesses.)*

* `index` : a sorting index
* `title` : title of the photo
* `caption` : caption text being displayed on the site
* `credit` : author
* `profileUrl` : profile page of the author
* `altText` : alternative text
* `full-path-url` : URL to the picture
* `url` : root url
* `originalUrl`
* `aspectRatio`
* `sizes` : a dictionary of sizes
* `internal`
* `pageUrl`
* `publishDate` : date of publication

The attribute we are going to look at more closely is the `size` one. It contains a dictionary with key/value as the size/url to the picture.

It is then easy to write a small function that will extract the data from the JSON file and download the picture and save it wherever you want.

Here is the simple code to get today's photo :

```python {linenos:table}
import datetime
import json
import logging
import os
from optparse import OptionParser
import time
import urllib


logging.basicConfig()
LOGGER = logging.getLogger()
LOGGER.setLevel(logging.INFO)
ROOT_URL = 'https://yourshot.nationalgeographic.com/'
JSON_ROOT_URL = 'https://www.nationalgeographic.com/photography/photo-of-the-day/_jcr_content/'
DATA_FORMAT = '.gallery.{YEAR}-{MONTH}.json'
ROOT_OUTPUT = '/home/victor/Images/'


def get_picture(item, size, base=ROOT_OUTPUT, force=False):
    """Get the picture of the day.

    This method will parse a given json file to retrieve a URL from which it
    will download the picture.
    Args:
        item(dict): The dictionary representing the item
        size(str): The size we cant to download.
        base(str): The base of the path where images will be saved.
        force(bool): Force overwrite of the image if it is already on disk.
    Return:
        bool: True if successful otherwise False
    """
    sizes = item.get('sizes', None)
    if sizes:
        url_part = sizes[size]
    else:
        url_part = item.get('url')

    photo_url = '{0}'.format(url_part)

    raw_date = item['publishDate']
    converted = time.strptime(raw_date, '%B %d, %Y')
    output = time.strftime('%Y_%m_%d', converted)
    try:
        if not os.path.exists(base):
            os.makedirs(base)
        # Photo already exists and no need to download it again
        if os.path.exists('{0}{1}.jpg'.format(base, output)) and not force:
            LOGGER.info('  * File already exists on disk.')
            return True
        urllib.urlretrieve(
            photo_url,
            '{0}{1}.jpg'.format(base, output),
        )
        LOGGER.info('File successfully retrieved.')
    except IOError:
        LOGGER.error('Could not save file...')
        return False

    return True
```

From there, we can modify the script to retrieve all the photos from the month
and add a bit of a command line interface with a parser, to look for the size we
want to download :

```python
import datetime
import json
import logging
import os
import urllib
import time
from optparse import OptionParser

logging.basicConfig()
LOGGER = logging.getLogger()
LOGGER.setLevel(logging.INFO)
ROOT_URL = 'https://yourshot.nationalgeographic.com/'
JSON_ROOT_URL = 'https://www.nationalgeographic.com/photography/photo-of-the-day/_jcr_content/'
DATA_FORMAT = '.gallery.{YEAR}-{MONTH}.json'
ROOT_OUTPUT = '/home/victor/Images/'


def get_picture(item, size, base=ROOT_OUTPUT, force=False):
    """Get the picture of the day.

    This method will parse a given json file to retrieve a URL from which it
    will download the picture.
    Args:
        item(dict): The dictionary representing the item
        size(str): The size we can to download.
        base(str): The base of the path where images will be saved.
        force(bool): Force overwrite of the image if it is already on disk.
    Return:
        bool: True if successful otherwise False
    """
    sizes = item.get('sizes', None)
    if sizes:
        url_part = sizes[size]
    else:
        url_part = item.get('url')

    photo_url = '{0}'.format(url_part)

    raw_date = item['publishDate']
    converted = time.strptime(raw_date, '%B %d, %Y')
    output = time.strftime('%Y_%m_%d', converted)
    try:
        if not os.path.exists(base):
            os.makedirs(base)
        # Photo already exists and no need to download it again
        if os.path.exists('{0}{1}.jpg'.format(base, output)) and not force:
            LOGGER.info('File already exists on disk.')
            return True
        urllib.urlretrieve(
            photo_url,
            '{0}{1}.jpg'.format(base, output),
        )
        LOGGER.info('File successfully retrieved.')
    except IOError:
        LOGGER.error('Could not save file...')
        return False

    return True


def get_all_month_photos(size=None, year=None, month=None, force=False):
    """Get all the photo of the month.

    Args:
        size(str): The size of the picture we want to download.
        year(str): The year to look for pictures.
        month(str): The month to look for pictures.
        force(bool): Force the download and save of the picture even if it is
            already present.
    """
    # Checking the size
    if size not in ['1024', '1600', '2048', '640', '320', '240', '800', '500']:
        raise RuntimeError('Size is not valid.')
    LOGGER.info('About to retrieve data...')

    # Getting the date and the month
    if not year or not month:
        today_date = year or datetime.date.today()
        today_iso = month or today_date.isoformat()

        year, month, _ = today_iso.split('-')

    LOGGER.info('Retrieving date : {0}-{1}'.format(year, month))
    # Retrieving the json file
    json_file_name = DATA_FORMAT.format(YEAR=year, MONTH=month)

    json_url = '{0}{1}'.format(JSON_ROOT_URL, json_file_name)

    response = urllib.urlopen(json_url)
    photos_data = json.loads(response.read())

    if photos_data:
        LOGGER.info(
            'Retrieving : {0} photos.'.format(len(photos_data['items']))
        )

    for index, item in enumerate(photos_data['items']):
        LOGGER.info(
            'Retrieving {0}/{1}'.format(index+1, len(photos_data['items']))
        )
        if not get_picture(item=item, size=size, force=force):
            LOGGER.error('Could not retrieve : {0}'.format(item['publishDate']))

if __name__ == '__main__':
    import sys
    parser = OptionParser()
    parser.add_option(
        "-s",
        "--size",
        dest="size",
        default='2048',
        help="Size for the image.",
    )
    parser.add_option(
        "-m",
        "--month",
        dest="month",
        default=None,
        help="The month to get photo from.",
    )
    parser.add_option(
        "-y",
        "--year",
        dest="year",
        default=None,
        help="Year to get photo from.",
    )
    parser.add_option(
        "-f",
        "--force",
        dest="force",
        action="store_true",
        default=False,
        help="Year to get photo from.",
    )

    options, _ = parser.parse_args()

    get_all_month_photos(
        size=options.size,
        year=options.year,
        month=options.month,
        force=options.force
    )

```

Now you just run it like this in a console :
```terminal
$ python daily_photo.py
```

Or with arguments :

```terminal
$ python daily_photo.py --size 2048 --year 2018 --month 08
```
Et voilà !

There you have all the *photos of the day* of the month.

Some improvements though, we could manage errors better in case the JSON file does not provide the correct size information, etc...

That's all folks !