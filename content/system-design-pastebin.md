# Design Pastebin.com (or Bit.ly)

*Note: This document links directly to relevant areas found in the [system design topics](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics) to avoid duplication.  Refer to the linked content for general talking points, tradeoffs, and alternatives.*

**Design Bit.ly** - is a similar question, except pastebin requires storing the paste contents instead of the original unshortened url.

## Step 1: Outline use cases and constraints

> Gather requirements and scope the problem.
> Ask questions to clarify use cases and constraints.
> Discuss assumptions.

Without an interviewer to address clarifying questions, we'll define some use cases and constraints.

### Use cases

#### We'll scope the problem to handle only the following use cases

* **User** enters a block of text and gets a randomly generated link
    * Expiration
        * Default setting does not expire
        * Can optionally set a timed expiration
* **User** enters a paste's url and views the contents
* **User** is anonymous
* **Service** tracks analytics of pages
    * Monthly visit stats
* **Service** deletes expired pastes
* **Service** has high availability

#### Out of scope

* **User** registers for an account
    * **User** verifies email
* **User** logs into a registered account
    * **User** edits the document
* **User** can set visibility
* **User** can set the shortlink

### Constraints and assumptions

#### State assumptions

* Traffic is not evenly distributed
* Following a short link should be fast
* Pastes are text only
* Page view analytics do not need to be realtime
* 10 million users
* 10 million paste writes per month
* 100 million paste reads per month
* 10:1 read to write ratio

#### Calculate usage

**Clarify with your interviewer if you should run back-of-the-envelope usage calculations.**

* Size per paste
    * 1 KB content per paste
    * `shortlink` - 7 bytes
    * `expiration_length_in_minutes` - 4 bytes
    * `created_at` - 5 bytes
    * `paste_path` - 255 bytes
    * total = ~1.27 KB
* 12.7 GB of new paste content per month
    * 1.27 KB per paste * 10 million pastes per month
    * ~450 GB of new paste content in 3 years
    * 360 million shortlinks in 3 years
    * Assume most are new pastes instead of updates to existing ones
* 4 paste writes per second on average
* 40 read requests per second on average

Handy conversion guide:

* 2.5 million seconds per month
* 1 request per second = 2.5 million requests per month
* 40 requests per second = 100 million requests per month
* 400 requests per second = 1 billion requests per month

## Step 2: Create a high level design

> Outline a high level design with all important components.

![Imgur](http://i.imgur.com/BKsBnmG.png)

## Step 3: Design core components

> Dive into details for each core component.

### Use case: User enters a block of text and gets a randomly generated link

We could use a [relational database](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms) as a large hash table, mapping the generated url to a file server and path containing the paste file.

Instead of managing a file server, we could use a managed **Object Store** such as Amazon S3 or a [NoSQL document store](https://github.com/donnemartin/system-design-primer#document-store).

An alternative to a relational database acting as a large hash table, we could use a [NoSQL key-value store](https://github.com/donnemartin/system-design-primer#key-value-store).  We should discuss the [tradeoffs between choosing SQL or NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql).  The following discussion uses the relational database approach.

* The **Client** sends a create paste request to the **Web Server**, running as a [reverse proxy](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
* The **Web Server** forwards the request to the **Write API** server
* The **Write API** server does the following:
    * Generates a unique url
        * Checks if the url is unique by looking at the **SQL Database** for a duplicate
        * If the url is not unique, it generates another url
        * If we supported a custom url, we could use the user-supplied (also check for a duplicate)
    * Saves to the **SQL Database** `pastes` table
    * Saves the paste data to the **Object Store**
    * Returns the url

**Clarify with your interviewer how much code you are expected to write**.

The `pastes` table could have the following structure:

```
shortlink char(7) NOT NULL
expiration_length_in_minutes int NOT NULL
created_at datetime NOT NULL
paste_path varchar(255) NOT NULL
PRIMARY KEY(shortlink)
```

Setting the primary key to be based on the `shortlink` column creates an [index](https://github.com/donnemartin/system-design-primer#use-good-indices) that the database uses to enforce uniqueness. We'll create an additional index on `created_at` to speed up lookups (log-time instead of scanning the entire table) and to keep the data in memory.  Reading 1 MB sequentially from memory takes about 250 microseconds, while reading from SSD takes 4x and from disk takes 80x longer.<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>

To generate the unique url, we could:

* Take the [**MD5**](https://en.wikipedia.org/wiki/MD5) hash of the user's ip_address + timestamp
    * MD5 is a widely used hashing function that produces a 128-bit hash value
    * MD5 is uniformly distributed
    * Alternatively, we could also take the MD5 hash of randomly-generated data
* [**Base 62**](https://www.kerstner.at/2012/07/shortening-strings-using-base-62-encoding/) encode the MD5 hash
    * Base 62 encodes to `[a-zA-Z0-9]` which works well for urls, eliminating the need for escaping special characters
    * There is only one hash result for the original input and Base 62 is deterministic (no randomness involved)
    * Base 64 is another popular encoding but provides issues for urls because of the additional `+` and `/` characters
    * The following [Base 62 pseudocode](http://stackoverflow.com/questions/742013/how-to-code-a-url-shortener) runs in O(k) time where k is the number of digits = 7:

```python
def base_encode(num, base=62):
    digits = []
    while num > 0
      remainder = modulo(num, base)
      digits.push(remainder)
      num = divide(num, base)
    digits = digits.reverse
```

* Take the first 7 characters of the output, which results in 62^7 possible values and should be sufficient to handle our constraint of 360 million shortlinks in 3 years:

```python
url = base_encode(md5(ip_address+timestamp))[:URL_LENGTH]
```

We'll use a public [**REST API**](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest):

```
$ curl -X POST --data '{ "expiration_length_in_minutes": "60", \
    "paste_contents": "Hello World!" }' https://pastebin.com/api/v1/paste
```

Response:

```
{
    "shortlink": "foobar"
}
```

For internal communications, we could use [Remote Procedure Calls](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc).

### Use case: User enters a paste's url and views the contents

* The **Client** sends a get paste request to the **Web Server**
* The **Web Server** forwards the request to the **Read API** server
* The **Read API** server does the following:
    * Checks the **SQL Database** for the generated url
        * If the url is in the **SQL Database**, fetch the paste contents from the **Object Store**
        * Else, return an error message for the user

REST API:

```
$ curl https://pastebin.com/api/v1/paste?shortlink=foobar
```

Response:

```
{
    "paste_contents": "Hello World"
    "created_at": "YYYY-MM-DD HH:MM:SS"
    "expiration_length_in_minutes": "60"
}
```

### Use case: Service tracks analytics of pages

Since realtime analytics are not a requirement, we could simply **MapReduce** the **Web Server** logs to generate hit counts.

**Clarify with your interviewer how much code you are expected to write**.

```python
class HitCounts(MRJob):

    def extract_url(self, line):
        """Extract the generated url from the log line."""
        ...

    def extract_year_month(self, line):
        """Return the year and month portions of the timestamp."""
        ...

    def mapper(self, _, line):
        """Parse each log line, extract and transform relevant lines.

        Emit key value pairs of the form:

        (2016-01, url0), 1
        (2016-01, url0), 1
        (2016-01, url1), 1
        """
        url = self.extract_url(line)
        period = self.extract_year_month(line)
        yield (period, url), 1

    def reducer(self, key, values):
        """Sum values for each key.

        (2016-01, url0), 2
        (2016-01, url1), 1
        """
        yield key, sum(values)
```

### Use case: Service deletes expired pastes

To delete expired pastes, we could just scan the **SQL Database** for all entries whose expiration timestamp are older than the current timestamp.  All expired entries would then be deleted (or  marked as expired) from the table.

## Step 4: Scale the design

> Identify and address bottlenecks, given the constraints.

![Imgur](http://i.imgur.com/4edXG0T.png)