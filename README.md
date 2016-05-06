About
======
FastCGI support for kaldi. It allows kaldi based speech recognition to be used though apache or nginx (or any other that support FastCGI) HTTP Servers. It also contains simple HTML based client, that allows to test kaldi speech recognition by microphone.

You can test it out here: http://asr-model-demo.api.ai/

Licence
-------
Apache 2.0

Installation guide
==================

Summary
-------

This guide will help you to download and build your own simple ASR 
web-service based on Kaldi ASR code.

Preparing prerequisites
-----------------------

### Creating a working dir

Let's create a directory where all data will be downloaded and built.

	mkdir ~/apiai
	cd ~/apiai

You are free to choose any other name and path you wish to, but will 
have to keep in mind that your name differs from the name given in the 
guide.

Due to server code is based on Kaldi almost all prerequisites matches 
to Kaldi ones. Besides that a FastCGI library is required to communicate 
with HTTP server.

### Getting Kaldi

As a first step you have to clone Kaldi source tree. Due to nnet3 support
is currently under active development and not yet added to the origin 
branch please use our branched off version available at 
<https://github.com/api-ai/kaldi>:

	git clone https://github.com/api-ai/kaldi kaldi-apiai

This command will clone source tree to kaldi-apiai directory. 
To configure and build Kaldi please refer to its official instruction:
<http://kaldi.sourceforge.net/install.html>

### Installing libraries

There are some extra libraries required. You may install them using 
system packet manager.

In openSuSE you may run:

	$ sudo zypper install FastCGI-devel

It you have Debian or Ubuntu:
	
	$ sudo apt-get install libfcgi-dev

Getting the code
--------------

Return to your working directory where you put Kaldi sources

	$ cd ~/apiai

and then clone server source code

	$ git clone https://github.com/api-ai/asr-server asr-server

It is recommended to checkout code to the same directory where 
kaldi-apiai is located to allow `configure` tool to detect Kaldi 
location automatically.

Building the app
--------------

	$ cd asr-server

Before running a make process you have to configure build scripts 
by running a special utility:

	$ ./configure

It will check that all required libraries installed to your system and 
also will look for Kaldi libraries in ../kaldi-apiai folder. If you 
have Kaldi installed somewhere else you may explicitly pass the 
path via --kaldi-root option:

	$ ./configure --kaldi-root=<path_to_kaldi>

If configuration process has finished successfully you may begin 
the building process by running make script:

	$ make

Getting a recognition model
------------------------

When application build complete you need to download language specific 
data. 

Return to your working directory where you put Kaldi sources

	$ cd ~/apiai

Builded ASR application uses a Kaldi nnet3 models, which you can get
by training a neural network with your personal data set or use a 
pretrained network provided by us. Currently it is only English model available
at <https://api.ai/downloads/api.ai-kaldi-asr-model.zip>. 

	$ wget https://api.ai/downloads/api.ai-kaldi-asr-model.zip

Unzip the archive to `asr-server` directory.

	$ unzip api.ai-kaldi-asr-model.zip

Running the app
--------------

Set the model directory as a working dir:

	$ cd api.ai-kaldi-asr-model

There are several ways available to run application. The first one is 
to run it as a standalone app listening on socket defined with 
`--fcgi-socket` option:

	$ ../asr-server/fcgi-nnet3-decoder --fcgi-socket=:8000

This command runs application listening on any IP address and port 8000. 
You are also free to define a path Unix socket, or explicit IP 
address (in a A.B.C.D:PORT form).

As an alternative way you may use special spawn-fcgi utility:

	$ spawn-fcgi -n -p 8000 -- ../asr-server/fcgi-nnet3-decoder

Configuring HTTP service
---------------------

You may use any web-server which have FastCGI support: Apache, Nginx, Lighttpd etc. 

### Installing Apache2

openSuSE:

	$ sudo zypper in apache2
	
Debian and Ubuntu:

	$ sudo apt-get install apache2
	
### Configuring Apache2

Enable FastCGI proxy module with `a2enmod`:
	
	$ sudo a2enmod proxy_fcgi
	
Then you have to add to Apache2 configuration file following line:

	ProxyPass "/asr" "fcgi://localhost:8000/"
	
If your Apache configured to include all .conf files from /etc/apache2/conf.d folder you may 
create separate asr_proxy.conf file with following content:

	ProxyPass "/asr" "fcgi://localhost:8000/"
	Alias /asr-html/ "/home/username/apiai/asr-server/asr-html/"
	<Directory "/home/username/apiai/asr-server/asr-html">
		Options Indexes MultiViews
		AllowOverride None
		Require all granted
	</Directory>
	
Now restart Apache:
	
	$ sudo /etc/init.d/apache2 restart

### Installing Nginx

You can download latest sources from official website <http://nginx.org/> and build Nginx 
with yourself or use your system package manager.

openSuSE:

	$ sudo zypper install nginx

Debian and Ubuntu:

	$ sudo apt-get install nginx

### Configuring Nginx

Open nginx.conf and write down the following code:

	http {
		server {
			location /asr {
				fastcgi_pass 127.0.0.1:8000;
				# Disabling this option invokes immediate sending replies to client
				fastcgi_buffering off;
				# Disabling this option invokes immediate decoding incoming audio data
				fastcgi_request_buffering off;
				include      fastcgi_params;
			}

			location /asr-html {
				root /home/username/apiai/asr-server/;
				index index.html;
			}
		}
	}

This will setup Nginx to pass all requests coming to url /asr directly 
to ASR service listening 8000 port via FastCGI gate. For detailed 
information please please refer to nginx documentation 
(e.g. <https://www.nginx.com/resources/wiki/start/topics/examples/fastcgiexample/>)

Speech Recognition
----------------

Server accepts raw mono 16-bits 16 KHz PCM data. You can convert your audio 
using any popular encoding utilities, for instance, you can use ffmpeg:

	$ ffmpeg -i audio.wav -f s16le -ar 16000 -ac 1 audio.raw

### Recognition using web browser

There is a simple JS implementation that allows you to recognize speech using system mic.
Open in your browser:

	http://localhost/asr-html/

and follow the instructions on the page.

### Recognition from command line using curl

Now, let’s recognize `audio.raw` by calling web-service with `curl` 
utility:

	$ curl -H "Content-Type: application/octet-stream" --data-binary @audio.raw http://localhost/asr

On successfull recognition the command will return something like this:

	{
		"status":"ok",
		"data":[
			{"confidence":0.900359,"text":"HELLO WORLD"}
		]
	}

On error the return value will be like this:

	{"status":"error","data":[{"text":"Failed to decode"}]}

You may specify some decoding parameters in request query string, 
e.g. to change the number of possible recognition results set 
`nbest` parameter:

	$ curl -H "Content-Type: application/octet-stream" --data-binary @audio.raw http://localhost/asr?nbest=2

to get something similar to folowing:

	{
		"status":"ok",
		"data":[
			{"confidence":0.900359,"text":"HELLO WORLD"},
			{"confidence":0.89012,"text":"HELLO WORD"}
		]
	}
