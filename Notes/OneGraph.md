# OneGraph

## Fragments

```graphql
fragment contributionFieldsUser on GitHubActor {
  login
  url
  avatarUrl
}
```

```graphql
fragment contributionFieldsDiscussion on GitHubDiscussion {
  id
  answerChosenAt
  createdAt
  lastEditedAt
  publishedAt
  updatedAt
  answerChosenBy {
    ...contributionFieldsUser
  }
  author {
    ...contributionFieldsUser
  }
  locked
  number
  title
  url
}
```

```graphql
fragment contributionFieldsIssue on GitHubIssue {
  id
  closedAt
  createdAt
  updatedAt
  lastEditedAt
  publishedAt
  labels(
    orderBy: { field: CREATED_AT, direction: DESC }
    last: 5
  ) {
    nodes {
      id
      name
      url
    }
  }
  state
  title
  url
}
```

```graphql
fragment contributionFieldsPullRequest on GitHubPullRequest {
  id
  closedAt
  createdAt
  lastEditedAt
  mergedAt
  publishedAt
  updatedAt
  author {
    ...contributionFieldsUser
  }
  isDraft
  labels(
    orderBy: { field: CREATED_AT, direction: DESC }
    last: 5
  ) {
    nodes {
      id
      name
      url
    }
  }
  mergeable
  merged
  mergedBy {
    ...contributionFieldsUser
  }
  number
  state
  title
  url
}
```

## Queries

```graphql
query ContributionsCollectionQuery($query: String!) {
  gitHub {
    search(query: $query, type: ISSUE, last: 10) {
      nodes {
        ... on GitHubPullRequest {
          ...contributionFieldsPullRequest
        }
        ... on GitHubIssue {
          ...contributionFieldsIssue
        }
        ... on GitHubDiscussion {
          ...contributionFieldsDiscussion
        }
      }
    }
  }
}
```

## Testing

```javascript
const ContributionsCollectionQueryVars = (owner, repo, target) => `repo:${owner}/${repo} involves:${target}`;

console.log(`\{
  "query": ${ContributionsCollectionQueryVars("finitesingularity", "tau", "mtfoley")}
\}`)
```