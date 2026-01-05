# Implementing a Shared Cognito Stack

Why do this?:
- create one user pool (list of who can access the app), which all the applications will share
- therefore, no need to create new user pools for each new application that you create
- especially helpful when using one shared stack for an enterprise that wants the same set of users using their different applications
    - for example, UST creates a bunch of staff-only websites. It would be complicated to set up a new user pool (which basically says only staff are allowed to log in)
    - instead, create a shared user pool, where only staff are allowed to authenticate, but use this single user pool for all the different 
    - Also, to register a single user pool with ADFS SSO requires exchanging metadata urls (basically, information about your user pool) with the Microsoft Security people. Doing this once is ideal. Imagine creating a new ticket each time to create the same functionality you requested before 🤯.


## Note: Separation of Groups:
- This is the part where I need the help of Chad to decide which type of "sepration of groups" are we going to use
- What i mean by separation is:
    - Say two sets of people need access to two sets of different web apps
    - Suppose the first group of web apps is staff-only, and the other set is everyone with UST Email
    - To separate these users (prevent non-staff from accessing staff only website), we need to decide between two different options

What are these two options (pick your poison type situation):
1. Create a new cognito stack for a particular group of people
    - Pros: logical separation, no extra logic on the frontend's side, easier for the developer (if the stack is already set up for them)
    - Cons: more stacks to maintain, more tickets need to be submitted to Azure Security people

2. Use one single shared stack, but in the token received from SSO, specify the position of the user:
    - say, the token received after authentication to Microsoft SSO says the user is a "staff", then the frontend says okay you are allowed to use the app -> authenticated
    - but, if the token says the user is a "student", then the frontend says you're not allowed to use the app -> not authenticated
    - Pros: Not requesting tickets for every different type of user group
    - Cons: Validation logic needs to be added to the frontend to check the user's permission level as specified in the auth token