# Robblog -- A robot blogging tool

Robblog is a tool which converts entries from the [ros_datacentre](https://github.com/strands-project/ros_datacentre) into blog posts for [Jekyll](http://jekyllrb.com). It runs as a ros node and provides its own web server, binding to a host and port of your choice.

## Installation

Robblog currently uses [Jekyll](http://jekyllrb.com) to generate pages. To install the dependencies under Ubuntu do:

```bash
sudo apt-get install ruby1.9.1-dev 
sudo gem install jekyll
sudo apt-get install nodejs
```
## Usage

Robblog has two parts: clients which add `robblog/RobblogEntry` entries to the datacentre, and the blog engine which creates blog posts from these entries. 

### Running the Robblog engine

The configuration information that you need to run Robblog is as follows: the datacentre collection to monitor for new entries; the path on the system where Jekyll will create your blog; and the hostname and port to server the webblog at. The code snippet below shows how to create and configure your blog. In this example the datacentre collection is provided as an argument `EntryConverter` (`'robblog'`), the system path is the `blog_path` variable which is defined relative to the `y1_interfaces` package (`roslib.packages.get_pkg_dir('y1_interfaces') + '/content'`), and the host details are obtained from local parameters if provided. 

```python
#!/usr/bin/env python

import rospy
import roslib
import robblog
import robblog.utils

from datetime import *

if __name__ == '__main__':
    rospy.init_node("robblog")

    server_host = rospy.get_param("~host_ip", "127.0.0.1")
    server_port = rospy.get_param("~port", "4000")

    # where are the blog files going to be put
    blog_path = roslib.packages.get_pkg_dir('y1_interfaces') + '/content'
    
    # initialise blog
    robblog.utils.init_blog(blog_path)

    # and start serving content at this place
    proc = robblog.utils.serve(blog_path, server_host, server_port)

    try: 
        converter = robblog.utils.EntryConverter(blog_path=blog_path, collection='robblog')
        
        while not rospy.is_shutdown():
            converter.convert()
            rospy.sleep(1)

    except Exception, e:
                rospy.logfatal(e)
    finally:
        proc.terminate()

```
### Adding entries

To add entries to robblog, clients should add instances of the `robblog/RobblogEntry` message type to ros_datacentre (usually via the message store proxy). The message type is simply:

```
string title
string body
```

The title is translated into the title of the blog post and the name of the markdown file containing the post in the Jekyll install. The body should be [Markdown](http://daringfireball.net/projects/markdown/). 

# Rules for creating entries

## Title choice

To march Jekyll's structuring rules, each entry is converted into a file which is named using the date (YYYY-MM-DD) plus the title (with spaces converted to hyphens). If you create two entries on the same day with the same title this will only create one blog entry in Jekyll, despite their being two different entries in the datacentre. Until Robblog is updated to fix this, if you do plan to create muliple entries with the same title on the same day, it might be worth adding a counter or some other unique characters to the title.


## Illegal characters

As titles are turned into filenames, you need to avoid illegal characters. Currently no sanitisation is done. As far as seen, characters within the body of the entry are fine.

## Images

Images can be included in entries. To include one, you must add the image to the datacentre, retaining the object ID you receive in response. You can then use standard Markdown image inclusion, but replace the image URL with the object ID wrapped in `ObjectID()`, e.g. 

```markdown
[My helpful screenshot](ObjectID(5367e93d54a6f7f69297335e))
```

This ID is used to automatically create a jpeg image to include in the blog post.


# Example

The following example (also available [in full here](https://github.com/strands-project/strands_ui/blob/hydro-devel/robblog/scripts/robblog_example.py)) adds a few entries to the datacentre then serves them. Once you've run this go to [http://localhost:4040](http://localhost:4040) and you should see the blog entries.

```python
#!/usr/bin/env python

import rospy
import roslib
from ros_datacentre.message_store import MessageStoreProxy
from robblog.msg import RobblogEntry
import robblog.utils
import cv
from sensor_msgs.msg import Image
from cv_bridge import CvBridge, CvBridgeError

from datetime import *

if __name__ == '__main__':
    rospy.init_node("robblog_example")


    # Create some blog entries
    msg_store = MessageStoreProxy(collection='example_blog')

    create_entries = True

    if create_entries:
        e1 = RobblogEntry(title='Test Title 1', body='blah blah')
        e2 = RobblogEntry(title='Test Title 2', body='blah blah')
        e3 = RobblogEntry(title='Test Title 3', body='blah blah')
        msg_store.insert(e1)
        msg_store.insert(e2)
        msg_store.insert(e3)

        # add a complex markdown example
        with open(roslib.packages.get_pkg_dir('robblog') + '/data/example.md' , 'r') as f:
            e4 = RobblogEntry(title='Markdown Example', body=f.read())
            msg_store.insert(e4)
            # print e4

        # add an example with an image
        cv_image = cv.LoadImage(roslib.packages.get_pkg_dir('robblog') + '/data/rur.jpg')
        bridge = CvBridge()
        img_msg = bridge.cv_to_imgmsg(cv_image)
        img_id = msg_store.insert(img_msg)
        e5 = RobblogEntry(title='Image Test', body='This is what a robot looks like.\n\n![My helpful screenshot](ObjectID(%s))' % img_id)
        msg_store.insert(e5)


    serve = True

    if serve:
        # where are the blog files going to be put
        blog_path = roslib.packages.get_pkg_dir('robblog') + '/content'
        
        # initialise blog
        robblog.utils.init_blog(blog_path)
        proc = robblog.utils.serve(blog_path, 'localhost', '4040')

        try: 
            converter = robblog.utils.EntryConverter(blog_path=blog_path, collection='example_blog')
            

            while not rospy.is_shutdown():
                # supply True convert to force all pages to be regenerated
                converter.convert()
                rospy.sleep(1)

        except Exception, e:
                    rospy.logfatal(e)
        finally:
            proc.terminate()

```






