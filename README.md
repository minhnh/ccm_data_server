# `ccm_data_server`

Backend script for handling database requests from [`ccm_components`](https://github.com/DigiKlausur/ccm_components).

- [`ccm_data_server`](#ccmdataserver)
  - [Initialize NPM packages](#initialize-npm-packages)
  - [Initialize/Reset Let's Encrypt Certificates](#initializereset-lets-encrypt-certificates)
  - [To start the backend script](#to-start-the-backend-script)
  - [Start the backend script as a `systemd` service](#start-the-backend-script-as-a-systemd-service)
  - [User access to collection and documents](#user-access-to-collection-and-documents)
  - [Database structure](#database-structure)
    - [`users` collection](#users-collection)
    - [Default collection](#default-collection)
      - [`questions`](#questions)
      - [`answer_<question_id>`](#answerquestionid)
      - [User document](#user-document)

## Initialize NPM packages

Inside the `ccm_data_server` directory, execute:
```
npm install
```

## Initialize/Reset Let's Encrypt Certificates

Note: certificates may be generated somewhere else and copied to the server, as long as the host name
(`digiklausur.ddns.net`) is correct. This may be necessary if the public IP of the server does not have
the HTTP port (80) open.

Largely based on the [tutorial by David Mellul](https://itnext.io/node-express-letsencrypt-generate-a-free-ssl-certificate-and-run-an-https-server-in-5-minutes-a730fbe528ca)
* Run `certbot` in manual mode as `root`
```
# certbot certonly --manual
```
* Follow the instructions to add appropriate domain names, until the window requesting for the specific file
under `<domain name>/.well-known/acme-challenge/<random file name>` with a specific random string as content.
* In another terminal, create `.well-known/acme-challenge/<random file name>` in `ccm_data_server` and add
the random string as its content
* Start [`certbot_setup.js`](./certbot_setup.js) as root:
```
# node certbot_setup.js
```
* Go back to the terminal with `certbot` command and hit `Enter`. If all is well, a message will appear saying
`Congratulations! Your certificate and chain have been saved at` with the certificate locations
* Modify [`config.json`](./config/configs.json) with the appropriate certificate paths, and the backend script is ready to go

## To start the backend script

* Modify [`config.json`](./config/configs.json) with the appropriate settings for the server
* Start [`index.js`](./index.js) as root:
```
# node index.js
```

## Start the backend script as a `systemd` service

* modify [ccm_data_server.service](./ccm_data_server.service) with appropriate system paths
* Create a symbolic link of the file [ccm_data_server.service](./ccm_data_server.service) in system service location
```
# ln -s `pwd`/ccm_data_server.service /lib/systemd/system/
```
* By default non-root users (e.g. `nodejs` like specified in [ccm_data_server.service](./ccm_data_server.service))
  cannot to bind to port 443. A secure alternative is to add a `iptables` rule which redirect TCP requests from port 443
  to a desired port. Sensible values for `<network_interface>` and `<port_num>`: `eth0` and `3000` (like specified in
  [`config.json`](./config/configs.json))
```
# iptables -t nat -A PREROUTING -i <network_interface> -p tcp --dport 443 -j REDIRECT --to-port <port_num>
```
* To make the rule persistent, after running the above command, you can install `iptables-persistent` (Debian & Ubuntu).
  After installation, rules will be automatically loaded from the files `/etc/iptables/rules.v4` for IPv4 and
  `/etc/iptables/rules.v6` for IPv6
* Reload, enable and start the data service
```
# systemctl daemon-reload
# systemctl enable ccm_data_server.service
# systemctl start ccm_data_server.service
```

## User access to collection and documents

Specified in [`user_roles.json`](./config/user_roles.json). Currently 3 roles are supported:
* `admin`
* `grader`
* `student`
TODO(minhnh) need further description of the roles

## Database structure

A sample layout of the database is available in the `resources/question_answers_data.js` on the
[`digiklausur/ccm_components`](https://github.com/DigiKlausur/ccm_components) repository.

### `users` collection

Contain user information e.g. user role. No external access by any user roles is possible for this collection
at the moment. In the future, an admin account may get to modify the role for each user.

### Default collection

Currently designed to store question and answer data for a lecture. Supported documents within this collection are
`questions`, `answers`, and a personal document for each user.

#### `questions`

Contain questions created by `grader` and/or `admin`, which are stored in the `entries` field. Each question in
`entries` should be identified by an unique key (e.g. hash of the question text). Deadlines for answering questions
and ranking answers are stored in `answer_deadline` and `ranking_deadline` fields, respectively.
```json
{
    "_id": "questions",
    "entries": {
        "<hash of question text>": {
            "text": "<question text>",
            "last_modified": "<last user to write to the question>"
        }
    },
    "answer_deadline" : {
        "date" : "YYYY-MM-DD",
        "time" : "hh:mm"
    },
    "ranking_deadline" : {
        "date" : "YYYY-MM-DD",
        "time" : "hh:mm"
    }
}
```

`grader` and `admin` has read/write access to this document, and `student` only has `read` access.

#### `answer_<question_id>`

Contain answers for each question (identified by the `question_id`), the answers' authors as well as the users who
ranked the answers.
```json
{
    "_id" : "answers_<hash of question text>",
    "entries" : {
        "<hash of answer text>" : {
            "text" : "<answer text>",
            "authors" : {
                "<username>" : true
            },
            "ranked_by" : { "<another username>": "<normalized ranking value>" }
        }
    }
}
```

#### User document

This document is named/identified by the user's username and contains answer for each question by the user. Users with
role `student` are allowed write access to this document.

```json
{
    "_id" : "<username>",
    "answers" : {
        "<hash of question text>" : {
            "text" : "<answer text>",
            "hash" : "<hash of answer text>"
    },
    "ranking" : {
        "<hash of question text>": {
            "<hash of answer text>": "<ranking index, e.g. 0-5>"
        }
    },
}
```
