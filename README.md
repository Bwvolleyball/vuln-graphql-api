# GraphQL Demo API

A simple GraphQL API demonstrating several common vulnerabilities.

## Requirements

Node, NPM, and Python (for "SSRF")

## Setup

```
# Install all dependencies.
npm install
# Build the TypeScript source.
npm run tsc
# Create the database and seed it with random users and comments.
npm run sequelize db:migrate
npm run sequelize db:seed:all
```

## Running

To run the main API:

```
node build/app.js
```

Then, from the `static` directory, start the Python server that represents a REST API backend:

```
python3 -m http.server --bind 127.0.0.1 8081
```

## Usage

The GraphQL API is available on port 3000. Visiting the homepage will take you to a GraphIQL IDE for exploration.

The API provides a simple social media/blog system. Users are able to make and view posts from other users, and they can be marked private so that they can't be seen by other users.

### Queries

me: returns information (id, firstName, lastName, list of posts) about the currently logged in user. On first access to the API, a user is assigned to your session.

allUsers: provides a list of all users. Has the same information as `me` -- but filters out private posts from other users.

post(id): Returns a specific post by ID. -- `(id, title, content, author)`.

search(query): Returns posts that contain the `query` string within their contents. E.g., search("asdf") would return any post containing "asdf".

getAsset(name): Retrieves an asset from the backend REST API and returns its contents as a string.


### Mutations

register(username, password, firstName, lastName): Registers a new account. Does not log into the new account.

login(username, password): Logs you into the account given by `username` if the password is correct.

createPost(title, content, public): Creates a new post.

superSecretPrivateMutation(command): ???

## Bugs

### Authorization

Authorization is handled by field resolvers. As a result, forgetting to check authorization in one location leads to authorization bugs.

In this API, private posts are directly accessible through the `post(id)` query.

```
query {
    post("1") {
        public
    }
}
```

### Expensive Queries

As API users are able to specify what they want, a massive, complicated-to-execute query can be sent: 

```
query {
  allUsers{
    posts
    {
      content
      author
      {
        posts
        {
          content
          author
          {
            firstName
            lastName
          }
        }
      }
    }
  }
}
```

### "Hidden" Mutations

Introspection reveals all. If introspection is enabled, any hidden/private queries and mutations can be found by sending an introspection query.

GraphIQL automatically does this, so hidden queries show up in it.

This API has a hidden mutation that exposes command execution.

### SSRF to Backend APIs

Many GraphQL APIs proxy some endpoints to external REST APIs. In many cases, they fail to validate the ID or other URL parameters, allowing for the endpoint reached to be modified.

`getAssets(name)` makes a request to a local HTTP server. However, as the field is not validated, directory traversal can be used to read files outside of the intended `assets` folder.

### SQLi (bonus?)

I have seen SQL injection in GraphQL APIs before, so this type of vulnerability is never going away.

The `search(query)` uses the query to build a SQL `LIKE` clause, and it does so unsafely. 
