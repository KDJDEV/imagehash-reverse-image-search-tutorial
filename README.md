# Creating a reverse image search with Python imagehash

Image hashing is essentially a sort of ultra-high compression for images, where images are transformed into hashes (or digests), typically of a fixed length of around 64 bits, where similar images should yield similar hashes. There are many practical applications of image hashing, but one of the most obvious is for *reverse image searches*. 

In this tutorial, you'll be guided through the whole process of creating a reverse image search using the Python *[imagehash](https://github.com/JohannesBuchner/imagehash) library by Johannes Buchner* for the hashes themselves.
> As proof that you can develop a working product by following this tutorial, here is a fully functional reverse image search I created using the same technology described in this article, and which spans all of the images living on English Wikipedia articles:
> https://www.reversewikipedia.com/

Specifically, by the end of this tutorial, you'll have an interface that allows you to add images to a database, and query that database to find near matches. Enough introduction, **let's begin**!

## Table of Contents

* [Prerequisites](#prerequisites)
* [Step 1 - Installing pg-spgist_hamming](#step-1)
* [Step 2 - Populating the Database](#step-2)
* [Step 3 - Querying the database](#step-3)
* [Conclusion](#conclusion)
  

<div id="prerequisites">

## Prerequisites
- [Debian](https://www.debian.org/) or [Ubuntu](https://ubuntu.com/) installed on the machine you wish to host the reverse image search database on. You can use any operating system you wish, as long as it is supported by Postgres, but I'll be using Debian 11, so I recommend you use one of these two OSs so that we have the same package manager (apt).<br/><br/>If your personal computer does not happen to be running Debian or Ubuntu, then **I would recommend you create a virtual machine**. There are tons of providers whom you can create VMs with, but I can personally recommend *Linode, Vultr, AWS, and Google Cloud Platform*, all of which have free trials at the time of writing. (not sponsored)<br/><br/>To follow this tutorial you'll likely want to have at least *10 GB of storage* and *1 GB of RAM*. You could probably squeeze by with less, but I would recommend this minimum.

- [PostgreSQL](https://www.postgresql.org/) version >9.6 installed as well. If you do have the apt package manager, you can easily install PostgreSQL with this command:
```bash 
apt install postgresql
```
> If you are not logged in as the "root" user, you will need to prepend almost all commands in this tutorial with "sudo". Otherwise you will get a permission error.

Note that you need to have direct access to this database in order to install the necessary extension later in this tutorial. If you are using a fully managed database, this tutorial will likely not work for you.

You can use the default PostgreSQL configuration and "postgres" database for this tutorial, but if you want to learn about setting up users and databases, see [this article](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart). Again, you can certainly use another database if you want, but you will most likely have to reimplement the [hamming distance bk-tree used in this tutorial](https://github.com/fake-name/pg-spgist_hamming/) as an extension for your own database.

We will install other things later in this tutorial, but these are the most basic requirements.

</div>

<div id="step-1">

## Step 1 - Installing pg-spgist_hamming
The first thing we need to do is install the [pg-spgist_hamming](https://github.com/fake-name/pg-spgist_hamming/) extension for Postgres. This extension provides us with a [bk-tree](https://en.wikipedia.org/wiki/BK-tree) index that will immensely speed-up the image search. This isn't an absolute requirement for any reverse image search, and there are many possible ways you could achieve a similar speed-up, but it is used here as it works well with the imagehash library.

> For my database, which contains about 30 million image hashes, a search takes on average around 6 seconds without the index, but only around 100 milliseconds with.

First, make sure git is installed:
```bash
apt install git
```

Then, clone in the pg-spgist_hamming repository:
```bash
git clone https://github.com/fake-name/pg-spgist_hamming/
```
```bash
**Output**
Cloning into 'pg-spgist_hamming'...
remote: Enumerating objects: 429, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 429 (delta 0), reused 3 (delta 0), pack-reused 426
Receiving objects: 100% (429/429), 227.48 KiB | 1.31 MiB/s, done.
Resolving deltas: 100% (209/209), done.
```

Now we can compile pg-spgist_hamming and install it so that it can be used by our PostgreSQL database.
First, install the build-essential package, which gives us the necessary compiling tools to build pg-spgist_hamming, and postgresql-server-dev-XX which provides more files necessary for compilation. ***It is very important to replace XX in postgresql-server-dev-XX with the version number of your PostgreSQL installation.***
> You can check which version of PostgreSQL you have installed by connecting to PostgreSQL
> ```bash
> sudo -u postgres psql
> ```
> and then running 
> ```sql
> SELECT VERSION();
> ```
> ```sql
> **Output**
>
>PostgreSQL 13.9 (Debian 13.9-0+deb11u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
> (1 row)
> ```
> Once you have noted the version number, you can quit back to your normal terminal with ```q```, and then ```quit```.

<div id="build-tools">For instance, since I'm using PostgreSQL v13.9, *XX* is replaced with *13*, and I install the postgresql-server-dev-13 package.

```bash
apt install build-essential
apt install postgresql-server-dev-13
```
</div>

Now we can build the extension using GNU make.
```bash
cd pg-spgist_hamming/bktree
make
make install
echo $?
```
```bash
**Output**
gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla 
-Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wformat-security 
-fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation 
-Wno-stringop-truncation -g -g -O2 -fstack-protector-strong -Wformat -Werror=format-security 
-fno-omit-frame-pointer -fPIC -I. -I./ -I/usr/include/postgresql/13/server 
-I/usr/include/postgresql/internal  -Wdate-time -D_FORTIFY_SOURCE=2 
-D_GNU_SOURCE -I/usr/include/libxml2   -c -o bktree.o bktree.c
...
0
```
After running ```echo $?```, you should get a status code of 0 if these commands succeeded. The pg-spgist_hamming extension should now be installed.

Finally, before moving on to the next step, install *Python*. Then install both the *imagehash* and *psycopg2*(which allows us to connect to Postgres through Python) libraries through *pip*.
```bash
apt install python3 python3-pip
pip install imagehash psycopg2
```
These will be necessary in the next step.
</div>
<div id="step-2">

## Step 2 - Populating the Database
At this point in the tutorial, you have a choice. 

1. **If you are looking to create a reverse image search over a specific set of images relevant to your own project**, then I'll give you some basic code for adding hashing to the database, and you'll have to implement this code in a way that is specific to your application. If this is what you want, **read the [Custom Database](#custom-database) section**, but skip the Reverse Wikipedia Database section.
2. **If you just want to play around with and query a reverse image search**, then you can use the precompiled database I created for my [Reverse Wikipedia](https://www.reversewikipedia.com) project. This database spans all of the images existing on Wikipedia articles from 2022, and will allow you to test out all of the functionality presented in this tutorial; you just won't have to populate your own database. If this is what you want, **read the [Reverse Wikipedia Database](#reverse-wikipedia-database) section**, but skip the Custom Database section.

### Custom Database
To populate your database, what you will want to do is loop through all of the images that you have, convert each one to a hash(which is where imagehash comes in), and then append that hash into your database, along with any other data you want to be attached to that image.

Unfortunately, the looping step is going to be dependent on the technicalities of your project. This step might manifest itself as a web crawler, querying of a database or API, or something else depending on what you are doing. To keep things simple here, I'll just pretend we are looping over image files within a directory using a plain old Python for loop.

First though, because we are using PostgreSQL which is a schema-enforced database, we need to define what our table will look like.

#### Some More Database Setup
<div id="connect">First, connect to the PostgreSQL:</div>

```bash
sudo -u postgres psql
```
After running this command, your command prompt should change to ```postgres=#``` to show that you are connected to the "postgres" database. In this tutorial I am leaving the command prompt bit off commands so that they are more easily copied.

First, we'll create a password for the postgres user.
```sql
ALTER USER postgres PASSWORD 'mypassword';
```
<div id="password">Obviously, you'll replace mypassword with a secure password for your database. </div>

>By default, PostgreSQL does not allow connections from external hosts, but it's still a good practice to set a password.

Now you need to create the table that will store your hashes. Here is the basic table we'll use for this tutorial:
| Column | Type |
| -------- | -------- |
| id | integer |
| hash | bigint |
| url | text |

<div id="database-metadata">

This table is created as follows:
```sql
CREATE TABLE hashes (
	id SERIAL PRIMARY KEY,
	hash bigint,
	url text
);
```
```bash
**Output**
CREATE TABLE
```


> This table is another thing that depends on your requirements. If you need to store more data than just the hash and a URL, you can certainly add more columns alongside these ones.
</div>

The *pg-spgist_hamming* extension is installed, but we need to enable it for this database, and for the table we just created.
```sql
CREATE EXTENSION bktree;
CREATE INDEX index ON hashes USING spgist (hash bktree_ops);
```
```bash
**Output**
CREATE EXTENSION
CREATE INDEX
```
Now the database is truly all setup, and we can move on to the Python code.
#### Populating the Database
Again, I'll be assuming that you have a directory of images you are iterating over, but you can modify this code to your needs.

> **You must have already installed the [postgresql-server-dev](#build-tools) package in the previous step for the psycopg2 package to install correctly.**

Now, we'll write the code to populate the database. Here is an example of a script that iterates over all of the images in a directory, and adds them to the database.
```py
import psycopg2
import os
from io import BytesIO
from PIL import Image
import imagehash

def twos_complement(hexstr, bits):
        value = int(hexstr,16) #convert hexadecimal to integer

		#convert from unsigned number to signed number with "bits" bits
        if value & (1 << (bits-1)):
            value -= 1 << bits
        return value

conn = psycopg2.connect(database = "postgres", user = "postgres", password = "mypassword", host = "127.0.0.1")
cursor = conn.cursor()
print("Connection Successful to PostgreSQL")
for entry in os.scandir("./images"):
	with open(entry.path, "rb") as imageBinary:
		img = Image.open(imageBinary)
		imgHash = str(imagehash.dhash(img))
		hashInt = twos_complement(imgHash, 64) #convert from hexadecimal to 64 bit signed integer
		cursor.execute("INSERT INTO hashes(hash, url) VALUES (%s, %s)", (hashInt, entry.name,))
		conn.commit()
		print(f"added image with hash {hashInt} to database")

```
>In this example, *url* is just the filename.

You'll want to replace "mypassword" with [the password you set earlier](#password). In this example I am using the *dhash* function from the imagehash library. Depending on the nature of the images you are working with, it's very likely that you might want to use [a different hashing function](https://github.com/JohannesBuchner/imagehash#references:~:text=written%20in%20Python.-,ImageHash%20supports%3A,-Average%20hashing).

Now is a good time for me to mention some useful Postgres SQL commands you can use to check on your database.

After [connecting to your database](#connect), you can use this command to list the total number of images in the database:
```sql
SELECT count(*) AS exact_count FROM hashes;
```
or this command to get 10 actual rows instead of just a count:
```sql
SELECT * FROM hashes limit 10;
```
You can also leave off the *limit 10* part to get all rows.

Once your database is populated, you can skip to [Step 3](#step-3), which explains how to query your database.

<div id="reverse-wikipedia-database">

### Reverse Wikipedia Database
Importing my Reverse Wikipedia database dump into your database is quite simple. First, download the ```reverseWikipediaDump.sql``` file from my website to your database machine. Don't worry, it's only 1.5 GB.
#### Downloads
```bash
wget https://api.reversewikipedia.com/reverseWikipediaDump.sql
```


Once you've downloaded the dump, load it into your database with this command:
```bash
sudo -u postgres psql postgres < reverseWikipediaDump.sql
```
This command can take a while. (I've found anywhere from 1 minute to around an hour depending on your hardware) Here, ```sudo -u postgres``` runs the command as the postgres user, which is required to access the database, ```psql``` is the CLI tool that allows you to connect to PostgreSQL, and ```postgres < reverseWikipediaDump.sql``` imports reverseWikipediaDump.sql into the "postgres" database.

The database you have imported is very simple in structure.
Each row in the database represents one image. Here is what any one row looks like:
| Column | Type |
| -------- | -------- |
| id | integer |
| hash | bigint |
| url | text |

There is an *id*, which simply gives each row a unique id. There is a *hash*, which is a dhash from the imagehash library encoded as a 64 bit signed integer (a bigint). And there is a *url* which points to the specific Wikipedia page on which the image exists.

You can take a peek at some actual rows with this command, which gets 10 random rows.
```sql
SELECT * FROM hashes ORDER BY random() LIMIT 10;
```
> Don't be worried if this command takes a very long time to complete. It doesn't mean that query times will be slow. It's simply because this specific command is highly unoptimized; it was chosen to be included in this tutorial for its simplicity.

Example result:
| id | hash | url |
| -------- | -------- | -------- |
17354527 | -3689349358451093618 | John_Stokes_(archdeacon_of_York)
26131559 |              8388608 | RMMV_TG_MIL_range_of_trucks
29206937 |  2364618640227041280 | Steg_(Liechtenstein)
16616120 |  2352174474736554650 | Intranodal_palisaded_myofibroblastoma
32710441 |  2640418222820299780 | Western_Desert_campaign
1063903 |   589980399022289036 | 1974_San_Jose_Earthquakes_season
9261783 |  7453280250891000040 | Castle_Vale_Town_F.C.
2528475 |  7588112697248913737 | 2010_in_paleontology
13602297 |  1018730239534117774 | Frazier_Brook
16175955 | -5707918106681791729 | Härryda_Municipality

The *url* is actually just a portion of a full Wikipedia page article. To visit the corresponding article in a web browser, you will need to prepend ```https://en.wikipedia.org/wiki/```. This is so that the database takes up less space.

The database should now be all setup! Now you can skip to [Step 3](#step-3), where you will learn how to query this database.
</div>
</div>

<div id="step-3">

## Step 3 - Querying the database
In this section, you'll learn how you can quickly query the database that you constructed in steps 1 and 2.

Here is the basic Postgres SQL command you can use to query the database.
```sql
SELECT url FROM hashes WHERE hash <@ (hashInt, maximumHammingDistance);
```
Here, the hashInt will be a 64 bit signed integer (the same as the hashes in the database), and maximumHammingDistance is the maximum Hamming distance within which you want to find matches.

The [Hamming distance](https://en.wikipedia.org/wiki/Hamming_distance) is simply the number of positions at which the bits are different. For example, the hamming distance between the numbers *1001* and *1010* is *2*, because the third and fourth bit are both different between the two numbers.

In practice, the number you'll want to replace **maximumHammingDistance** depends on your dataset, and the hash function you are using. **I would recommend starting with a number around 3**, but you'll want to play around with it and see what gets you the best results.

For example, if I wanted to query for images with a hash similar to *2165990786990419763*, I could run something like the following:
```sql
SELECT url FROM hashes WHERE hash <@ (2165990786990419763, 3);
```
The choice of 3 as a maximum hamming distance is of course arbitrary here.

> **You must have already installed the [postgresql-server-dev](#build-tools) package in the previous step for the psycopg2 package to install correctly.**

Here is some code that can query an image from a file. 
```py
import psycopg2
from PIL import Image
import imagehash

maxDifference = 3 #the maximum hamming distance
fileName = "imageToSearch.jpg" #image can be any file type accepted by PIL

conn = psycopg2.connect(database = "postgres", user = "postgres", password = "mypassword", host = "127.0.0.1")
cursor = conn.cursor()
print("Connection Successful to PostgreSQL")

def twos_complement(hexstr, bits):
        value = int(hexstr,16) #convert hexadecimal to integer

		#convert from unsigned number to signed number with "bits" bits
        if value & (1 << (bits-1)):
            value -= 1 << bits
        return value
        
with open(fileName, "rb") as imageBinary:
        img = Image.open(imageBinary)
        imgHash = str(imagehash.dhash(img))
        hashInt = twos_complement(imgHash, 64) #convert from hexadecimal to 64 bit signed integer
        cursor.execute(f"SELECT url FROM hashes WHERE hash <@ ({hashInt}, {maxDifference})")
        hashRows = cursor.fetchall()
        urls = [x[0] for x in hashRows]
        print(urls)
```

> You'll notice that the way we connect to the database and compute the hashes is exactly the same as in the previous section when we populated the database.

This code reads an image from a file and searches for it in the database. It will return the *url* column of any images found, though of course if you decided to add more [metadata alongside your image](#database-metadata) you can return that as well.

And just like that, you have a reverse image search.

If you followed the [Reverse Wikipedia Database](#reverse-wikipedia-database) section of step 2 to test things out with my database, you can replace *"imageToSearch.jpg"* with any image loadable by PIL that is pulled from Wikipedia. For example, try this photo of Mount Everest from the ["Mount Everest" Wikipedia article](https://en.wikipedia.org/wiki/Mount_Everest).

<p align="center">
  <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/e7/Everest_North_Face_toward_Base_Camp_Tibet_Luca_Galuzzi_2006.jpg/408px-Everest_North_Face_toward_Base_Camp_Tibet_Luca_Galuzzi_2006.jpg" alt="GNU" />
</p>

You should get output like this, showing you the pages on which this image exists.
```
['1933_British_Mount_Everest_expedition', '2008_Summer_Olympics_summit_of_Mount_Everest', 'Adrian_Ballinger', 'Cat_in_the_Hat', 'Cat_in_the_Hat_Comes_Back', 'Chomolungma', 'Climate_of_China', 'Everest', 'Geography_of_China', 'Government_of_Tibet_Autonomous_Region', 'Hannah_Shields', ...]
```

> To minimize the size of this database, the results are not the full URLs. If you want to actually visit these pages, prepend each page title with "https://en.wikipedia.org/wiki/".

I'll also note that if you want to be able to search for the SVGs and images with an alpha channel, then you'll need to preprocess images you search for. I explain how to do this in the [Additional resources](#additional-resources) section.

<div id="conclusion">

## Conclusion
In this tutorial you learned how to make a fully functional reverse image search using Python and imagehash. The code presented here is of course rather bare bones, but **I hope it's useful as a basis for whatever project you are working on**. There was a lot I could have put into this tutorial, but I tried to condense it into a form that is actually followable.

Once again, if you would like to see the code from this tutorial in action, take a look at my demo:
https://www.reversewikipedia.com/

<div id="additional-resources">

### Additional resources
Incidentally, here are some additional changes that I needed to get this solution to be viable for my project.

* [Handling SVGs](https://github.com/KDJDEV/probable-octo-chainsaw/blob/main/additionalResources/handlingSVGs.md)
* [Handling images with a transparent alpha channel](https://github.com/KDJDEV/probable-octo-chainsaw/blob/main/additionalResources/imagesWithAlphaTransparency.md)
</div>

### Contributing
If you think you can make this tutorial more informative, please open a pull request!

If you find a problem with this tutorial or have a question, please open a GitHub issue.

</div>