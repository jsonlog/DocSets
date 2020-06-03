
lucasg.github.io blog about rss github
Downloading dash docsets
05 Feb 2017

Dash docsets and the compatible viewers (Dash on OSX, Velocity on Windows and Zeal on Linux/X-Plat) are a godsend for whoever work/develop offline (or on a limited bandwidth). It also has the benefit of being a one-stop shop for documentation (no need to have multiple tabs opened for $ProgrammingLanguage; $BuildPipelineTechno; $VersionControlSoftware; etc.).

Velocity

However I recently wanted to set up Velocity on a truly disconnected computer and unfortunately the application desperately wants to download the official dash docset. I had to find another way to get them.

There is an easy way to download all the user-generated docset (just git clone https://github.com/Kapeli/Dash-User-Contributions.git) in order to manually import them into Velocity, but not for the official Dash docsets.

UPDATE : user contributed docsets have been moved into a CDN, just like the official ones. I’ve updated the bulk downloader to support also UserContributed docsets : dash-doggybag
#!/usr/bin/env python3
import sys
import json
import os
import os.path
import shutil
import logging
import tempfile
import glob
import argparse
import xml.etree.ElementTree as ET
import json
from fnmatch import fnmatch

from tqdm import tqdm # pip install tqdm
import requests # pip install requests



def download_file(url, dest_filepath = None, 
        chunk_size = 32*1024,
        strict_download = False,
        expected_content_type = None
        ):
    """ Download a file a report the progress via the reporthook """

    if not url:
        logging.warning("url not provided : doing nothing")
        return False

    logging.info("Downloading %s in %s" % (url, dest_filepath))
    os.makedirs(os.path.dirname(dest_filepath), exist_ok=True)

    # Streaming, so we can iterate over the response.
    r = requests.get(url, stream=True, allow_redirects = not strict_download)

    # Raise error if the response isn't a 200 OK
    if strict_download and (r.status_code != requests.codes.ok):
        logging.info("Download failed [%d] : %s \n" % (r.status_code, r.headers))
        #r.raise_for_status()
        return False

    content_type = r.headers.get('Content-Type', "")
    if expected_content_type and content_type != expected_content_type:
        logging.info("Wrong expected type : %s != %s \n" % (content_type, expected_content_type))
        #r.raise_for_status()
        return False

    # Total size in bytes.
    total_size = int(r.headers.get('content-length', 0)); 

    with open(dest_filepath, 'wb') as f:
        with tqdm(total=total_size, unit='B', unit_scale=True) as pbar:
            
            for data in r.iter_content(chunk_size):
                read_size = len(data)

                f.write(data)
                pbar.update(read_size)

    logging.info("Download done \n")
    return True

def download_dash_docsets(dest_folder = None, prefered_cdn = "" , docset_pattern = "*"):
    """ 
    Dash docsets are located via dash feeds : https://github.com/Kapeli/feeds
    zip file : https://github.com/Kapeli/feeds/archive/master.zip
    """
    feeds_zip_url = "https://github.com/Kapeli/feeds/archive/master.zip"

    if not dest_folder:
        dest_folder = os.getcwd()

    # Creating destination folder
    dash_docset_dir = dest_folder #os.path.join(dest_folder, "DashDocsets")
    os.makedirs(dash_docset_dir, exist_ok=True)

    with tempfile.TemporaryDirectory() as tmpdirname:
        logging.debug('created temporary directory : %s', tmpdirname)

        feeds_archive = os.path.join(tmpdirname, "feeds.zip")
        feeds_dir = os.path.join(tmpdirname, "feeds-master")

        # Download and unpack feeds
        download_file(feeds_zip_url, feeds_archive)
        shutil.unpack_archive(feeds_archive, os.path.dirname(feeds_archive))

        # parse xml feeds and extract urls
        for feed_filepath in glob.glob("%s/%s.xml" % (feeds_dir, docset_pattern)):

            feed_name, xml_ext  = os.path.splitext(os.path.basename(feed_filepath))
            logging.debug("%s : %s" % (feed_name, feed_filepath))
            
            cdn_url = None
            tree = ET.parse(feed_filepath)
            root = tree.getroot()
            for url in root.findall("url"):
                logging.debug("\turl found : %s" % url.text)

                if "%s.kapeli.com" % prefered_cdn in url.text:
                    logging.debug("\tselected cdn url : %s" % url.text)
                    cdn_url = url.text


            if cdn_url :
                docset_dest_filepath = os.path.join(dash_docset_dir, "%s.tgz" % feed_name)
                download_file(cdn_url, docset_dest_filepath, strict_download = True)
                shutil.move(feed_filepath, os.path.join(dash_docset_dir, os.path.basename(feed_filepath)))

def download_user_contrib_docsets(dest_folder = None, prefered_cdn = "sanfransisco" , docset_pattern = "*"):
    """ 
    Dash docsets are located via dash feeds : https://github.com/Kapeli/feeds
    zip file : https://github.com/Kapeli/feeds/archive/master.zip
    """
    feeds_json_url = "http://%s.kapeli.com/feeds/zzz/user_contributed/build/index.json" % prefered_cdn

    if not dest_folder:
        dest_folder = os.getcwd()

    # Creating destination folder
    user_contrib_docset_dir = os.path.join(dest_folder, "zzz","user_contributed","build")
    os.makedirs(user_contrib_docset_dir, exist_ok=True)
    download_file(feeds_json_url, os.path.join(user_contrib_docset_dir,"index.json"))

    with tempfile.TemporaryDirectory() as tmpdirname:
        logging.debug('created temporary directory : %s', tmpdirname)

        feeds_json = os.path.join(tmpdirname, "feeds.json")

        # Download feed
        download_file(feeds_json_url, feeds_json)
        with open (feeds_json, "r") as js_fd:
            json_feeds = json.load(js_fd)
        docsets = json_feeds['docsets']
        
        # parse xml feeds and extract urls
        for docset in sorted(filter(lambda x: fnmatch(x, docset_pattern), docsets)):
            docset_info = docsets[docset]

            # url format for packages that specify "specific_versions"
            # docset_url = "http://%s.kapeli.com/feeds/zzz/user_contributed/build/%s/versions/%s/%s" % (
            #     prefered_cdn,
            #     docset,
            #     docset_info['version'],
            #     docset_info['archive'],
            # )

            docset_url = "http://%s.kapeli.com/feeds/zzz/user_contributed/build/%s/%s" % (
                prefered_cdn,
                docset,
                docset_info['archive'],
            )

            docset_dest_filepath = os.path.join(user_contrib_docset_dir, docset, docset_info['archive'])
            download_file(docset_url, docset_dest_filepath, strict_download = True, expected_content_type = 'application/x-tar')



if __name__ == '__main__':

    parser = argparse.ArgumentParser(
        description='A downloader for Dash Docsets'
    )

    parser.add_argument("--dash", 
        help="only download dash docsets", 
        action="store_true"
    )

    parser.add_argument("--user-contrib", 
        help="only download user contrib docsets", 
        action="store_true"
    )

    parser.add_argument("-d", "--docset", 
        help="only download a specifics docsets. This option support the glob pattern",
        default="*", 
    )

    parser.add_argument("-v", "--verbose", 
        help="increase output verbosity", 
        action="store_true"
    )

    parser.add_argument("-o", "--output", 
        help="change output directory ", 
        default=os.getcwd()
    )

    parser.add_argument("-c", "--cdn", 
        help="choose cdn (sanfrancisco by default)",
        default = "sanfrancisco", 
        choices=[
            'sanfrancisco',
            'london',
            'newyork',
            'tokyo',
            'frankfurt',
            'sydney',
            'singapore',
        ],
    )

    args = parser.parse_args()

    if args.verbose:
        logging.basicConfig(level=logging.DEBUG)
    else:
        logging.basicConfig(level=logging.INFO)

    os.makedirs(args.output, exist_ok=True)

    with open(os.path.join(args.output, "latencyTest.txt"), 'w') as latency:
        pass
    with open(os.path.join(args.output, "latencyTest_v2.txt"), 'w') as latency:
        pass

    if not args.user_contrib:
        download_dash_docsets(
            dest_folder = args.output,
            prefered_cdn = args.cdn,
            docset_pattern = args.docset
        )

    if not args.dash:
        download_user_contrib_docsets(
            dest_folder = args.output,
            prefered_cdn = args.cdn,
            docset_pattern = args.docset
        )
view raw
dash-doggybag.py hosted with ❤ by GitHub

I had to write a semi-automated wget downloader in Python(3) :

#!/usr/bin/env python3

import sys
import json
import os.path
from urllib.request import urlretrieve

def reporthook(blocknum, blocksize, totalsize):
    readsofar = blocknum * blocksize
    if totalsize > 0:
        percent = readsofar * 1e2 / totalsize
        s = "\r%5.1f%% %*d / %d" % (
            percent, len(str(totalsize)), readsofar, totalsize)
        sys.stderr.write(s)
        if readsofar >= totalsize: # near the end

            sys.stderr.write("\n")
    else: # total size is unknown

        sys.stderr.write("read %d\n" % (readsofar,))

with open("urlist.json", "r") as docset_json :
    docset_list = json.load(docset_json)

    for docset in docset_list:
        docset_name = "%s.tgz" % docset["name"]
        docset_filepath = os.path.join("official_docsets", docset_name)

        sys.stderr.write("Downloading docset %s in %s \n" % (docset_name, docset_filepath))
        urlretrieve(docset["url"], docset_filepath, reporthook)

The reporthook callback function is something I have copied from StackOverflow in order to have a download progress bar (some docsets weight more than 100 MB). In the end the whole official docset list is around 9 GB.

If you’re using Velocity or Dash regularly, I encourage you to buy a license (moreso if you’re using it professionally). Each application is the work of a single developer, and probably their daily breadwinner. I know I intend to buy a license, if only to be able to send features requests.

My only gripes so far with Velocity (I haven’t tried Dash) :

    There is no multi-tab viewer (kinda like HelpViewer for MDSN).
    There is no mass import feature for manually imported docset. You have to import the docsets one by one.
    There is no official MSDN docset. I found a custom built docset but I have no idea if it’s up-to-date and complete.
    There is no official StackOverflow docset. I realize Dash/Velocity is not an offline search engine app, but there is currently no way to easily wade through the offline SO dump.

Here is the urlist.json file regrouping all the docsets’ download urls :

[
{"name" : "NET_Framework", "url" : "http://sanfrancisco.kapeli.com/feeds/NET_Framework.tgz"},
{"name" : "ActionScript", "url" : "http://sanfrancisco.kapeli.com/feeds/ActionScript.tgz"},
{"name" : "Akka", "url" : "http://newyork.kapeli.com/feeds/Akka.tgz"},
{"name" : "Android", "url" : "http://london.kapeli.com/feeds/Android.tgz"},
{"name" : "AngularJS", "url" : "http://sanfrancisco.kapeli.com/feeds/AngularJS.tgz"},
{"name" : "Angular.dart", "url" : "http://sanfrancisco.kapeli.com/feeds/Angular.dart.tgz"},
{"name" : "Ansible", "url" : "http://frankfurt.kapeli.com/feeds/Ansible.tgz"},
{"name" : "Apache_HTTP_Server", "url" : "http://frankfurt.kapeli.com/feeds/Apache_HTTP_Server.tgz"},
{"name" : "Appcelerator_Titanium", "url" : "http://london.kapeli.com/feeds/Appcelerator_Titanium.tgz"},
{"name" : "AppleScript", "url" : "http://london.kapeli.com/feeds/AppleScript.tgz"},
{"name" : "Arduino", "url" : "http://newyork.kapeli.com/feeds/Arduino.tgz"},
{"name" : "AWS_JavaScript", "url" : "http://newyork.kapeli.com/feeds/AWS_JavaScript.tgz"},
{"name" : "BackboneJS", "url" : "http://sanfrancisco.kapeli.com/feeds/BackboneJS.tgz"},
{"name" : "Bash", "url" : "http://frankfurt.kapeli.com/feeds/Bash.tgz"},
{"name" : "Boost", "url" : "http://london.kapeli.com/feeds/Boost.tgz"},
{"name" : "Bootstrap_2", "url" : "http://newyork.kapeli.com/feeds/Bootstrap_2.tgz"},
{"name" : "Bootstrap_3", "url" : "http://sanfrancisco.kapeli.com/feeds/Bootstrap_3.tgz"},
{"name" : "Bootstrap_4", "url" : "http://sanfrancisco.kapeli.com/feeds/Bootstrap_4.tgz"},
{"name" : "Bourbon", "url" : "http://sanfrancisco.kapeli.com/feeds/Bourbon.tgz"},
{"name" : "Neat", "url" : "http://sanfrancisco.kapeli.com/feeds/Neat.tgz"},
{"name" : "C", "url" : "http://sanfrancisco.kapeli.com/feeds/C.tgz"},
{"name" : "C++", "url" :"http://newyork.kapeli.com/feeds/C++.tgz"},
{"name" : "CakePHP", "url" : "http://london.kapeli.com/feeds/CakePHP.tgz"},
{"name" : "Cappuccino", "url" : "http://sanfrancisco.kapeli.com/feeds/Cappuccino.tgz"},
{"name" : "Chai", "url" : "http://sanfrancisco.kapeli.com/feeds/Chai.tgz"},
{"name" : "Chef", "url" : "http://london.kapeli.com/feeds/Chef.tgz"},
{"name" : "Clojure", "url" : "http://newyork.kapeli.com/feeds/Clojure.tgz"},
{"name" : "CMake", "url" : "http://newyork.kapeli.com/feeds/CMake.tgz"},
{"name" : "Cocos2D", "url" : "http://sanfrancisco.kapeli.com/feeds/Cocos2D.tgz"},
{"name" : "Cocos2D-X","url" : "http://frankfurt.kapeli.com/feeds/Cocos2D-X.tgz"},
{"name" : "Cocos3D", "url" : "http://london.kapeli.com/feeds/Cocos3D.tgz"},
{"name" : "CodeIgniter", "url" : "http://newyork.kapeli.com/feeds/CodeIgniter.tgz"},
{"name" : "CoffeeScript", "url" : "http://sanfrancisco.kapeli.com/feeds/CoffeeScript.tgz"},
{"name" : "ColdFusion", "url" : "http://sanfrancisco.kapeli.com/feeds/ColdFusion.tgz"},
{"name" : "Common_Lisp", "url" : "http://newyork.kapeli.com/feeds/Common_Lisp.tgz"},
{"name" : "Compass", "url" : "http://london.kapeli.com/feeds/Compass.tgz"},
{"name" : "Cordova", "url" : "http://sanfrancisco.kapeli.com/feeds/Cordova.tgz"},
{"name" : "Corona", "url" : "http://sanfrancisco.kapeli.com/feeds/Corona.tgz"},
{"name" : "CouchDB", "url" : "http://sanfrancisco.kapeli.com/feeds/CouchDB.tgz"},
{"name" : "Craft", "url" : "http://sanfrancisco.kapeli.com/feeds/Craft.tgz"},
{"name" : "CSS", "url" : "http://london.kapeli.com/feeds/CSS.tgz"},
{"name" : "D3JS", "url" : "http://frankfurt.kapeli.com/feeds/D3JS.tgz"},
{"name" : "Dart", "url" : "http://frankfurt.kapeli.com/feeds/Dart.tgz"},
{"name" : "Django", "url" : "http://sanfrancisco.kapeli.com/feeds/Django.tgz"},
{"name" : "Docker", "url" : "http://frankfurt.kapeli.com/feeds/Docker.tgz"},
{"name" : "Doctrine_ORM", "url" : "http://frankfurt.kapeli.com/feeds/Doctrine_ORM.tgz"},
{"name" : "Dojo", "url" : "http://frankfurt.kapeli.com/feeds/Dojo.tgz"},
{"name" : "Drupal_7", "url" : "http://london.kapeli.com/feeds/Drupal_7.tgz"},
{"name" : "Drupal_8", "url" : "http://frankfurt.kapeli.com/feeds/Drupal_8.tgz"},
{"name" : "ElasticSearch", "url" : "http://sanfrancisco.kapeli.com/feeds/ElasticSearch.tgz"},
{"name" : "Elixir", "url" : "http://sanfrancisco.kapeli.com/feeds/Elixir.tgz"},
{"name" : "Emacs_Lisp", "url" : "http://sanfrancisco.kapeli.com/feeds/Emacs_Lisp.tgz"},
{"name" : "EmberJS", "url" : "http://newyork.kapeli.com/feeds/EmberJS.tgz"},
{"name" : "Emmet", "url" : "http://london.kapeli.com/feeds/Emmet.tgz"},
{"name" : "Erlang", "url" : "http://sanfrancisco.kapeli.com/feeds/Erlang.tgz"},
{"name" : "Express", "url" : "http://sanfrancisco.kapeli.com/feeds/Express.tgz"},
{"name" : "ExpressionEngine", "url" : "http://london.kapeli.com/feeds/ExpressionEngine.tgz"},
{"name" : "ExtJS", "url" : "http://frankfurt.kapeli.com/feeds/ExtJS.tgz"},
{"name" : "Flask", "url" : "http://sanfrancisco.kapeli.com/feeds/Flask.tgz"},
{"name" : "Font_Awesome", "url" : "http://sanfrancisco.kapeli.com/feeds/Font_Awesome.tgz"},
{"name" : "Foundation", "url" : "http://frankfurt.kapeli.com/feeds/Foundation.tgz"},
{"name" : "GLib", "url" : "http://london.kapeli.com/feeds/GLib.tgz"},
{"name" : "Go", "url" : "http://london.kapeli.com/feeds/Go.tgz"},
{"name" : "Grails", "url" : "http://sanfrancisco.kapeli.com/feeds/Grails.tgz"},
{"name" : "Groovy", "url" : "http://sanfrancisco.kapeli.com/feeds/Groovy.tgz"},
{"name" : "Groovy_JDK", "url" : "http://sanfrancisco.kapeli.com/feeds/Groovy_JDK.tgz"},
{"name" : "Grunt", "url" : "http://sanfrancisco.kapeli.com/feeds/Grunt.tgz"},
{"name" : "Gulp", "url" : "http://sanfrancisco.kapeli.com/feeds/Gulp.tgz"},
{"name" : "Haml", "url" : "http://newyork.kapeli.com/feeds/Haml.tgz"},
{"name" : "Handlebars", "url" : "http://london.kapeli.com/feeds/Handlebars.tgz"},
{"name" : "Haskell", "url" : "http://london.kapeli.com/feeds/Haskell.tgz"},
{"name" : "HTML", "url" : "http://sanfrancisco.kapeli.com/feeds/HTML.tgz"},
{"name" : "Ionic", "url" : "http://frankfurt.kapeli.com/feeds/Ionic.tgz"},
{"name" : "Jade", "url" : "http://frankfurt.kapeli.com/feeds/Jade.tgz"},
{"name" : "Jasmine", "url" : "http://frankfurt.kapeli.com/feeds/Jasmine.tgz"},
{"name" : "Java_SE6", "url" : "http://london.kapeli.com/feeds/Java_SE6.tgz"},
{"name" : "Java_SE7", "url" : "http://newyork.kapeli.com/feeds/Java_SE7.tgz"},
{"name" : "Java_SE8", "url" : "http://sanfrancisco.kapeli.com/feeds/Java_SE8.tgz"},
{"name" : "Java_EE6", "url" : "http://sanfrancisco.kapeli.com/feeds/Java_EE6.tgz"},
{"name" : "Java_EE7", "url" : "http://london.kapeli.com/feeds/Java_EE7.tgz"},
{"name" : "JavaFX", "url" : "http://london.kapeli.com/feeds/JavaFX.tgz"},
{"name" : "JavaScript", "url" : "http://sanfrancisco.kapeli.com/feeds/JavaScript.tgz"},
{"name" : "Jinja", "url" : "http://sanfrancisco.kapeli.com/feeds/Jinja.tgz"},
{"name" : "Jekyll", "url" : "http://sanfrancisco.kapeli.com/feeds/Jekyll.tgz"},
{"name" : "Joomla", "url" : "http://sanfrancisco.kapeli.com/feeds/Joomla.tgz"},
{"name" : "jQuery", "url" : "http://frankfurt.kapeli.com/feeds/jQuery.tgz"},
{"name" : "jQuery_Mobile", "url" : "http://london.kapeli.com/feeds/jQuery_Mobile.tgz"},
{"name" : "jQuery_UI", "url" : "http://sanfrancisco.kapeli.com/feeds/jQuery_UI.tgz"},
{"name" : "Julia", "url" : "http://sanfrancisco.kapeli.com/feeds/Julia.tgz"},
{"name" : "KnockoutJS", "url" : "http://frankfurt.kapeli.com/feeds/KnockoutJS.tgz"},
{"name" : "Kobold2D", "url" : "http://london.kapeli.com/feeds/Kobold2D.tgz"},
{"name" : "Laravel", "url" : "http://newyork.kapeli.com/feeds/Laravel.tgz"},
{"name" : "LaTeX", "url" : "http://sanfrancisco.kapeli.com/feeds/LaTeX.tgz"},
{"name" : "Less", "url" : "http://sanfrancisco.kapeli.com/feeds/Less.tgz"},
{"name" : "Lo-Dash","url" :"http://london.kapeli.com/feeds/Lo-Dash.tgz"},
{"name" : "Lua_5.1","url" :"http://london.kapeli.com/feeds/Lua_5.1.tgz"},
{"name" : "Lua_5.2","url" :"http://sanfrancisco.kapeli.com/feeds/Lua_5.2.tgz"},
{"name" : "Lua_5.3","url" :"http://sanfrancisco.kapeli.com/feeds/Lua_5.3.tgz"},
{"name" : "MarionetteJS", "url" : "http://sanfrancisco.kapeli.com/feeds/MarionetteJS.tgz"},
{"name" : "Markdown", "url" : "http://sanfrancisco.kapeli.com/feeds/Markdown.tgz"},
{"name" : "Matplotlib", "url" : "http://sanfrancisco.kapeli.com/feeds/Matplotlib.tgz"},
{"name" : "Meteor", "url" : "http://frankfurt.kapeli.com/feeds/Meteor.tgz"},
{"name" : "Mocha", "url" : "http://frankfurt.kapeli.com/feeds/Mocha.tgz"},
{"name" : "MomentJS", "url" : "http://frankfurt.kapeli.com/feeds/MomentJS.tgz"},
{"name" : "MongoDB", "url" : "http://london.kapeli.com/feeds/MongoDB.tgz"},
{"name" : "Mongoose", "url" : "http://newyork.kapeli.com/feeds/Mongoose.tgz"},
{"name" : "Mono", "url" : "http://sanfrancisco.kapeli.com/feeds/Mono.tgz"},
{"name" : "MooTools", "url" : "http://frankfurt.kapeli.com/feeds/MooTools.tgz"},
{"name" : "MySQL", "url" : "http://london.kapeli.com/feeds/MySQL.tgz"},
{"name" : "Nginx", "url" : "http://london.kapeli.com/feeds/Nginx.tgz"},
{"name" : "NodeJS", "url" : "http://sanfrancisco.kapeli.com/feeds/NodeJS.tgz"},
{"name" : "NumPy", "url" : "http://sanfrancisco.kapeli.com/feeds/NumPy.tgz"},
{"name" : "OCaml", "url" : "http://sanfrancisco.kapeli.com/feeds/OCaml.tgz"},
{"name" : "OpenCV_C", "url" : "http://london.kapeli.com/feeds/OpenCV_C.tgz"},
{"name" : "OpenCV_C++", "url" : "http://london.kapeli.com/feeds/OpenCV_C++.tgz"},
{"name" : "OpenCV_Java", "url" : "http://sanfrancisco.kapeli.com/feeds/OpenCV_Java.tgz"},
{"name" : "OpenCV_Python", "url" : "http://sanfrancisco.kapeli.com/feeds/OpenCV_Python.tgz"},
{"name" : "OpenGL_2", "url" : "http://frankfurt.kapeli.com/feeds/OpenGL_2.tgz"},
{"name" : "OpenGL_3", "url" : "http://newyork.kapeli.com/feeds/OpenGL_3.tgz"},
{"name" : "OpenGL_4", "url" : "http://sanfrancisco.kapeli.com/feeds/OpenGL_4.tgz"},
{"name" : "Pandas", "url" : "http://frankfurt.kapeli.com/feeds/Pandas.tgz"},
{"name" : "Perl", "url" : "http://frankfurt.kapeli.com/feeds/Perl.tgz"},
{"name" : "Phalcon", "url" : "http://london.kapeli.com/feeds/Phalcon.tgz"},
{"name" : "PhoneGap", "url" : "http://london.kapeli.com/feeds/PhoneGap.tgz"},
{"name" : "PHP", "url" : "http://london.kapeli.com/feeds/PHP.tgz"},
{"name" : "PHPUnit", "url" : "http://london.kapeli.com/feeds/PHPUnit.tgz"},
{"name" : "Play_Java", "url" : "http://sanfrancisco.kapeli.com/feeds/Play_Java.tgz"},
{"name" : "Play_Scala", "url" : "http://sanfrancisco.kapeli.com/feeds/Play_Scala.tgz"},
{"name" : "Polymer.dart","url" : "http://sanfrancisco.kapeli.com/feeds/Polymer.dart.tgz"},
{"name" : "PostgreSQL", "url" : "http://london.kapeli.com/feeds/PostgreSQL.tgz"},
{"name" : "Processing", "url" : "http://london.kapeli.com/feeds/Processing.tgz"},
{"name" : "PrototypeJS", "url" : "http://sanfrancisco.kapeli.com/feeds/PrototypeJS.tgz"},
{"name" : "Puppet", "url" : "http://frankfurt.kapeli.com/feeds/Puppet.tgz"},
{"name" : "Python_2", "url" : "http://newyork.kapeli.com/feeds/Python_2.tgz"},
{"name" : "Python_3", "url" : "http://frankfurt.kapeli.com/feeds/Python_3.tgz"},
{"name" : "Qt_4", "url" : "http://sanfrancisco.kapeli.com/feeds/Qt_4.tgz"},
{"name" : "Qt_5", "url" : "http://frankfurt.kapeli.com/feeds/Qt_5.tgz"},
{"name" : "R", "url" : "http://london.kapeli.com/feeds/R.tgz"},
{"name" : "Racket", "url" : "http://london.kapeli.com/feeds/Racket.tgz"},
{"name" : "React", "url" : "http://london.kapeli.com/feeds/React.tgz"},
{"name" : "Redis", "url" : "http://london.kapeli.com/feeds/Redis.tgz"},
{"name" : "RequireJS", "url" : "http://london.kapeli.com/feeds/RequireJS.tgz"},
{"name" : "Ruby", "url" : "http://sanfrancisco.kapeli.com/feeds/Ruby.tgz"},
{"name" : "Ruby_2", "url" : "http://frankfurt.kapeli.com/feeds/Ruby_2.tgz"},
{"name" : "Ruby_on_Rails_3", "url" : "http://london.kapeli.com/feeds/Ruby_on_Rails_3.tgz"},
{"name" : "Ruby_on_Rails_4", "url" : "http://london.kapeli.com/feeds/Ruby_on_Rails_4.tgz"},
{"name" : "Ruby_on_Rails_5", "url" : "http://london.kapeli.com/feeds/Ruby_on_Rails_5.tgz"},
{"name" : "Rust", "url" : "http://newyork.kapeli.com/feeds/Rust.tgz"},
{"name" : "SaltStack", "url" : "http://sanfrancisco.kapeli.com/feeds/SaltStack.tgz"},
{"name" : "Sass", "url" : "http://sanfrancisco.kapeli.com/feeds/Sass.tgz"},
{"name" : "SailsJS", "url" : "http://sanfrancisco.kapeli.com/feeds/SailsJS.tgz"},
{"name" : "Scala", "url" : "http://sanfrancisco.kapeli.com/feeds/Scala.tgz"},
{"name" : "SciPy", "url" : "http://sanfrancisco.kapeli.com/feeds/SciPy.tgz"},
{"name" : "Semantic_UI", "url" : "http://frankfurt.kapeli.com/feeds/Semantic_UI.tgz"},
{"name" : "Sencha_Touch", "url" : "http://frankfurt.kapeli.com/feeds/Sencha_Touch.tgz"},
{"name" : "Sinon", "url" : "http://frankfurt.kapeli.com/feeds/Sinon.tgz"},
{"name" : "Smarty", "url" : "http://london.kapeli.com/feeds/Smarty.tgz"},
{"name" : "Sparrow", "url" : "http://sanfrancisco.kapeli.com/feeds/Sparrow.tgz"},
{"name" : "Spring_Framework", "url" : "http://sanfrancisco.kapeli.com/feeds/Spring_Framework.tgz"},
{"name" : "sproutcore", "url" : "http://docs.sproutcore.com/feeds/sproutcore.tgz"},
{"name" : "SQLAlchemy", "url" : "http://newyork.kapeli.com/feeds/SQLAlchemy.tgz"},
{"name" : "SQLite", "url" : "http://newyork.kapeli.com/feeds/SQLite.tgz"},
{"name" : "Statamic", "url" : "http://london.kapeli.com/feeds/Statamic.tgz"},
{"name" : "Stylus", "url" : "http://london.kapeli.com/feeds/Stylus.tgz"},
{"name" : "Susy", "url" : "http://london.kapeli.com/feeds/Susy.tgz"},
{"name" : "SVG", "url" : "http://sanfrancisco.kapeli.com/feeds/SVG.tgz"},
{"name" : "Swift", "url" : "http://sanfrancisco.kapeli.com/feeds/Swift.tgz"},
{"name" : "Symfony", "url" : "http://frankfurt.kapeli.com/feeds/Symfony.tgz"},
{"name" : "Tcl", "url" : "http://frankfurt.kapeli.com/feeds/Tcl.tgz"},
{"name" : "Tornado", "url" : "http://frankfurt.kapeli.com/feeds/Tornado.tgz"},
{"name" : "Twig", "url" : "http://newyork.kapeli.com/feeds/Twig.tgz"},
{"name" : "Twisted", "url" : "http://sanfrancisco.kapeli.com/feeds/Twisted.tgz"},
{"name" : "TypeScript", "url" : "http://frankfurt.kapeli.com/feeds/TypeScript.tgz"},
{"name" : "TYPO3", "url" : "http://frankfurt.kapeli.com/feeds/TYPO3.tgz"},
{"name" : "UnderscoreJS", "url" : "http://london.kapeli.com/feeds/UnderscoreJS.tgz"},
{"name" : "Unity_3D", "url" : "http://frankfurt.kapeli.com/feeds/Unity_3D.tgz"},
{"name" : "Vagrant", "url" : "http://sanfrancisco.kapeli.com/feeds/Vagrant.tgz"},
{"name" : "Vim", "url" : "http://sanfrancisco.kapeli.com/feeds/Vim.tgz"},
{"name" : "VMware_vSphere", "url" : "http://newyork.kapeli.com/feeds/VMware_vSphere.tgz"},
{"name" : "VueJS", "url" : "http://newyork.kapeli.com/feeds/VueJS.tgz"},
{"name" : "WordPress", "url" : "http://london.kapeli.com/feeds/WordPress.tgz"},
{"name" : "Xamarin", "url" : "http://sanfrancisco.kapeli.com/feeds/Xamarin.tgz"},
{"name" : "Xojo", "url" : "http://frankfurt.kapeli.com/feeds/Xojo.tgz"},
{"name" : "XSLT", "url" : "http://london.kapeli.com/feeds/XSLT.tgz"},
{"name" : "XUL", "url" : "http://london.kapeli.com/feeds/XUL.tgz"},
{"name" : "Yii", "url" : "http://sanfrancisco.kapeli.com/feeds/Yii.tgz"},
{"name" : "YUI", "url" : "http://frankfurt.kapeli.com/feeds/YUI.tgz"},
{"name" : "Zend_Framework_1", "url" : "http://london.kapeli.com/feeds/Zend_Framework_1.tgz"},
{"name" : "Zend_Framework_2", "url" : "http://newyork.kapeli.com/feeds/Zend_Framework_2.tgz"},
{"name" : "ZeptoJS", "url" : "http://newyork.kapeli.com/feeds/ZeptoJS.tgz"}
]

Related Posts

    Calcule le code d'un autoradio Renault 03 Aug 2019
    Discovering Nvidia NvBackend endpoint 01 Feb 2019
    AppV ISV virtual filesystem 22 Aug 2018

© 2019. All rights reserved.
