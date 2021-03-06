# NewHackers Hacker News API

A Flask REST API server with a redis backend and Celery background tasks for Hacker News.

## Installation

    $ git clone http://github.com/mapleoin/newhackers
    $ cd newhackers
    $ # install the redis server using your package manager and start it
    $ ./newhackers/celer.py -A tasks worker --loglevel=INFO
    $ ./server

Test a normal request:

    $ curl http://localhost:5000/stories/

Run the unit test suite:

    $ ./setup.py nosetests

Check for other test suites in `tests/functional` and `tests/performance`

## API

All `GET` API functions are cached up to one minute. `POST` requests can not be cached, so be careful about triggering HN's IP block.

All API functions responses have `Content-Type: application/json`.

Errors set the proper HTTP code and return a message stored in the `error` field:

    HTTP/1.1 404 NOT FOUND
    Content-Type: application/json
    Content-Length: 31
    Date: Sun, 28 Oct 2012 13:41:19 GMT

    {
      "error": "404: Not Found"
    }


## Available functions

### Stories

`GET /stories/<page_id>`

#### Arguments

**page_id** - an *optional* string identifier of a page. If blank, then the first page of HN stories will be returned. Otherwise, it tries to return the page identified by `page_id`.

#### Returns

A JSON document with two fields:

A **more** string which contains an identifier to the next page following the one that was returned. This string can be used as the `page_id` argument for `GET /stories/`.

A **stories** list of dictionaries. Each *story* in this dictionary will have the following attributes:

 * **title** - title of a story on the HN page
 * **link** - HTTP link of the story (this can be a path to HN itself in the case of an Ask HN link or a Jobs post)
 * **comments_no** - the number of comments attached to that submission. Can be `null` in the case when the submission is a Jobs post and accepts no comments. If it is `-1` then the remote HN server didn't provide the number of comments.
 * **score** - the number of points that this submission has accumulated
 * **author** - the original submitter of the story to HN; can be `null` if this was a Jobs post.
 * **time** - a float indicating when the item was created (number of seconds from the Epoch)

e.g.

```javascript
{'more': '4AVKeJz9TP',
 'stories': [
      {'title': "I sooo don't like Apple anymore",
       'link': "http://iwoz.woo",
       'comments_no': 1337,
       'score': 42,
       'time': 1350901062.0,
       'author': 'woz'},

      {'title': "Work for my startup for free",
       'link': "item?id=1111",
       'time': 1351333328.0,
       'score': None,
       'author': None,
       'comments_no': None},
       ...]}
```

### Ask HN

`GET /ask/<page_id>`

#### Arguments

**page_id** - an *optional* string identifier of a page. If blank, then the first page of Ask HN stories will be returned. Otherwise, it tries to return the page identified by `page_id`.

#### Returns

A JSON document with two fields:

A **more** string which contains an identifier to the next page following the one that was returned. This string can be used as the `page_id` argument for `GET /stories/`.

A **stories** list of dictionaries. Each *story* in this dictionary will have the following attributes:

 * **title** - title of a story on the HN page
 * **link** - HTTP link of the story this is a path to HN itself in the case of an Ask HN link or a Jobs post
 * **comments_no** - the number of comments attached to that submission. If it is `-1` then the remote HN server didn't provide the number of comments.
 * **score** - the number of points that this submission has accumulated
 * **author** - the original submitter of the story to HN
 

e.g.

```javascript
{"more": "tzTR4eFwuT",
 "stories": [
      {"score": 3,
       "link": "item?id=4704287",
       "time": 1351324947.0,
       "title": "Ask HN: How are you setting up your Java applications?",
       "author": "foo",
       "comments_no": 9},
      {"score": 43,
       "link": "item?id=4704198",
       "time": 1351324947.0,
       "title": "I want to sell my startup to one of company.",
       "author": "matrix",
       "comments_no": 0}
       ...]}
```

### Comments

`GET /comments/<int:item_id>`

#### Arguments

**item_id** - *optional* int identifying the item (story or comment) which the comments are attached to

#### Returns

Information about the item the comments are attached to plus all the comments. E.g.

 * **title** - title of a story on the HN page
 * **link** - HTTP link of the story (this can be a path to HN itself in the case of an Ask HN link or a Jobs post)
 * **comments_no** - the number of comments attached to that submission. Can be `null` in the case when the submission is a Jobs post and accepts no comments. If it is `-1` then the remote HN server didn't provide the number of comments.
 * **score** - the number of points that this submission has accumulated
 * **author** - the original submitter of the story to HN; can be `null` if this was a Jobs post.
 * **time** - a float indicating when the item was created (number of seconds from the Epoch)
 * **comments** - a list of comment dictionaries with the following fields: *author*, *body*, *link*, *time*.

e.g.

```javascript
{'title': "I sooo don't like Apple anymore",
 'link': "http://iwoz.woo",
 'comments_no': 1337,
 'score': 42,
 'time': 1350901062.0,
 'author': 'woz',
 'comments': [{'author': 'foo',
               'body': 'lorem ipsum',
               'link': '123123123',
               'time': 1350901232.0},
              {'author': 'bar',
               'body': 'lorem ipsum',
               'link': '321321321',
               'time': 1350901244.0}
               ...]}
```

### Authentication

`POST /get_token`

#### Arguments

Arguments should be included in the POST request. All arguments are mandatory:

 - **user** - an existing HN username
 - **password** - user's password

#### Returns

**token** - a token string which must be used on other POST requests which require authentication.


### Voting

`POST /vote`

#### Arguments

Arguments should be included in the POST request. All arguments are mandatory:

**token** - an authentication token which will be sent as a Cookie
**item** - string identifying the item
**direction** - either 'up' or 'down'

#### Returns

**vote** - a string which is either *Success* or *Fail*
