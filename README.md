# NAME

Linode::API - A client for the Linode API, generated from Linode's own OpenAPI specification

# VERSION

version 0.001

# SYNOPSIS

```perl
use Linode::API();

my $linode = Linode::API->new( token => $personal_access_token );

my $tx = $linode->get_linode_instances( { page_size => 500 } );
die $tx->error->{message} if $tx->error;

say $_->{label} for @{ $tx->res->json->{data} };

# Every operation also has a _p form that returns a Mojo::Promise.
$linode->get_linode_instance_p( { linodeId => $id } )->then(
    sub {
        my ($tx) = @_;
        say $tx->res->json->{status};
    }
)->wait;
```

# DESCRIPTION

This is [OpenAPI::Client](https://metacpan.org/pod/OpenAPI%3A%3AClient) pointed at the specification Linode publishes for its
API, with the two things every caller would otherwise repeat done once: the
bearer token goes on every request, and the API version is fixed when the
client is built rather than passed to every call.

There is one method per operation in the specification, named by its
`operationId` with the hyphens made underscores -- `get_linode_instances`,
`post_linode_instance`, `delete_domain_record` and so on.  Linode's API
reference lists the operationIds, and so does the specification in this
distribution's share directory.  The hyphenated names work too, through
`call`:

```perl
my $tx = $linode->call( 'get-linode-instances' => { page_size => 500 } );
```

Each takes a hashref
of the operation's path, query and header parameters, and a request body as
`json => {...}`:

```perl
my $tx = $linode->post_linode_instance(
    {},
    json => {
        region => 'us-east',
        type   => 'g6-nanode-1',
        image  => 'linode/debian12',
        label  => $label,
    },
);
```

What comes back is a [Mojo::Transaction::HTTP](https://metacpan.org/pod/Mojo%3A%3ATransaction%3A%3AHTTP).  Its `res` is the response,
and `$tx->res->json` is the decoded body.

## What is changed in the specification

The specification is changed in memory as it is loaded, where following it to
the letter would send the wrong thing or refuse the right one:

- The API version is written into each path, as ["new"](#new) describes.
- A header parameter is always a string.  Linode describes `X-Filter` as
the object its JSON encodes, and that object would be sent as `HASH(0x...)`.
- A member of an `allOf` never forbids additional properties.  Linode
closes one member of several, which forbids everything the other members
declare: `post_linode_instance` would refuse `region` and `type`, which it
also requires.  Linode still checks the body when it gets it.

## Telling the failures apart

`$tx->error` is set for three different things, and it is worth knowing
which you have before acting on it:

- The request did not match the specification.

    Nothing was sent.  The response is a `400` that this module made up, and its
    body lists what was wrong under `errors`.  `$tx->req->url` still shows
    where it would have gone.

- The request never got an answer.

    DNS, a refused connection, TLS, or a timeout.  `$tx->error->{code}` is
    undefined, and `$tx->error->{message}` says which.

- Linode said no.

    `$tx->error->{code}` is the status, and the body is Linode's own
    `{ errors => [ { reason => ..., field => ... } ] }`.

## Paging

A list operation returns one page, 100 items long unless you ask for up to 500
with `page_size`.  The body says `page` and `pages`; ask for the next page
until they are equal.  The operations that take an `X-Filter` header can narrow
the list on Linode's side instead.  Its value is JSON, which you encode
yourself; a hashref is refused rather than sent:

```perl
$linode->get_linode_instances( { 'X-Filter' => encode_json( { label => $label } ) } );
```

## Timeouts

The requests go through a [Mojo::UserAgent](https://metacpan.org/pod/Mojo%3A%3AUserAgent), whose defaults are a 10 second
connect timeout and a 40 second inactivity timeout, with no limit on the request
as a whole.  To change them, pass a `ua` of your own.

# METHODS

## new

```perl
my $linode = Linode::API->new(%options);
```

Loads the specification and returns a client for it.  Takes a list of
key-value pairs:

- token

    A Linode personal access token, sent as a bearer token on every request.
    Without one, only the handful of operations that need no authentication will
    work.

- api\_version

    `v4`, which is the default, or `v4beta` for operations Linode has not
    released yet.  It is a preference rather than a rule: an operation that exists
    in only one version is sent to that version whichever you ask for.

- specification

    The path to an OpenAPI specification to use instead of the one shipped with
    this distribution.  Linode publishes theirs from
    [https://github.com/linode/linode-api-docs](https://github.com/linode/linode-api-docs), and a newer copy of it is the
    reason to pass this.

Anything else is passed to ["new" in OpenAPI::Client](https://metacpan.org/pod/OpenAPI%3A%3AClient#new) unchanged: `base_url`,
`ua`, `coerce`, and `app` for testing against a local [Mojolicious](https://metacpan.org/pod/Mojolicious)
application.

The object returned is of a class generated from the specification, which
inherits from this one.  Test it with `isa`, not `ref`.

Reading the specification and building a method for each of its operations
takes a couple of seconds.  That is done once for each specification file and
API version in a process, and every later client built from the same pair
reuses it.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/Troglodyne-Internet-Widgets/Linode-API/issues](https://github.com/Troglodyne-Internet-Widgets/Linode-API/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
