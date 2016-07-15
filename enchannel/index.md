---
layout: page
title: enchannel
permalink: /enchannel/
---

# Build your own REPL with Jupyter via enchannel

This guide is here to help you on your journey to working with nteract's node
packages. The main goal of this guide is to go from scratch to a working interactive
[REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) you can
use at the command line.

Core topics that will be introduced include:

* Jupyter kernels
* Messaging with Jupyter (via ZeroMQ)
* Observables

<!-- TODO: Provide a clear picture of intent and purpose in the intro -->

<!-- TODO: Talk about jupyter console, how it works, and how we can write our own -->

## Requirements

First thing you'll need for this tutorial is the IPython kernel for Python3.

```
python3 -m pip install ipykernel
python3 -m ipykernel install --user
```

You'll also need:

* [Node.js 5+ and npm 3](https://nodejs.org/en/)
* ZeroMQ
* Python 2 (Python 2 is required for builds - you may still run either Python 2 or Python 3)
* IPython kernels installed
* `git`

Each operating system has its own instruction set to get node zmq working. Please read on down to save yourself time.

Note: If you're gripping tightly to an older node version, try out
[nvm](https://github.com/creationix/nvm) or [n](https://github.com/tj/n) for all
your node environment switching needs. As for zmq,
once [zmq-prebuilt](https://github.com/nteract/zmq-prebuilt) is ready for
everyone, you won't have to do these steps.

### OS X

#### Homebrew on OS X

- [`pkg-config`](http://www.freedesktop.org/wiki/Software/pkg-config/): `brew install pkg-config`
- [ZeroMQ](http://zeromq.org/intro:get-the-software): `brew install zeromq`

### Windows

- You'll need a compiler! [Visual Studio 2013 Community Edition](https://www.visualstudio.com/en-us/downloads/download-visual-studio-vs.aspx) is required to build zmq.node.
- Python (tread on your own or install [Anaconda](http://continuum.io/downloads))

After these are installed, you'll likely need to restart your machine (especially after Visual Studio).

### Linux

For Debian/Ubuntu based variants, you'll need `libzmq3-dev` (preferred) or alternatively `libzmq-dev`.   
For RedHat/CentOS/Fedora based variants, you'll need `zeromq` and `zeromq-devel`.

## Initialization

Let's start by creating a new project, backed by `git`:

```
mkdir iconsole
cd iconsole
git init
```

Then we'll make a node package:

```
npm init
```

Feel free to accept the defaults or mix it up. Go ahead and add and commit your new `package.json` to `git`:

```
git add package.json
git commit -m "Add node package"
```

Time to install some packages to kick things off.

```
npm install --save enchannel-zmq-backend@1.0.1
```

If the install failed, try to figure out what went wrong with the zeromq install. Otherwise, join [our slack](http://slack.nteract.in) and we'll see if we can help.

## Spawning a runtime

Before we cover the runtimes (called kernels), let's install our other main dependencies:

```
npm install --save spawnteract@2.0.0 @reactivex/rxjs@5.0.0-beta.2
```

We'll also go ahead and write some code out for us to talk about right after. Create `index.js` with these contents:

```js
const spawnteract = require('spawnteract');
const fs = require('fs');

function cleanup(kernel) {
  kernel.spawn.kill();
  fs.unlink(kernel.connectionFile);
}

spawnteract.launch('python3').then(kernel => {
  console.log(`${kernel.connectionFile}:`);
  console.log(kernel.config);
  cleanup(kernel);
}).catch(err => {
  console.error(err);
});
```

The kernel is the runtime environment that allows a user to have an interactive
session with their language and environment of choice:

```
In [1]: import random

In [2]: random.random()
Out[2]: 0.037744852607581425

In [3]:
```

It functions like a REPL by allowing a user to provide input and get responses back. What's different is that the kernels also provide a rich cross-language protocol
for sending back data as text, HTML, images, etc. while also providing a means to do things like tab completion.

A kernel always operates in a separate process from the application that launches it. Here, `spawnteract` does the launching for us, spawning the process (`kernel.spawn`) as well as creating a connection file, `kernel.connectionFile`, which has `kernel.config` as its contents to allow frontends to connect to it.

Go ahead and run `node index.js` and you should get output similar to:

```
/Users/rgbkrk/Library/Jupyter/runtime/kernel-b2ac934b-2808-49b2-b0c4-44f5b01db219.json:
{ version: 5,
  key: '1b765ffa-f531-4b3e-9ef6-3245784a8262',
  signature_scheme: 'hmac-sha256',
  transport: 'tcp',
  ip: '127.0.0.1',
  hb_port: 9000,
  control_port: 9001,
  shell_port: 9002,
  stdin_port: 9003,
  iopub_port: 9004 }
```

Those are the details for connection, essential for us to ask to execute code, get tab completion hints, query about kernel status, and much more. There are two superheroes that will help us in this quest:

* the [Jupyter messaging spec](http://jupyter-client.readthedocs.org/en/latest/messaging.html)
* [ZeroMQ](http://zguide.zeromq.org/page:all#ZeroMQ-in-a-Hundred-Words)

<!-- Bring in the kernel sketch ups -->

## Talking to the kernel

In order for us to talk to the kernel, we'll need to use a ZeroMQ backend. We're also going to need to identify our session and our messages with unique IDs.

```
npm install --save uuid
```

<!-- TODO: Provide diagram of how Jupyter frontends and backends are connected -->

Near the top of `index.js`, we have a few packages to `require`:

```
const uuid = require('uuid');
const enchannel = require('enchannel-zmq-backend');
```

We're going to make our first request to a kernel, on the `shell` channel, requesting information about the running kernel. We'll use `enchannel-zmq-backend` to take that kernel info and create a nice RxJS subject for us to operate on:

```js
spawnteract.launch('python3').then(kernel => {
  const identity = uuid.v4();
  const session = uuid.v4();

  const shell = enchannel.createShellSubject(identity, kernel.config);
```

After that `shell` is setup, we can start forming a message:

```js
const request = {
  header: {
    username: 'jovyan',
    session,
    msg_type: 'kernel_info_request',
    msg_id: uuid.v4(),
    date: new Date(),
    version: '5.0',
  },
  metadata: {},
  parent_header: {},
  content: {},
};
```

This is probably lower level than you would expect to work with at the get-go and we'll do our best to create helper functions along the way. Part of the adventure here is learning the Jupyter protocol and formats. :smile:

The next item we want to setup is a listener for our message response:

```js
shell.filter(msg => msg.parent_header.msg_id === request.header.msg_id)
     .map(msg => msg.content)
     .first()
     .subscribe(content => {
       console.log(content);
       cleanup(kernel);  // Clean up the spawned process and file like before
       shell.complete(); // Close up shop on the channel
     });
```

<!-- TODO: Outline the format of the messages from this part of the Jupyter spec -->

Once we're ready with a subscription we can send off our message:

```js
shell.next(request);
```

Go ahead and run `index.js` and you'll get the response from the kernel.

```js
{ implementation_version: '4.1.2',
  protocol_version: '5.0',
  help_links:
   [ { url: 'http://docs.python.org/3.5', text: 'Python' },
     { url: 'http://ipython.org/documentation.html',
       text: 'IPython' },
     { url: 'http://docs.scipy.org/doc/numpy/reference/',
       text: 'NumPy' },
     { url: 'http://docs.scipy.org/doc/scipy/reference/',
       text: 'SciPy' },
     { url: 'http://matplotlib.org/contents.html',
       text: 'Matplotlib' },
     { url: 'http://docs.sympy.org/latest/index.html',
       text: 'SymPy' },
     { url: 'http://pandas.pydata.org/pandas-docs/stable/',
       text: 'pandas' } ],
  implementation: 'ipython',
  language_info:
   { version: '3.5.1',
     file_extension: '.py',
     name: 'python',
     nbconvert_exporter: 'python',
     mimetype: 'text/x-python',
     codemirror_mode: { version: 3, name: 'ipython' },
     pygments_lexer: 'ipython3' },
  banner: 'Python 3.5.1 (default, Dec  7 2015, 21:59:08) \nType "copyright", "credits" or "license" for more information.\n\nIPython 4.1.2 -- An enhanced Interactive Python.\n?         -> Introduction and overview of IPython\'s features.\n%quickref -> Quick reference.\nhelp      -> Python\'s own help system.\nobject?   -> Details about \'object\', use \'object??\' for extra details.\n%guiref   -> A brief reference about the graphical user interface.\n' }
```

If you have other kernels, feel free to swap out the `python3` string for your other kernel name. Here's the response from [IJavascript](https://github.com/n-riesco/ijavascript):

```js
{ protocol_version: '5.0',
  implementation: 'ijavascript',
  implementation_version: '5.0.0',
  language_info:
   { name: 'javascript',
     version: '5.5.0',
     mimetype: 'application/javascript',
     file_extension: '.js' },
  banner: 'IJavascript v5.0.0\nhttps://github.com/n-riesco/ijavascript\n',
  help_links:
   [ { text: 'IJavascript Homepage',
       url: 'https://github.com/n-riesco/ijavascript' } ] }
```

### Time to execute code and get results

We're going to make a little toy on top now that sends some pre-fab code over
and prints the full response back.

Instead of hand writing the requests out, let's make a little helper function

We're going to create a little helper function for us to create messages that sets up the boilerplate Jupyter message, accepting the session and the message type.

```js
function createMessage(session, msg_type) {
  const username = process.env.LOGNAME || process.env.USER ||
                   process.env.LNAME || process.env.USERNAME;
  return {
    header: {
      username,
      session,
      msg_type,
      msg_id: uuid.v4(),
      date: new Date(),
      version: '5.0',
    },
    metadata: {},
    parent_header: {},
    content: {},
  };
}
```

To use this, let's switch up the message creation with a few lines:

```js
const create = createMessage.bind(null, session);
const request = create('kernel_info_request');
```

Now you can run it again and see the same output as before. Onward to writing
our first execution request and listening in on it.

We need another channel for us to listen to the response. This time we'll need IOPub. IOPub provides all the messages for side effects from all sending messages to the shell channel. Create it right next to the shell channel:

```js
const shell = enchannel.createShellSubject(identity, kernel.config);
const iopub = enchannel.createIOPubSubject(identity, kernel.config);
```

We now have two channels to cleanup, so let's move more into that `cleanup` function:

```js
function cleanup(kernel, channels) {
  channels.forEach(channel => channel.complete());
  kernel.spawn.kill();
  fs.unlink(kernel.connectionFile);
}
```

This time, we'll cleanup only when the process exits (using a `SIGINT` / `ctrl-c`). We won't be closing the channel during our `kernel_info_request` subscription:

```js
const request = create('kernel_info_request');
shell.filter(msg => msg.parent_header.msg_id === request.header.msg_id)
  .map(msg => msg.content)
  .first()
  .subscribe(content => {
    console.log(content);
  });
shell.next(request);

process.on('SIGINT', () => {
  cleanup(kernel, [shell, iopub]);
});
```

Time to make our execution request. First we'll do the setup of creating the
message and subscribing on our channels.

```js
const executeRequest = create('execute_request');
executeRequest.content = {
  code: 'import random\nrandom.random()',
  silent: false,
  store_history: true,
  user_expressions: {},
  allow_stdin: true,
  stop_on_error: false,
};

iopub.subscribe(msg => console.log('IOPUB', msg));
shell.subscribe(msg => console.log('SHELL', msg));
```

The kernel can take some time to boot before you're able to connect up the IOPub
receiver. Before we do this properly, let's introduce a "sleep" to handle the warmup:

```js
function sleep(ms) {
  return new Promise((resolve) => {
    setTimeout(() => resolve(), ms);
  });
}
sleep(1000).then(() => {
  shell.next(executeRequest);
});
```

As you run `index.js`, watch what you get in the messages. Deep within the messages you should see one similar to this:

```js
{
  header:
   { date: '2016-03-09T23:35:15.868385',
     msg_type: 'execute_result',
     version: '5.0',
     session: 'e62a4f54-d87c-481d-b27e-dc549cbdac1d',
     msg_id: 'b9fa8bab-b8e5-4b5f-b4bf-11cd39500ab4',
     username: 'rgbkrk' },
  parent_header:
   { date: '2016-03-10T05:35:14.854000',
     msg_type: 'execute_request',
     session: '7e5abbd2-6cf5-421d-8c37-c2b515682f91',
     version: '5.0',
     msg_id: '17d7af7a-2dfe-4713-a21d-3a3dac60c2b2',
     username: 'rgbkrk' },
  metadata: {},
  content:
   { execution_count: 1,
     metadata: {},
     data: { 'text/plain': '0.2604591482980253' } } }
```

Navigating into that Object, check out `msg.content.data`. It's an Object with a `mimetype -> data` mapping.

```js
{ 'text/plain': '0.2604591482980253' }
```

That's all we have for now. Please file issues or PRs to expand on this tutorial!
