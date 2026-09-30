# API Reference

Complete reference for the REST and GraphQL APIs exposed by this backend. Endpoints, parameters, status codes and example payloads are derived from the controllers in `src/main/java/io/spring/api`, the GraphQL schema in `src/main/resources/schema/schema.graphqls`, and responses captured from the running application.

- [Conventions](#conventions)
- [Authentication](#authentication)
- [Errors](#errors)
- [Users](#users)
- [Profiles](#profiles)
- [Articles](#articles)
- [Favorites](#favorites)
- [Comments](#comments)
- [Tags](#tags)
- [GraphQL](#graphql)

## Conventions

| Item | Value |
| --- | --- |
| Base URL (local) | `http://localhost:8080` — endpoints are served from the root; there is **no** `/api` prefix |
| Content type | `application/json` |
| Request bodies | Must be wrapped in a root object: `{"user": {...}}`, `{"article": {...}}`, `{"comment": {...}}` (`spring.jackson.deserialization.UNWRAP_ROOT_VALUE=true`). Unwrapped bodies are rejected with `400`. |
| Timestamps | ISO-8601 in UTC with milliseconds, e.g. `2026-09-30T19:49:51.304Z` |
| IDs | UUID strings |
| Pagination (REST) | `offset` (default `0`) and `limit` (default `20`, max `100`) |

**Auth column legend** used below:

- **None** — no token needed.
- **Optional** — works anonymously; if a valid token is sent, per-user fields (`favorited`, `following`) are computed for that user.
- **Required** — a valid token is required, otherwise `401`.

### Endpoint summary

| Method | Path | Auth | Success |
| --- | --- | --- | --- |
| `POST` | `/users` | None | `201` |
| `POST` | `/users/login` | None | `200` |
| `GET` | `/user` | Required | `200` |
| `PUT` | `/user` | Required | `200` |
| `GET` | `/profiles/{username}` | Optional | `200` |
| `POST` | `/profiles/{username}/follow` | Required | `200` |
| `DELETE` | `/profiles/{username}/follow` | Required | `200` |
| `GET` | `/articles` | Optional | `200` |
| `GET` | `/articles/feed` | Required | `200` |
| `POST` | `/articles` | Required | `200` |
| `GET` | `/articles/{slug}` | Optional | `200` |
| `PUT` | `/articles/{slug}` | Required (author) | `200` |
| `DELETE` | `/articles/{slug}` | Required (author) | `204` |
| `POST` | `/articles/{slug}/favorite` | Required | `200` |
| `DELETE` | `/articles/{slug}/favorite` | Required | `200` |
| `GET` | `/articles/{slug}/comments` | Optional | `200` |
| `POST` | `/articles/{slug}/comments` | Required | `201` |
| `DELETE` | `/articles/{slug}/comments/{id}` | Required (comment or article author) | `204` |
| `GET` | `/tags` | None | `200` |
| `POST` | `/graphql` | Per operation | `200` |
| `GET` | `/graphiql` | None | `200` (HTML UI) |

## Authentication

Obtain a token from `POST /users` or `POST /users/login`, then send it on subsequent requests:

```http
Authorization: Token eyJhbGciOiJIUzUxMiJ9...
```

- The filter only looks at the second space-separated part of the header, so `Bearer <jwt>` also works.
- Tokens are HS512-signed JWTs with `sub` = user id and `exp` = issue time + `jwt.sessionTime` (default 86400 s = 24 h). There is no refresh endpoint; log in again to get a new token.
- A missing, malformed or expired token on a **Required** endpoint returns `401` with an empty body. On **Optional** endpoints an invalid token is ignored and the request is treated as anonymous.

## Errors

| Status | When | Body |
| --- | --- | --- |
| `400` | Malformed JSON or body not wrapped in the expected root object | Spring Boot default (or empty) |
| `401` | Missing/invalid token on a protected endpoint | *(empty)* |
| `403` | Authenticated but not allowed (editing/deleting someone else's article, deleting a comment you may not delete) | Spring Boot default error JSON |
| `404` | Unknown article slug, username, or comment id; unfollowing a user you do not follow | Spring Boot default error JSON |
| `422` | Validation failure | `{"errors": {"<field>": ["<message>", ...]}}` |
| `422` | Wrong email/password on login | `{"message": "invalid email or password"}` |
| `500` | Unexpected server error (see [known issue](#create-article)) | Spring Boot default error JSON |

Validation error example:

```json
{"errors":{"password":["can't be empty"],"email":["duplicated email"],"username":["duplicated username"]}}
```

Spring Boot default error JSON example (`403`):

```json
{"timestamp":"2026-09-30T19:49:51.364+00:00","status":403,"error":"Forbidden","path":"/articles/post-a"}
```

---

## Users

### Register

`POST /users` — Auth: **None** — `UsersApi.createUser`

Request:

```json
{
  "user": {
    "email": "sam@x.io",
    "username": "sam",
    "password": "samsamsam"
  }
}
```

| Field | Rules |
| --- | --- |
| `email` | required, valid email, not already registered (`duplicated email`) |
| `username` | required, not already taken (`duplicated username`) |
| `password` | required |

Response `201 Created`:

```json
{
  "user": {
    "email": "sam@x.io",
    "username": "sam",
    "bio": "",
    "image": "https://static.productionready.io/images/smiley-cyrus.jpg",
    "token": "eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJjODVm...."
  }
}
```

New users get an empty `bio` and the `image.default` property as their `image`.

### Login

`POST /users/login` — Auth: **None** — `UsersApi.userLogin`

Request:

```json
{
  "user": {
    "email": "sam@x.io",
    "password": "samsamsam"
  }
}
```

| Field | Rules |
| --- | --- |
| `email` | required, valid email (`should be an email`, `can't be empty`) |
| `password` | required (`can't be empty`) |

Response `200 OK`: same shape as registration (`{"user": {..., "token": "..."}}`).

Wrong credentials → `422 {"message":"invalid email or password"}`.

### Get current user

`GET /user` — Auth: **Required** — `CurrentUserApi.currentUser`

Response `200 OK`:

```json
{
  "user": {
    "email": "sam@x.io",
    "username": "sam",
    "bio": "",
    "image": "https://static.productionready.io/images/smiley-cyrus.jpg",
    "token": "<the token you sent>"
  }
}
```

### Update current user

`PUT /user` — Auth: **Required** — `CurrentUserApi.updateProfile`

All fields are optional; empty or missing fields are left unchanged.

```json
{
  "user": {
    "email": "sam@x.io",
    "username": "sam",
    "password": "newpassword",
    "bio": "I like to skateboard",
    "image": "https://i.stack.imgur.com/xHWG8.jpg"
  }
}
```

| Field | Rules |
| --- | --- |
| `email` | valid email if present (`should be an email`); must not belong to another user (`email already exist`) |
| `username` | must not belong to another user (`username already exist`) |
| `password`, `bio`, `image` | free-form |

Response `200 OK`:

```json
{
  "user": {
    "email": "sam@x.io",
    "username": "sam",
    "bio": "I like to skateboard",
    "image": "https://i.stack.imgur.com/xHWG8.jpg",
    "token": "<the token you sent>"
  }
}
```

---

## Profiles

Profile object:

```json
{
  "profile": {
    "username": "jake",
    "bio": "",
    "image": "https://static.productionready.io/images/smiley-cyrus.jpg",
    "following": false
  }
}
```

### Get profile

`GET /profiles/{username}` — Auth: **Optional** — `ProfileApi.getProfile`

Response `200 OK`: profile object. `following` is `false` for anonymous callers. Unknown username → `404`.

### Follow user

`POST /profiles/{username}/follow` — Auth: **Required** — `ProfileApi.follow`

No body. Response `200 OK`: profile object with `"following": true`. Unknown username → `404`.

### Unfollow user

`DELETE /profiles/{username}/follow` — Auth: **Required** — `ProfileApi.unfollow`

No body. Response `200 OK`: profile object with `"following": false`. Unknown username, or not currently following that user → `404`.

---

## Articles

Article object (single-article responses wrap it as `{"article": {...}}`):

```json
{
  "article": {
    "id": "2f90acf4-4f34-44c3-85d5-4487e0ec4e2c",
    "slug": "how-to-train-your-dragon",
    "title": "How to train your dragon",
    "description": "Ever wonder how?",
    "body": "You have to believe",
    "favorited": false,
    "favoritesCount": 0,
    "createdAt": "2026-09-30T19:49:51.304Z",
    "updatedAt": "2026-09-30T19:49:51.304Z",
    "tagList": ["training", "dragons"],
    "cursor": {"data": "2026-09-30T19:49:51.304Z"},
    "author": {
      "username": "sam",
      "bio": "I like to skateboard",
      "image": "https://i.stack.imgur.com/xHWG8.jpg",
      "following": false
    }
  }
}
```

`id` and `cursor` are extra fields beyond the RealWorld spec (used internally for GraphQL pagination); clients can ignore them. `tagList` order is not guaranteed.

Multiple-article responses:

```json
{
  "articles": [ { "...": "article object without the 'article' wrapper" } ],
  "articlesCount": 4
}
```

`articlesCount` is the total number of matches, not the page size.

### List articles

`GET /articles` — Auth: **Optional** — `ArticlesApi.getArticles`

| Query param | Default | Description |
| --- | --- | --- |
| `tag` | — | Only articles with this tag |
| `author` | — | Only articles by this username |
| `favorited` | — | Only articles favorited by this username |
| `offset` | `0` | Number of articles to skip |
| `limit` | `20` | Page size (max `100`; values ≤ 0 fall back to the default) |

Results are ordered by creation time, newest first.

```bash
curl 'http://localhost:8080/articles?tag=dragons&limit=10&offset=0'
```

Response `200 OK`: multiple-article response.

### Feed

`GET /articles/feed` — Auth: **Required** — `ArticlesApi.getFeed`

Articles written by users the caller follows, newest first. Query params: `offset` (default `0`), `limit` (default `20`, max `100`).

Response `200 OK`: multiple-article response (`{"articles":[],"articlesCount":0}` if the caller follows nobody).

### Create article

`POST /articles` — Auth: **Required** — `ArticlesApi.createArticle`

```json
{
  "article": {
    "title": "How to train your dragon",
    "description": "Ever wonder how?",
    "body": "You have to believe",
    "tagList": ["dragons", "training"]
  }
}
```

| Field | Rules |
| --- | --- |
| `title` | required; its slug must not already exist (`article name exists`) |
| `description` | required (`can't be empty`) |
| `body` | required (`can't be empty`) |
| `tagList` | array of strings; duplicates are collapsed. **Send `[]` when there are no tags** — omitting the field currently causes a `500` (`NullPointerException` in the `Article` constructor). |

Response `200 OK` (not `201`): single article object. The slug is derived from the title — lower-cased with whitespace and some punctuation replaced by `-` (e.g. `"Did you train your dragon?"` → `did-you-train-your-dragon-`).

### Get article

`GET /articles/{slug}` — Auth: **Optional** — `ArticleApi.article`

Response `200 OK`: single article object. Unknown slug → `404`.

### Update article

`PUT /articles/{slug}` — Auth: **Required, author only** — `ArticleApi.updateArticle`

All fields optional; empty or missing fields are left unchanged. Tags cannot be changed through this endpoint.

```json
{
  "article": {
    "title": "Did you train your dragon?",
    "description": "optional",
    "body": "optional"
  }
}
```

Response `200 OK`: the updated article object. Changing `title` changes the `slug`, so use the returned slug for later calls. Unknown slug → `404`; caller is not the author → `403`.

Note: `updatedAt` is not refreshed on update (it remains equal to `createdAt`).

### Delete article

`DELETE /articles/{slug}` — Auth: **Required, author only** — `ArticleApi.deleteArticle`

Response `204 No Content`. Unknown slug → `404`; caller is not the author → `403`.

---

## Favorites

### Favorite article

`POST /articles/{slug}/favorite` — Auth: **Required** — `ArticleFavoriteApi.favoriteArticle`

No body. Idempotent. Response `200 OK`: single article object with `"favorited": true` and updated `favoritesCount`. Unknown slug → `404`.

### Unfavorite article

`DELETE /articles/{slug}/favorite` — Auth: **Required** — `ArticleFavoriteApi.unfavoriteArticle`

No body. Response `200 OK`: single article object with `"favorited": false`. Unknown slug → `404`.

---

## Comments

Comment object:

```json
{
  "comment": {
    "id": "4b410084-90aa-4b9e-8c44-15fbac5e106a",
    "body": "Thank you so much!",
    "createdAt": "2026-09-30T19:49:51.334Z",
    "updatedAt": "2026-09-30T19:49:51.334Z",
    "cursor": {"data": "2026-09-30T19:49:51.334Z"},
    "author": {
      "username": "sam",
      "bio": "I like to skateboard",
      "image": "https://i.stack.imgur.com/xHWG8.jpg",
      "following": false
    }
  }
}
```

### List comments

`GET /articles/{slug}/comments` — Auth: **Optional** — `CommentsApi.getComments`

Response `200 OK`:

```json
{
  "comments": [
    {
      "id": "4b410084-90aa-4b9e-8c44-15fbac5e106a",
      "body": "Thank you so much!",
      "createdAt": "2026-09-30T19:49:51.334Z",
      "updatedAt": "2026-09-30T19:49:51.334Z",
      "cursor": {"data": "2026-09-30T19:49:51.334Z"},
      "author": {"username": "sam", "bio": "I like to skateboard", "image": "https://i.stack.imgur.com/xHWG8.jpg", "following": false}
    }
  ]
}
```

Unknown slug → `404`. No pagination.

### Add comment

`POST /articles/{slug}/comments` — Auth: **Required** — `CommentsApi.createComment`

```json
{
  "comment": {
    "body": "Thank you so much!"
  }
}
```

| Field | Rules |
| --- | --- |
| `body` | required (`can't be empty`) |

Response `201 Created`: comment object. Unknown slug → `404`.

### Delete comment

`DELETE /articles/{slug}/comments/{id}` — Auth: **Required** (comment author **or** article author) — `CommentsApi.deleteComment`

Response `204 No Content`. Unknown slug or comment id → `404`; caller is neither the comment author nor the article author → `403`.

---

## Tags

### List tags

`GET /tags` — Auth: **None** — `TagsApi.getTags`

Response `200 OK`:

```json
{"tags":["graphql","x","training","dragons"]}
```

Returns every tag name stored in the database (order not guaranteed).

---

## GraphQL

| Item | Value |
| --- | --- |
| Endpoint | `POST /graphql` with `{"query": "...", "variables": {...}}` |
| Interactive UI | `GET /graphiql` |
| Schema | `src/main/resources/schema/schema.graphqls` |
| Auth | Same `Authorization: Token <jwt>` header. The HTTP endpoint is open; individual resolvers require a user. |

### Queries

| Field | Arguments | Returns | Auth |
| --- | --- | --- | --- |
| `article` | `slug: String!` | `Article` | Optional |
| `articles` | `first, after, last, before, authoredBy, favoritedBy, withTag` | `ArticlesConnection` | Optional |
| `me` | — | `User` | Required (`Authorization` header must be present) |
| `feed` | `first, after, last, before` | `ArticlesConnection` | Required |
| `profile` | `username: String!` | `ProfilePayload` | Optional |
| `tags` | — | `[String]` | None |

### Mutations

| Field | Arguments | Returns | Auth |
| --- | --- | --- | --- |
| `createUser` | `input: CreateUserInput {email, username, password}` | `UserResult` (`UserPayload \| Error`) | None |
| `login` | `email: String!, password: String!` | `UserPayload` | None |
| `updateUser` | `changes: UpdateUserInput! {email, username, password, image, bio}` | `UserPayload` | Required |
| `followUser` / `unfollowUser` | `username: String!` | `ProfilePayload` | Required |
| `createArticle` | `input: CreateArticleInput! {title, description, body, tagList}` | `ArticlePayload` | Required |
| `updateArticle` | `slug: String!, changes: UpdateArticleInput! {title, description, body}` | `ArticlePayload` | Required (author) |
| `favoriteArticle` / `unfavoriteArticle` | `slug: String!` | `ArticlePayload` | Required |
| `deleteArticle` | `slug: String!` | `DeletionStatus {success}` | Required (author) |
| `addComment` | `slug: String!, body: String!` | `CommentPayload` | Required |
| `deleteComment` | `slug: String!, id: ID!` | `DeletionStatus {success}` | Required (comment or article author) |

### Types (abridged)

```graphql
type Article {
  author: Profile!
  body: String!
  comments(first: Int, after: String, last: Int, before: String): CommentsConnection
  createdAt: String!
  description: String!
  favorited: Boolean!
  favoritesCount: Int!
  slug: String!
  tagList: [String],
  title: String!
  updatedAt: String!
}

type Profile {
  username: String!
  bio: String
  following: Boolean!
  image: String
  articles(first: Int, after: String, last: Int, before: String): ArticlesConnection
  favorites(first: Int, after: String, last: Int, before: String): ArticlesConnection
  feed(first: Int, after: String, last: Int, before: String): ArticlesConnection
}

type User {
  email: String!
  profile: Profile!
  token: String!
  username: String!
}
```

Connections follow the Relay shape: `edges { cursor node { ... } }` and `pageInfo { startCursor endCursor hasNextPage hasPreviousPage }`. Pass exactly one of `first` or `last` (max 1000); omitting both returns an error (`errorType: INTERNAL`). Cursors are epoch-millisecond strings.

### Examples

Login:

```graphql
mutation {
  login(email: "sam@x.io", password: "samsamsam") {
    user { email username token }
  }
}
```

```json
{"data":{"login":{"user":{"email":"sam@x.io","username":"sam","token":"<jwt>"}}}}
```

Register with union handling:

```graphql
mutation {
  createUser(input: {email: "sam@x.io", username: "sam2", password: "p"}) {
    ... on UserPayload { user { username token } }
    ... on Error { message errors { key value } }
  }
}
```

```json
{"data":{"createUser":{"message":"BAD_REQUEST","errors":[{"key":"email","value":["duplicated email"]}]}}}
```

Paginated article list:

```graphql
{
  articles(first: 2, withTag: "x") {
    edges { cursor node { slug favoritesCount author { username } } }
    pageInfo { hasNextPage endCursor }
  }
}
```

```json
{"data":{"articles":{"edges":[{"cursor":"1790797752186","node":{"slug":"post-c","favoritesCount":0,"author":{"username":"jake"}}},{"cursor":"1790797752175","node":{"slug":"post-b","favoritesCount":0,"author":{"username":"jake"}}}],"pageInfo":{"hasNextPage":true,"endCursor":"1790797752175"}}}}
```

Create an article (with `Authorization: Token <jwt>`):

```graphql
mutation {
  createArticle(input: {title: "GraphQL rocks", description: "d", body: "b", tagList: ["graphql"]}) {
    article { slug title tagList favorited author { username } }
  }
}
```

```json
{"data":{"createArticle":{"article":{"slug":"graphql-rocks","title":"GraphQL rocks","tagList":["graphql"],"favorited":false,"author":{"username":"jake"}}}}}
```

Add a comment (with `Authorization: Token <jwt>`):

```graphql
mutation {
  addComment(slug: "graphql-rocks", body: "Nice") {
    comment { id body author { username } }
  }
}
```

```json
{"data":{"addComment":{"comment":{"id":"93a03279-96f6-4fb4-a03d-639d8df3d8b4","body":"Nice","author":{"username":"jake"}}}}}
```

Using curl:

```bash
curl -X POST http://localhost:8080/graphql \
  -H 'Content-Type: application/json' \
  -H "Authorization: Token $TOKEN" \
  -d '{"query":"{ me { username email } }"}'
```

### GraphQL errors

Errors are returned with HTTP `200` in the standard `errors` array; `extensions.errorType` indicates the category.

| Situation | `errorType` | Example `message` / `extensions` |
| --- | --- | --- |
| Validation failure (e.g. `createArticle` with an existing title) | `BAD_REQUEST` | `extensions: {"title":["article name exists"],"description":["can't be empty"], ...}` |
| Wrong credentials on `login` | `UNAUTHENTICATED` | `invalid email or password` |
| Mutation requiring auth called anonymously | `INTERNAL` | `io.spring.graphql.exception.AuthenticationException: null` |
| `me` without an `Authorization` header | `INTERNAL` | `Required header 'Authorization' was not provided` |
| Connection field without `first`/`last` | `INTERNAL` | `java.lang.IllegalArgumentException: ...` |
