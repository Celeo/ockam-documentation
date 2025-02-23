---
description: Connecting a web app and database
---

# Basic Web App

In this example we will explore the Ockam command line interface, [`ockam`](https://docs.ockam.io/#install) and see how we can connect a traditional web app to a PostgreSQL database, with minimal / no code changes. We will create a very basic Python Flask app that simply increments a counter in a PostgreSQL database. Then we will move the connection between the application and database through an Ockam secure channel.

### Background

If you store your data in a relational database, NoSQL, graph database, or something similar, that data is probably private. And you probably don't want to expose it to the Internet. So you can resolve this issue by placing it inside a private subnet. However, now you have to manage network access control lists, security groups, or route tables to allow other machines to open a connection to the database. That is a lot of overhead.

With Ockam, network administrators don't have to update network access control lists, security groups, or route tables. Ockam applies fine grained control to your services via [Attribute-Based Access Control](https://docs.ockam.io/guides/examples/abac). And you can even [integrate with an external identity provider](https://docs.ockam.io/guides/use-cases/use-employee-attributes-from-okta-to-build-trust-with-cryptographically-verifiable-credentials) like [Okta](https://www.okta.com/) to restrict who can access your services.

### Our journey

Before we get started, let's take a look at the steps we'll perform in this example.

<img src="../../.gitbook/assets/file.excalidraw.svg" alt="" class="gitbook-drawing">

1. Use `ockam enroll` to install the Ockam application and create an Ockam project. This is the first prerequisite.
2. Set up the PostgreSQL database. This is the second prerequisite. Then configure an Ockam ["outlet"](https://docs.ockam.io/reference/command/advanced-routing) to the database server. We will learn more about this in the "[connect the database](basic-web-app.md#connect-the-database)" section below.
3. Set up the web app (Python Flask). This is the third prerequisite. Then configure an Ockam ["inlet"](https://docs.ockam.io/reference/command/advanced-routing) from the Python app. We will learn more about this in the "[connect the web app](basic-web-app.md#connect-the-web-app)" section below.

### Prerequisites

In order to follow along, please make sure to install all the prerequisites on the machine where you plan on carrying out the steps below.

1. [Ockam Command](https://www.ockam.io/#install)
   * Run `brew install build-trust/ockam/ockam` to install this via [`brew`](https://brew.sh/). You'll then be able to run the `ockam` CLI app in your terminal.
2. [Python](https://www.python.org/downloads/), and libraries: [Flash](https://github.com/pallets/flask/), [psycopg2](https://github.com/psycopg/psycopg2)
   * Run `brew install python` to install this via [`brew`](https://brew.sh/). You'll then be able to run the `python3` command in your terminal.
   * Instructions on how to get the dependencies (`Flask`, `psycopg2`) are in the [Python Code](https://www.ockam.io/blog/basic-web-app.md#python-code) section below.
3. [Postgresql](https://www.postgresql.org/)
   * Run `brew install postgresql@15` via [`brew`](https://brew.sh/). You'll then be able to run the PostgreSQL database server on your machine on the default port of `5432`. Please make sure to follow `brew`'s instructions and add PostgreSQL to your path.
   * Run `brew services start postgresql@15` to start the PostgreSQL server.
   * Then you can set a new password for the database user `postgres`. Set this password to `password`. The Python Code below uses `postgres:password@localhost` as the connection string for the db driver. These instructions below allow you to do this on Linux and macOS.
     * In a terminal run `sudo -u postgres psql --username postgres --password --dbname template1` to login to the database locally as the `postgres` user.
     * Then type this into REPL: `ALTER USER postgres PASSWORD 'password';`, and finally type `exit`.
     * You can learn more about this [here](https://stackoverflow.com/a/12721095/2085356).

### The Web App - Python Code

The Python Flask web app increments a counter in a PostgreSQL database. The entire app fits in a single file. Create a `main.py` file on your machine and copy and paste the code below into it.

```python
import os
import psycopg2
from flask import Flask

CREATE_TABLE = (
    "CREATE TABLE IF NOT EXISTS events (id SERIAL PRIMARY KEY, name TEXT);"
)

INSERT_RETURN_ID = "INSERT INTO events (name) VALUES (%s) RETURNING id;"

app = Flask(__name__)
pg_port = os.environ['APP_PG_PORT'] # 5432 is the default port
url = "postgres://postgres:password@localhost:%s/"%pg_port
connection = psycopg2.connect(url)

@app.route("/")
def hello_world():
    with connection:
        with connection.cursor() as cursor:
            cursor.execute(CREATE_TABLE)
            cursor.execute(INSERT_RETURN_ID, ("",))
            id = cursor.fetchone()[0]
    return "I've been visited {} times".format(id), 201
```

In this script, we use `"postgres://postgres:password@localhost:%s/"%pg_port` to establish a connection to the database.

* `pg_port` gets its value from the environment variable `APP_PG_PORT`.
* We will set the environment variable `APP_PG_PORT` to `5432` before we run the Python script (instructions below).
* So the database connection string simply points to `localhost:5432`.

{% hint style="info" %}
Please make a note of the `pg_port` Python variable and `APP_PG_PORT` environment variable. In production we usually load the port from an environment variable and it is not hardcoded in the source.
{% endhint %}

#### Run the web app <a href="#run-the-web-app" id="run-the-web-app"></a>

Follow the instructions below to run the web app.

1. First, make sure to add the required Python dependencies with:

```bash
# Install flask.
pip3 install flask
# Install psycopg2.
pip3 install psycopg2-binary
```

2. Then start the `Flask` app (`main.py`) with:

```bash
export APP_PG_PORT=5432
flask --app main run
```

3. Finally, in a web browser open this URL: [`http://localhost:5000/`](http://localhost:5000/).

This Flask app will show you how many times you visited it, and store each new visit in the PostgreSQL database 🎉.

### Install Ockam <a href="#install-ockam" id="install-ockam"></a>

Now that we have set up our web app and database let's do this next:

1. Add Ockam to the mix.
2. Update our `APP_PG_PORT` environment variable so that it connects to a new port (not `5432` which is the where the PostgreSQL server runs).

First, let's run `ockam enroll`. Make sure that you've already installed the Ockam CLI as described in the prerequisites section above.

In a terminal window, run this command and follow the prompts to complete the enrollment process (into Ockam Orchestrator).

```bash
ockam enroll
```

This is what the `ockam enroll` command does:

* It checks that everything is installed correctly after successful enrollment with Ockam Orchestrator.
* It creates a Space and Project for you in Ockam Orchestrator and provisions an End-to-End Encrypted Relay in your `default` project at `/project/default`.

### Connect the database <a href="#connect-the-database" id="connect-the-database"></a>

Next, let's set up a `tcp-outlet` that allows us to send raw TCP traffic to the PostgreSQL server on port `5432`. Then create a relay in our default Orchestrator project. To do this, run these commands in your terminal.

```bash
export PG_PORT=5432
ockam tcp-outlet create --to $PG_PORT
ockam relay create
```

Notes:

* We use `PG_PORT` environment variable here, and not `APP_PG_PORT` (which is used in our web app). It points to the default PostgreSQL port of `5432`. In the section below we will change `APP_PG_PORT` to a different value.
* We'll create the corresponding `tcp-inlet` in the next section.

{% hint style="info" %}
Relays allow you to establish end-to-end protocols with services that operate in remote private networks. They eliminate the need to expose ports on the remote service (to a hostile network like the Internet).
{% endhint %}

### Connect the web app <a href="#connect-the-web-app" id="connect-the-web-app"></a>

Finally, let's setup a local `tcp-inlet` so we can receive raw TCP traffic on port `5433` before it is forwarded.

```bash
export OCKAM_PORT=5433
ockam tcp-inlet create --from $OCKAM_PORT
```

Notes:

* The new environment variable `$OCKAM_PORT` points to a new port `5433`.
* This is the port that the `tcp-inlet` will listen on. And it is different from the default PostgreSQL port.

{% hint style="info" %}
A TCP inlet is a way to define where a node listens for its connections. And then where it should forward that traffic to. An inlet and outlet work together to form a portal.
{% endhint %}

Next, start your web app again with the commands below.

```bash
export APP_PG_PORT=$OCKAM_PORT
flask --app main run
```

<img src="../../.gitbook/assets/file.excalidraw (1).svg" alt="The web app now has a secure channel connection with the database" class="gitbook-drawing">

Finally, connect to this URL again from your web browser `http://localhost:5000/`.

1. We have changed the `$APP_PG_PORT` to the same value as `$OCKAM_PORT` (`5433`). Our web app (`main.py` script) does not directly connect to the unsecure database server (on port `5432`). It now goes through the secure channel 🔐.
2. The counter will continue to increment just as it did before, with zero code changes to your application. But the web app now communicates with the database through an Ockam secure channel 🎉.

### Multiple machines <a href="#multiple-machines" id="multiple-machines"></a>

You can also extend this example and move the PostgreSQL service into a Docker container or to an entirely different machine. Once the nodes are registered (after `ockam enroll` runs), this demo will continue to work, with no application code changes and no need to expose the PostgreSQL ports directly to the Internet.

Also, you can run the web app and the database on different machines. To do this:

1. Change `localhost` in the `main.py` script to the IP address of the machine that hosts the database.
2. Run `ockam enroll` on both machines (the web app machine and the database server machine).

### Other commands to explore

Now that you've completed this example, here are some commands for you to try and see what they do. You can always look up the details on what they do in the [<mark style="color:blue;">manual</mark>](https://command.ockam.io/manual/). As you try each of these, please keep an eye out for things you may have created in this exercise.

* Try `ockam node list`. Do you see the nodes that you created in this exercise?
* Try `ockam node --help`. These are shorter examples for you to get familiar with commands.
* Try `ockam identity list`. Do you see the identities you created in this exercise?
