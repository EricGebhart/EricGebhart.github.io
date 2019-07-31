---
layout: post
title: "Converting from wordpress to markdown"
description: "A tool to recover pages and posts from a wordpress SQLdump into markdown"
date: 2019-07-30
category: code
tags: [wordpress, jekyll, markdown, html, awk, sed, shell]
---

I recently abandoned wordpress for github pages and jekyll.
Wordress is such an unweildy beast and can be so much work to maintain. Serving a static website using markdown is so much nicer.

I had one problem. My wordress site no longer existed and all I had was the *sqldump* and the site's *tgz* _(zipped tar)_ file.

I found several articles on how to extract and convert my pages and posts into markdown. But none were all that simple, so I embarqued on my own path to do it myself.

---
## For mavericks. Just tell me how!

If you know what you are doing or you just don't want the details,
Get your *sqldump* loaded up and then go clone my [wp-md repo](http://github.com/ericgebhart/wp-md) 

_cd_ into the project and run *gen-posts* like this.

`gen-posts -u <username> -d <databasename>`

Or like this:

`gen-posts -u <username> -d <databasename> -q select_pages.sql -w wp_pages -m md_pages`

Or if you don't want to query again but want to reconvert:

`gen-posts -n`

For help:

`gen-posts -h`

It'll take just a few seconds depending on how many posts you have.
And it will create 2 directories. __wp_posts__ and __md_posts__.
Check your posts to see how it did. If they are not quite what 
you want, then it's time to read the _Readme_ in the project. 
Or maybe just add some sed commands to `convert.sed` and rerun.

---

## Get MySQL running.

There are many articles on this, so I'm not going to help you here.
I run Arch Linux so I followed the [directions for MariaDB](https://wiki.archlinux.org/index.php/Mariadb) which is the prefered flavor of mysql on Arch.

## Load your SQLDump

`mysql -u <username> -p <database-name> < yoursqldump.sql`

If you don't actually know the database name you can find it
with this.

`head yoursqldump.sql | grep Database`

For me that gave me this.

_Host: localhost       Database: ericgebhart_


## Take a look around if you like

Startup the mysql REPL. `mysql -u <username> -p`
Once inside some commands to try are:

* `show databases;`
* `use <databasename>;`
* `show tables;`
* `describe <tablename>;`
* `describe wp_posts;`

Of course you can do some queries too.  I'll leave that up to you.

## The query to extract the posts.

I searched all over, and there are many different ways to do this.
There are queries which join with other tables with subqueries so you can extract just the posts by a certain author for instance. In my case I'm the only author here so it didn't matter. The query I ended up with is this. There is probably a better one, but this is the first one that worked so that's where I stopped. The `\G` is important, it makes the post pretty, separates the rows and puts the column names in front of each value.  `title:, date: and Content:`

There are two queries here, one for posts which is the default
query and one for pages.

In *select-posts.sql*
```sql
select post_title as title, post_date as date, post_content as Content 
from wp_posts 
where post_type LIKE 'post' and post_status = 'publish' 
group by post_title 
order by post_date DESC\G;
```

Extracting the pages is nearly the same:

This query is in the file *select-pages.sql*
```sql
select post_title as title, post_date as date, post_content as Content 
from wp_posts 
where post_type LIKE 'page' and post_status = 'publish' 
group by post_title 
order by post_date DESC\G;
```

## getting the posts

So that's great, but what you'll get when you run something like 
this;

```shell
mysql -u username -p databasename < select-posts.sql > all_posts.txt`
```
Is a big file _all_posts.txt_ with all the posts in it.
To take care of that we can run an *awk* script to split them all up and put them in a different directory to keep things organized.

This will make sure the directory is there and then put each post
in the __wp_posts__ directory with names _wp-post1_, _wp-post2_, etc.

The awk script creates a new filename `x` each time it encounters
a new row which is `*****************Row #************` it then prints each line it gets to that file. If you are new to awk, it matches on a pattern `/*\*\*\*\*\*\.*/` when that happens it assigns `x` a new file name. Every line is printed by default because the print block doesn't have a pattern to match in front of it. So it prints everything to `x` no matter what. Simple.

```shell
mkdir -p wp_posts
# split all_posts into individual posts and put them in the wp_posts directory.
awk '/\*\*\*\*\*.*/{x="wp_posts/wp-post"++i;}{print > x;}' all_posts.txt
```

## Convert HTML to markdown.

I started out pretty simply with just a few *sed* patterns. A
command like this would replace all the `<H1>` tags with `# `. 

`sed 's/<H1>/# /g'` 

A more interesting pattern is the one to convert urls.

My urls looked something like this.
`<a href="http://some/url/somewhere" target="_blank" title="sometitle">The Link</a>`

What markdown wants is this:

`[The Link](http://some/url/somewhere)`

This is the sed command to do that.

`s:<a href="\([^"]*\)"\([^>].*\)>\(.*\)</a>:\[\3\]\(\1\):g`

Basically, anything that starts with `<a href="` get what follows as
long as there is no `"` until the `"` then get everything that does not
have a `>` until the `>`, then get everything until `</a>`. That gives
us 3 pieces. We put number 3, the link, in '[]' and number 1, the url,
in '()'. We throw number 2 away.

But we need a lot more than that to convert all the html tags to markdown.
A nice feature of *sed* is that
you can just put a list of these commands in a file. For the HTML
I had this was not too complicated. I edit each post anyway, so
If I find something I just add it to the _convert.sed_ file and 
convert the post (for post1) again with

`sed -f convert.sed wp_posts/wp-post1 > Post1.md`

I did that for about five minutes because it's not a complete solution. Yes the post is in markdown, but jekyll and other static site generators expect a header and a reasonable name. _Post1.md_ is not a good name.  We need something like _2018-07-30-Converting-from-wordpress-to-markdown.md_ 

To get a nicely named file with a nice header we can use this.
`gen-posts -f wp_posts/wp_post1` 
But you still want to know how this works, yes?

## Awk and Sed again.

### Get the date, not the time.*

 `awk -F ' ' '/date:/{print $2}' wp-post1`

### Get the title, remove . and , then replace <space> with '-'.

This uses some sneaky field separator action. By using ':' 
The first field becomes the label, and the second becomes the rest of the line, as long as you don't have a colon in your title.
It's a pretty common awk trick.
I could have also used `$0` and used _substr_ to break it the way
I wanted. That would be necessary if titles actually have `:`.
We also get the leading space on the title this way, which turns
into a '-' which is perfect when we make the filename.

Awk gets the title, then sed does the edits.

```shell
`awk -F ':' '/title:/{print $2}' wp_post1 | sed -e 's/[,\.]//g' -e 's/ /-/g'`
```

## Create markdown file with a header and a nice name.

Combining all of this together in a shell script we have this,
which I named `html-md`.


```shell

## shell script to convert a wordpress post into a
## markdown post. The filename is created from the date
## and title of the post.  The basic header is added,
## then the contents are processed by convert.sed to
## to replace most of the html with markdown.
## tweak as needed. Most posts don't need much done to
## them. HTML links are mostly fixed, but they will need
## human editing.

## Takes one argument.  The filename for the wordpress post file
## as created by gen-posts. $1 from here on...

# get the date, not the time.
date=`awk -F ' ' '/date:/{print $2}' $1`

# get the title, remove . and ,. replace <space> with '-'.
# sneaky field separator action. A pretty common awk trick.
# we get the leading space on the title this way, which turns
# into a '-' which is perfect when we make the filename.
title=`awk -F ':' '/title:/{print $2}' $1 | sed -e 's/[,\.]//g' -e 's/ /-/g'`

# make sure we have a place to put stuff.
mkdir -p md_posts

filename=md_posts/$date$title".md"

# so we know what is happening.
echo $filename

# create a new file with a nice jekyll/markdown header.
echo "---
layout: post" > $filename

# get the date and time from the original post file.
# lines 2 and 3, then get rid of the leading spaces.
# could be one sed I think.
sed -n '2,3p' $1 | sed 's/^[ ]*//' >> $filename

# finish up the header.
echo 'description: "Some description here."
category:
tags: []
---' >> $filename

# process the contents and append to the new file.
# get everything from the 4th line on, and send it
# through sed with our set of conversion commands.
# use fmt to break lines into manageble sizes.
sed -n '4,$p' $1 | sed -f convert.sed | fmt >> $filename
```

If you have a single post you would like to convert or 
reconvert after an update to the _convert.sed_ file you
can do that with this command.

`html-md _wp-post-file_ <optional directory>`

or maybe it's easier,

`gen-posts -f wp-post-file -m <optional directory>`

Add _-m directory-name_ if you want to put it somewhere other than *md_posts*. 

It will put the new markdown file in a __md_posts__ directory for you if you don't specify another place.

Alternately if you have your posts in the __wp-posts__ directory you can convert them all with this command.

`find wp_posts -type f -name 'wp-post*' | xargs -n 1 ./html-md`

or this:

`gen-posts -n`

If they are somewhere else.

`gen-posts -n -w somewhere-else`

If you want to put them in another place other than md_posts.

`gen-posts -n -w somewhere-else -m another-place`

## Customizations

If you would like, it's definitely easy enough to create and
use your own *SQL* query. The '-q' option will allow to use it.
If something isn't converted correctly from html to markdown, just
modify or add to *convert.sed*. Those are really the only things
you will most likely need to worry about unless your titles have
`:`s in them.  If you do find things to add, please do a pull request so everyone can enjoy the fruits of your labor.

It might be nice if it would fill in the tags and category as well.  Hmmm. 


## Conclusion

So now, you mostly know how this thing works. The nice part is
that all you really need to do is clone my repo and use **gen-posts** to do what you want. If it works 100% the first time you are
done. Maybe all you need is a little bit editing that isn't too
painful. Or maybe you'll want to update *convert.sed* with some 
tweaks and rerun the conversion with `gen-posts -n`.

Of course just using **gen-posts** as your interface is going to
be easier just because it's consistent. But now you know how it
all works. The *html-md* script is really just a helper function,
**gen-posts** is the interface to everything.
 
All of this code is in [my wp-md github repo](https://github.com/ericgebhart/wp-md).  Just get your mysql server up and running with your database loaded.  Then run *gen-posts*. For my 22 posts this ran in a second or two.

For further information see the README in the repo or ask *gen-posts* for help `gen-posts -h` It has enough options to do whatever you might want to do.


