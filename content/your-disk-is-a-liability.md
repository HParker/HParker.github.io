# Your web server filesystem is a liability

There seems to be a disconnect about how we should manage our
infrastructure. "Infrastructure as code" and "immutable
infrastructure" are common approaches now. The "pet servers"
anti-pattern seems well explained, but no critique of file system use
on a web server.

The filesystem on your web servers is a liability and you should avoid
touching it at run time if at all possible. There are a number of ways
that it can go wrong. Maybe you deploy to hosts with hard disks or you
are referencing your Kubernetes cluster's persistent volume, either
way persistent storage access should be treated with respect.

## Local static assets

Let's assume that your infrastructure is a typical web stack, `n` web
servers talking to primary db with replica. A common approach I see to
managing assets such as images, css and javascript is to server them
from the web server filesystem. Write some nginx or haproxy config
that tells the web server to serve your static assets from some
directory on the file system; Probably `/var/www/html/<asset type>` or
similar. All set. works great. The downside of this approach is that
you have to build the assets on every web server or rsync the assets
from the deploy server to each of the web servers. Either way, you now
have a lot of duplicate files on each web server, but this alone isn't
a big deal. Now that you have assets on each web server, there are
some weird traps that you can fall in.

### Requests crossing web servers

You are in the middle of a deploy, some percentage of your web servers
are booting the new version of the app as existing web servers finish
fielding their current requests. Given enough traffic, it becomes
likely that you could request an asset from a newer version of the app
that does not exist in the new version of the code. This will cause
the asset to be missing for the client. This will result an error that
is very unlikely to result in a bug report, but will still annoy
users. I have never seen the, "I refreshed the page once and the CSS
was missing, but it was fine the second time" bug report. Regardless,
it will erode customer confidence since the website's behavior will be
inconsistent.

## File uploads

One common way to handle file uploads is to have the user attach the
file in their browser. When the user submits the page the file is sent
up to the server. The file is then received and saved to the web
server's local filesystem.

## Alternative solution using the file system

One technique for avoiding file upload issues is sticky sessions. Make
sure that requests for assets, uploaded files, etc. all go to the same
web server that the request was made from. That way the asset is
always available on the web server that it was uploaded to. The
downside is that this can cause uneven traffic patterns and has some
easy to overlook consequences that I won't go over here. That being
said, it is a real option that would make filesystem use less likely
to cause transient issues.

Another option is use a shared disk via NFS or similar software. this
also works, but means that a potentially critical part of your system
is reliant on that NFS as a single point of failure.

## Alternative approach avoiding the file system

Regardless of your feelings about services, operations that access the
file system should be limited to a set of hosts whose primary job is
not serving web requests. They can server **specific** web requests
for files or other operations that require interacting with the
filesystem, but not something that the web server itself should ever
interact with. Even better if you can pay someone else to manage these
servers for you.