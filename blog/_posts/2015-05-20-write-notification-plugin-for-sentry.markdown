---
layout: post
comments: true
title: Write Notification Plugin for Sentry
---

## Introduction

This post assumes that you already know what sentry is. If not following is a simply introduction.

Sentry is a error logging and aggregation platform. As it said in their document:

> The Sentry package fundamentally is just a simple server and web UI. It will handle authenticating clients (such as Raven) and all of the logic behind storage and aggregation.
>
> That said, Sentry is not limited to Python. The primary implementation is in Python, but it contains a full API for sending events from any language, in any application.

To use sentry, you can either pay for the [service](http://getsentry.com) managed by sentry team or you can build a sentry server yourself with the open sourced [code](http://github.com/getsentry/sentry).

## Preparation

### Dummy Server

In order to test our notification plugin, first we will write a dummy server to receive the nofications.
With `python-gevent` we can write one very quickly.

```python
import gevent.server


def handle_notify(socket, address):
    """Print the nofication so we can see it."""
    print socket.recv(1024)

def main():
    server = gevent.server.StreamServer(("127.0.0.1", 9999), handle_notify)
    server.serve_forever()


if __name__ == "__main__":
    main()
```

The `gevent.server.StreamServer` will handle in-coming connections and call `handle_notify` for each accepted socket, concurrently. In the `handle_notify` function, we simply print the message sent from plugin to test if it's alright.

### Dummy Client

Besides the dummy server, we also need a dummy client to trigger the error thus send the message to sentry. Sentry support almost any language and there are many client library for us to use.
Here we choose python language and [raven-python](https://github.com/getsentry/raven-python) and it will be very simple:

```python
import raven

client = raven.Client(dsn=sentry_dsn)

try:
	1/0
except Exception:
	client.captureException()
```

The sentry_dsn will be the dsn address of your sentry server. Now when you run script by `python dummy_client.py`, a error message will be sent to sentry to trigger the notification.

## Structure

A sentry plugin needs a specific structure to work. The sentry [document](https://sentry.readthedocs.org/en/latest/developer/plugins/index.html#structure) has a great description so I will not elaborate it here. As an example, the notification plugin in the post has the following structure.

```
setup.py
sentry_notify_dummy/
sentry_notify_dummy/__init__.py
sentry_notify_dummy/plugin.py
```

Inside `sentry_notify_dummy/plugin.py` the plugin class is declared:

```python
import socket

from sentry.plugins.bases.notify import NotificationPlugin


class NotifyDummyPlugin(NotificationPlugin):
	title = 'Dummy Notification'
	slug = 'notify-dummy'
	author = 'Kai Zhang'
	version = '0.1.0'
```

As you can see, sentry already has a base class for notification. It makes writing a notification plugin really easy.
For now, the `NotifyDummyPlugin` only has some meta information like `title`, `author` and so on.

Inside of `setup.py`, the plugin is registered via `entry_points`:

```python
#!/usr/bin/evn python
from setuptools import setup, find_packages

setup(
	name='sentry-notify-dummy',
	version='0.1.0',
	description='A sentry plugin notify the messages to the dummy server.',
	author='Kai Zhang',
	packages=find_packages(),
	install_requires=[
		'sentry',
	],
	entry_points={
		'sentry.apps': [
			'notify_dummy = sentry_notify_dummy',
		],
		'sentry.plugins': [
			'notify_dummy = sentry_notify_dummy.plugin:NotifyDummyPlugin',
		],
	},
)
```

Now we can install the plugin by `pip install -e .` inside the plugin directory.
The `-e` argument of `pip` install a project in editable mode (i.e. setuptools "develop mode")
from a local project path. After this, when we restart the sentry server, it can
automatically find the plugin by the `entry_points`. Now go to the setting page of your
project in sentry, then go to the `Manage Integrations` page, you should see
`notify_dummy` among all the plugins. Enable it and save the changes.


## Notification

Now comes to code that actually notify the messages to our dummy server.
`NotificationPlugin`, the class `NotifyDummyPlugin` is inherited from, has a
method named `notify_users` for sub-classes to override:

```python
	def notify_users(self, group, event, fail_silently=False):
		project_name = group.project.name
		times_seen = group.times_seen
		link = group.get_absolute_url()

		message = (
			u"{project_name} {error} ({times_seen} times seen). {link}").format(
				project_name=project_name,
				error=event.error(),
				times_seen=times_seen,
				link=link)

		connection = socket.create_connection(("localhost", 9999))
		connection.sendall(message.encode("utf-8"))
```

The `notify_users` have two important parameters, `group` and `event`. You can
get the project information, how many times the message has been seen, the link
of the message page in sentry and so on from `group`. And the message from `error()`
of `event`. Then what we need to do is simply sending the message to the dummy
server.

There are more information you can get from `group` and `event`. In fact, you can get
`group` from `event`. You can look at `sentry.models.event.Event` and `sentry.model.group.Group`
for more information.

Now, run the dummy client and you should see the notification message in dummy server.
