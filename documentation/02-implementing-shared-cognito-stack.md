# Implementing a Shared Cognito Stack

Why do this?:

- create one user pool (list of who can access the app), which all the applications will share
- therefore, no need to create new user pools for each new application that you create
- especially helpful when using one shared stack for an enterprise that wants the same set of users using their different applications
  - for example, UST creates a bunch of staff-only websites. It would be complicated to set up a new user pool (which basically says only staff are allowed to log in)
  - instead, create a shared user pool, where only staff are allowed to authenticate, but use this single user pool for all the different web apps
  - Also, to register a single user pool with ADFS SSO requires exchanging metadata urls (basically, information about your user pool) with the Microsoft Security people. Doing this once is ideal. Imagine creating a new ticket each time to create the same functionality you requested before 🤯.

## Steps to Implement a Shared Stack:

### 1. Decide on method of Separation of Groups & Access Controls:
- decide how to manage different levels of users on your application (Admins, General Users, etc)
- Read the documentation for [methods of separation of groups and access controls](../code-implementation-examples/groups-separation.md)

### 2. Create a Cognito Authentication Stack for your applications:
- deploy the AWS Resources to create a cognito user pool, domain, and all the resources necessary to have users log into your services.
- Read the documentation for [implementing a cognito authentication stack](../code-implementation-examples/implement-cognito-stack.md)