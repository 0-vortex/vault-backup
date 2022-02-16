# GraphiQL implementations

## OneGraph

- https://github.com/OneGraph/onegraph-apollo-server-auth-example
- https://github.com/OneGraph/authguardian-react-starter
- https://github.com/OneGraph/graphql-codex
- https://github.com/OneGraph/graphiql-explorer-example
- https://github.com/OneGraph/graphiql-with-extensions
- https://www.onegraph.com/graphiql?shortenedId=MYE8NX

## GraphiQL

- https://github.com/graphql/graphiql/blob/main/packages/graphiql/README.md
- https://github.com/graphql/graphiql/blob/main/packages/graphiql-toolkit/docs/create-fetcher.md


## Bugs

```graphql
query getData {
  organization(login: "open-sauced") {
    id,
    login
    teams (first: 10) {
      edges {
        node {
          id
          name
        }
      }
    }
  }
}

mutation createOpenSaucedRepository {
  __typename
  createRepository(input: {
    name: "semantic-release-conventional-config",
    ownerId: "MDEyOk9yZ2FuaXphdGlvbjU3NTY4NTk4",
    teamId: "MDQ6VGVhbTM4OTU0MzM",
    visibility: PUBLIC
  }) {
    repository {
      id
    }
  }
}
```


## explore.opensauced.pizza

```shell
npx octoherd-script-sync-repo-settings \
  --template "open-sauced/open-sauced" \
  -T $GH_TOKEN \
  -R "open-sauced/explore.opensauced.pizza"
```

```shell
npx octoherd-script-copy-labels \
  --template "open-sauced/open-sauced" \
  -T $GH_TOKEN \
  -R "open-sauced/explore.opensauced.pizza" 
```

```shell
npx @octoherd/script-sync-branch-protections \
  --template "open-sauced/open-sauced" \
  -T $GH_TOKEN \
  -R "open-sauced/explore.opensauced.pizza"
```
