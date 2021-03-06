- https://github.com/sibelius/relay-modern-network-deep-dive
- https://github.com/facebook/relay/tree/master/packages/relay-test-utils
- https://twitter.com/sseraphini/status/1078595758801203202

TODO:

- connection with handler? (https://github.com/facebook/relay/issues/2570#issuecomment-438026375)

# Local schema

First define local schema (`schema.local.graphql`):

```graphql
"""
Extend type: http://facebook.github.io/graphql/draft/#sec-Object-Extensions
"""
extend type Article {
  draft: String!
}

"""
Or add new query: https://github.com/facebook/relay/issues/1656#issuecomment-382461183
"""
extend type Query {
  errors: [Error!]
}

type Error {
  id: ID!
  message: String!
}
```

Run Relay Compiler as usual (no special option required):

```
$ relay-compiler --src ./packages --schema ./packages/schema.graphql --verbose
```

File `schema.local.graphql` must be somewhere in `src` folder with the `*.graphql` extension. You should be good to go - just fetch the fields as usual. You have to commit local update to fill these fields and types:

```js
Relay.commitLocalUpdate(environment, store => {
  const articleID = 'f9496862-4fb7-4a09-bc05-a9a3ce2cb7b3'; // ID of the `Article` type you want to update
  store.get(articleID).setValue('My custom draft text', 'draft');

  // or create new types:
  const root = store.getRoot();
  const errRecord = store.create('ID_1', 'Error');
  errRecord.setValue('ID_1', 'id');
  errRecord.setValue('My custom error message', 'message');
  root.setLinkedRecords([errRecord, ...], 'errors');
});
```

More info here: http://facebook.github.io/relay/docs/en/relay-store.html

Protip: create many local GraphQL extensions closely related to one specific part of your application. For example you could create `gdsv.local.graphql` with the following content:

```graphql
extend type PNRInfo {
  successMessage: String
}
```

This way I created `successMessage` client field on `PNRInfo` type and it should be more or less obvious that it's related only to this `GDSV` part. All local schemas are being auto-discovered thanks to `*.graphql` file extension. You can now fetch and render this success message somewhere in GDSV application. Propagation of this message is trivial (you have to fetch the `PNRInfo` ID):

```js
Relay.commitLocalUpdate(environment, store => {
  store
    .get(response.id) // unique opaque ID identifying PNRInfo record
    .setValue(
      'Request has been successfully sent.', // the actual message
      'successMessage' // client field name
    )
  }
);
```

Please note that client schema is still somehow experimental feature and that server may introduce the same field `successMessage` which will conflict with the client one (new kind of BC break). Luckily, Relay will recognize this BC break and it will throw an error:

```
ERROR:
Field "PNRInfo.successMessage" already exists in the schema. It cannot also be defined in this type extension.
```

## @__clientField(handle: " ... ")

> This directive is not intended for use by developers directly. To set a field handle in product code use a compiler plugin

([source](https://github.com/facebook/relay/blob/8f08aaad9dae241ba6706b39160b89f4ed00c5c8/packages/graphql-compiler/core/GraphQLParser.js#L86-L91))

Anyway, you can compute the client field value from other server field:

```graphql
fragment Example on Article {
  body

  # Relay is a bit broken now (see: https://github.com/facebook/relay/issues/2488)
  _: body @__clientField(handle: "draft")

  # this is a client field and it will contain uppercased `body` value
  draft
}
```

And you have to create the handler (https://facebook.github.io/relay/docs/en/relay-environment.html#adding-a-handlerprovider):

```js
const DraftHandler = {
  update(store, payload) {
    const record = store.get(payload.dataID);
    const content = record.getValue(payload.fieldKey);
    record.setValue(content.toUpperCase(), 'draft');
  }
};
```

More info: https://medium.com/@matt.krick/replacing-redux-with-relay-47ed085bfafe

# @match(onTypes: [ ... ]), @module( ... )

TODO: researching

- https://github.com/facebook/relay/blob/59d655027deff31f42b632a17c40162d377c964d/packages/relay-compiler/transforms/__tests__/__snapshots__/RelayMatchTransform-test.js.snap
- https://github.com/facebook/graphql/issues/488
- https://github.com/facebook/relay/blob/688dd4367cbe59ba223a027e3fcdc820455310e3/packages/relay-test-utils/testschema.graphql#L167
- https://github.com/mrtnzlml/relay/pull/32/files
- https://github.com/mrtnzlml/relay/pull/75/files

# Persisted queries

TODO

- https://github.com/facebook/relay/pull/2354 (awaiting)
- https://github.com/graphql/express-graphql/pull/109

# Uploadables

Sending normal GraphQL mutation is trivial:

```js
commitMutation(graphql`
  mutation TestMutation {
    test(input: "test")
  }
`);
```

It sends POST request with these headers:

```http
Accept: application/json
Content-type: application/json
User-Agent: Mozilla/5.0
 ...
```

And with this request payload:

```json
{"query":"mutation TestMutation {\n  test(input: \"test\")\n}\n","variables":{}}
```

We can use the same POST request to send our files as well. To do so, you have to use uploadables from Relay and `multipart/form-data` content type. Mutation is similar:

```js
commitMutation(
  graphql`
    mutation TestMutation {
      test(input: "test")
    }
  `,
  {
    uploadables: {
      file1: new File(["foo"], "foo.txt", {
        type: "text/plain"
      }),
      file2: new File(["bar"], "bar.txt", {
        type: "text/plain"
      })
    }
  }
);
```

With these headers:

```http
Accept: */*
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryshXxygBlT4ATOyhW
User-Agent: Mozilla/5.0
 ...
```

And with these form data:

```
------WebKitFormBoundaryshXxygBlT4ATOyhW
Content-Disposition: form-data; name="query"

mutation TestMutation {
  test(input: "test")
}

------WebKitFormBoundaryshXxygBlT4ATOyhW
Content-Disposition: form-data; name="variables"

{}
------WebKitFormBoundaryshXxygBlT4ATOyhW
Content-Disposition: form-data; name="file1"; filename="foo.txt"
Content-Type: text/plain

foo
------WebKitFormBoundaryshXxygBlT4ATOyhW
Content-Disposition: form-data; name="file2"; filename="bar.txt"
Content-Type: text/plain

bar
------WebKitFormBoundaryshXxygBlT4ATOyhW--
```

Plase note - it's good idea to create GraphQL type representing the file and send it in the query as well. Look at how Absinthe is doing it:

```
$ curl -X POST \\
-F query="{mutation uploadFile(users: \"users_csv\", metadata: \"metadata_json\")" \\
-F users_csv=@users.csv \\
-F metadata_json=@metadata.json \\
localhost:4000/graphql
```

They send the actual files and query as `multipart/form-data` as well but they require the file name (with underscore) in the query and it will fail if these two things do not match. Simple and elegant.

- https://github.com/facebook/relay/issues/1844#issuecomment-316893590
- https://hexdocs.pm/absinthe/file-uploads.html

# Caching

TODO: simple result cache, AsyncStorage-like cache

# RelayNetworkLogger

```js
import RelayNetworkLogger from 'relay-runtime/lib/RelayNetworkLogger'

import fetchFunction from './fetchFunction'
import subscribeFunction from './subscribeFunction'

const fetch = __DEV__
    ? RelayNetworkLogger.wrapFetch(fetchFunction)
    : fetchFunction

const subscribe = __DEV__
    ? RelayNetworkLogger.wrapSubscribe(subscribeFunction)
    : subscribeFunction

const network = Network.create(fetch, subscribe)
const source = new RecordSource()
const store = new Store(source)

const env = new Environment({
  network,
  store
})

export default env
```
