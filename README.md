# react-apollo-hooks

Use [Apollo Client](https://github.com/apollographql/apollo-client) as React
[hooks](https://reactjs.org/docs/hooks-intro.html).

# Installation

`npm install @jeremydurnell/react-apollo-hooks`

Or if using [yarn](https://yarnpkg.com/en/)

`yarn add @jeremydurnell/react-apollo-hooks`

# Example

<https://codesandbox.io/s/8819w85jn9> is a port of Pupstagram sample app to
react-apollo-hooks.

> NOTE: This example is outdated and may not work properly. Support for React Suspense was removed in version 0.4.6 of this package.

# API

## ApolloProvider

Similar to
[ApolloProvider from react-apollo](https://www.apollographql.com/docs/react/essentials/get-started.html#creating-provider).
Both packages can be used together, if you want to try out using hooks and
retain `Query`, `Mutation`, `Subscription`, etc. HOCs from `react-apollo`
without having to rewrite existing components throughout your app.

In order for this package to work, you need to wrap your component tree with
`ApolloProvider` at an appropriate level, encapsulating all components which
will use hooks.

### Standalone usage

If you would like to use this package standalone, this can be done with:

```javascript
import React from 'react';
import { render } from 'react-dom';

import { ApolloProvider } from 'react-apollo-hooks';

const client = ... // create Apollo client

const App = () => (
  <ApolloProvider client={client}>
    <MyRootComponent />
  </ApolloProvider>
);

render(<App />, document.getElementById('root'));
```

### Usage with react-apollo

To use with `react-apollo`'s `ApolloProvider` already present in your project:

```javascript
import React from 'react';
import { render } from 'react-dom';

import { ApolloProvider } from 'react-apollo';
import { ApolloProvider as ApolloHooksProvider } from 'react-apollo-hooks';

const client = ... // create Apollo client

const App = () => (
  <ApolloProvider client={client}>
    <ApolloHooksProvider client={client}>
      <MyRootComponent />
   </ApolloHooksProvider>
  </ApolloProvider>
);

render(<App />, document.getElementById('root'));
```

## useQuery

```javascript
import gql from 'graphql-tag';
import { useQuery } from 'react-apollo-hooks';

const GET_DOGS = gql`
  {
    dogs {
      id
      breed
    }
  }
`;

const Dogs = () => {
  const { data, error, loading } = useQuery(GET_DOGS);
  if (loading) {
    return <div>Loading...</div>;
  };
  if (error) {
    return <div>Error! {error.message}</div>;
  };

  return (
    <ul>
      {data.dogs.map(dog => (
        <li key={dog.id}>{dog.breed}</li>
      ))}
    </ul>
  );
};

```

## useMutation

```javascript
import gql from 'graphql-tag';
import { useMutation } from 'react-apollo-hooks';

const TOGGLE_LIKED_PHOTO = gql`
  mutation toggleLikedPhoto($id: String!) {
    toggleLikedPhoto(id: $id) @client
  }
`;

const DogWithLikes = ({ url, imageId, isLiked }) => {
  const toggleLike = useMutation(TOGGLE_LIKED_PHOTO, {
    variables: { id: imageId },
  });
  return (
    <div>
      <img src={url} />
      <button onClick={toggleLike}>{isLiked ? 'Stop liking' : 'like'}</button>
    </div>
  );
};
```

You can provide any
[mutation options](https://www.apollographql.com/docs/react/api/apollo-client.html#ApolloClient.mutate)
as an argument to the `useMutation` hook or to the function returned by it, e.
g.:

```javascript
function AddTaskForm() {
  const inputRef = useRef();
  const addTask = useMutation(ADD_TASK_MUTATION, {
    update: (proxy, mutationResult) => {
      /* your custom update logic */
    },
    variables: {
      text: inputRef.current.value,
    },
  });

  return (
    <form>
      <input ref={inputRef} />
      <button onClick={addTask}>Add task</button>
    </form>
  );
}
```

Or:

```javascript
function TasksWithMutation() {
  const toggleTask = useMutation(TOGGLE_TASK_MUTATION);

  return (
    <TaskList
      onChange={task => toggleTask({ variables: { taskId: task.id } })}
      tasks={data.tasks}
    />
  );
}
```

## useSubscription

If you are just interested in the last subscription value sent by the
server (e. g. a global indicator showing how many new messages you have in an
instant messenger app) you can use `useSubscription` hook in this form:

```javascript
const NEW_MESSAGES_COUNT_CHANGED_SUBSCRIPTION = gql`
  subscription onNewMessagesCountChanged($repoFullName: String!) {
    newMessagesCount
  }
`;

const NewMessagesIndicator = () => {
  const { data, error, loading } = useSubscription(
    NEW_MESSAGES_COUNT_CHANGED_SUBSCRIPTION
  );

  if (loading) {
    return <div>Loading...</div>;
  };

  if (error) {
    return <div>Error! {error.message}</div>;
  };

  return <div>{data.newMessagesCount} new messages</div>;
}
```

For more advanced use cases, e. g. when you'd like to show a notification
to the user or modify the Apollo cache (e. g. you'd like to show a new comment
on a blog post page for a user visiting it just after it was created) you can
use the `onSubscriptionData` callback:

```javascript
const { data, error, loading } = useSubscription(MY_SUBSCRIPTION, {
  variables: {
    // ...
  },
  onSubscriptionData: ({ client, subscriptionData }) => {
    // Optional callback which provides you access to the new subscription
    // data and the Apollo client. You can use methods of the client to update
    // the Apollo cache:
    // https://www.apollographql.com/docs/react/advanced/caching.html#direct
  }
  // ... rest options
});
```

In some cases you might want to subscribe only after you have all the information available, eg. only when user has selected necessary filters. Since hooks cannot be used conditionally, it would lead to unnecessary complicated patterns.

Instead, you can use the `skip` option which turns the subsciption dormant until toggled again. It will also unsubscribe if there was any previous subscription active and throw away previous result.

## useApolloClient

```javascript
const MyComponent = () => {
  const client = useApolloClient();
  // now you have access to the Apollo client
};
```

# Testing

An example showing how to test components using react-apollo-hooks:
<https://github.com/trojanowski/react-apollo-hooks-sample-test>

# Server-side rendering

react-apollo-hooks supports server-side rendering with the `getMarkupFromTree`
function. Example usage:

```javascript
import express from 'express';
import { ApolloProvider, getMarkupFromTree } from 'react-apollo-hooks';
import { renderToString } from 'react-dom/server';

const HELLO_QUERY = gql`
  query HelloQuery {
    hello
  }
`;

function Hello() {
  const { data } = useQuery(HELLO_QUERY);

  return <p>{data.message}</p>;
}

const app = express();

app.get('/', async (req, res) => {
  const client = createYourApolloClient();
  const renderedHtml = await getMarkupFromTree({
    renderFunction: renderToString,
    tree: (
      <ApolloProvider client={client}>
        <Hello />
      </ApolloProvider>
    ),
  });
  res.send(renderedHtml);
});
```


