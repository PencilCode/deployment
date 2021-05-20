Pencil Code Deployment Scripts
------------------------------

A set of ansible playbooks for deploying the pencilcode service on a GCE cluster.

To prepare a project to deploy a cluster, install ansible locally and follow these steps (tailored for Unix-style systems).

1. Create the Google Cloud project and enable Compute Engine it. (The default project is "pencil-io".)
2. Add service account ```ansible-service```. Download its service key JSON; store it as ```/etc/ansible/keys/service-key.json```.
3. Grant ```ansible-service``` the IAM roles "Compute Admin", "Compute OS Admin Login", and "Service Account User" for the project.
4. Locally, create ```ansible-service``` SSH keys using this command: ```ssh-keygen -C "ansible-service" -f ~/.ssh/ansible-service```
5. Add the public key (```~/.ssh/ansible-service.pub```) to the project under the Compute Engine -> Metadata -> SSH Keys tab.

To deploy the ansible bootstrap VM instance execute this locally:
```
ansible-playbook provision-ansible.yml
```

From that bootstrap instance, the following sets up the whole cluster:

```
sudo -u ansible-service /bin/bash
ansible-playbook provision-nfs.yml
ansible-playbook provision-web.yml
ansible-playbook web.yml
```

Right now these scripts are not particularly parameterized to
allow you deploy your own pencilcode service (e.g., under a
different DNS name).  But these scripts document everything, so
in principle things can be generalized.


Cluster Architecture
--------------------

The current architectural goal of PencilCode is to to scale up to
a few tens of thousands of simultaneous users without code changes,
while retaining flexiblity and simplicity.

We have made three key choices:

 * Distribute traffic to N identical web servers with no user storage.
 * Provision a single NFS server for all persistent user storage.
 * Use domain-based hashing to proxy socket.io services between web servers.

Here is a general picture of what's going on.

```
           (users in the world)
                    |
   Google Compute Engine Load Balancer
                 /  |  \
             web1, web2, (etc)
                 \  |  /
             single nfs server
                   / \
            "/data"   "/backup"
```

The architecture compensates for the fact that the webservers
are running very slow javascript and python code, and it takes
advantage of the fact that NFS servers are very fast.

A benchmark of our node servers suggest that our node.js scripts
top out around 20 requests per second on one CPU (likely short
of our needs).  By improving some code, we can increase that number,
but on our system, we expect the python and javascript application
servers to remain to be the bottleneck.  Therefore, our cluster
is designed to distribute python and javascript load on many CPUs.

In contrast, NFS is fast.  On adequately provisioned hardware,
a single NFS CPU can provide than 100,000 operations per second
(far in excess of our needs).  The main bottleneck for NFS is not
the server software, but the underlying storage: a 100GB SSD disk on
Google Compute Engine provides about 3,000 operations per second
(likely more than adequate for our needs); and that bandwidth
can be increased linearly with size and cost.

There is one type of state on pencilcode that is not on disk:
the connection state of our socket.io servers which provide
realtime connections for student projects. Although every socket.io
server is identical, it is important that all users trying to
connect to the socket.io server for a specific subdomain be
routed to the same server.  To do this, we route socket.io
traffic between web servers, proxying to the web server
specific to the socket.io subdomain.


Web Server Services
-------------------

All the application logic for Pencil Code runs on the web
server instances.  Running all application services other than
storage on identical servers simplifies development and
deployment.

Each web server runs a number of services behind an nginx
server:

```
port 80: nginx
 |
 +- serves static content out of several directories
 |
 +- /load, /save, etc go to the local 'pencils' (node.js) service
 |
 +- /img, /goto, etc go to the local 'uwsgi' (python) service
 |
 +- /socket.io is proxied to a remote 'pencilsock' service.

port 8811: pencilsock
 |
 +- serves socket.io server (node.js)

port 8816: pencils
 |
 +- serves /load, /save, /edit, /run, /code, /home, /print (node.js)

socket /run/uwsgi/app/img: uwsgi
 |
 +- serves uwsgi /img (python)
```

Each service is configured to start on boot and can be bounced with
`sudo service restart [nginx|pencilsock|pencils|uwsgi]`


Static File Configuration
-------------------------

Most of the complexity of pencilcode is in the browser, and our
Javascript, HTML, and CSS static content changes frequently as
the code evolves.  For the most part, our server code is stable.

Although our servers run a single version of our server code,
they serve several different versions of the browser code under
different domains.  For example a "stable" version is served
on "pencilcode.net"; a "staging" version is on "pencil.cc";
an "experimental" version is on "pencil.codes.

In addition, we serve several static content websites on pencilcode,
such as the "gym", "blog", and "ref" subdomains.

Each fork of pencilcode or the static website is pulled from git.
Our webservers have a 'source' user, and they pull
subdirectories of the /home/source directory from several
projects on github.

A current listing of `/home/source`:

```
aimate   # static website
blog     # static website
gym      # static website
ref      # static website

pencilcode   # the main website and the server code
staging      # staging for content
experiment   # experimental fork of content

fish     # a github webhook server for automatic deployment
```
