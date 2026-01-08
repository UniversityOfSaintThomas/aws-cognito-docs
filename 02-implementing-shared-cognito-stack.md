# Implementing a Shared Cognito Stack

Why do this?:

- create one user pool (list of who can access the app), which all the applications will share
- therefore, no need to create new user pools for each new application that you create
- especially helpful when using one shared stack for an enterprise that wants the same set of users using their different applications
  - for example, UST creates a bunch of staff-only websites. It would be complicated to set up a new user pool (which basically says only staff are allowed to log in)
  - instead, create a shared user pool, where only staff are allowed to authenticate, but use this single user pool for all the different web apps
  - Also, to register a single user pool with ADFS SSO requires exchanging metadata urls (basically, information about your user pool) with the Microsoft Security people. Doing this once is ideal. Imagine creating a new ticket each time to create the same functionality you requested before 🤯.

## Separation of Groups (Handling Admins vs. General Users)

When you need to separate different types of users (e.g., Staff vs. Students, or Admins vs. Public), we have two recommended approaches:

### Option 1: Single Stack with Cognito Groups (Recommended for most apps)

Use one User Pool for everyone, but sort users into **Cognito Groups**.

**How it works:**

1. Create groups inside your User Pool (e.g., `Admins`, `GeneralUsers`). Option to make groups found in Cognito > User Management > Groups (Or use Cloudformation Template, recommended and declaration shown below:
```yaml
Resources:
# 1. Create the User Pool (standard)
MyUserPool:
   Type: AWS::Cognito::UserPool
   Properties: 
   UserPoolName: "MyUniversityPool"

# 2. Define the 'Admins' Group
AdminGroup:
   Type: AWS::Cognito::UserPoolGroup
   Properties:
   GroupName: "Admins"
   UserPoolId: !Ref MyUserPool
   Description: "University Staff and IT Admins"
   Precedence: 0 # Higher precedence overrides lower ones if IAM roles conflict

# 3. Define the 'GeneralUsers' Group
GeneralGroup:
   Type: AWS::Cognito::UserPoolGroup
   Properties:
   GroupName: "GeneralUsers"
   UserPoolId: !Ref MyUserPool
   Description: "Standard Student Access"
```

Note:
- There is a limit to the number of groups that can exist in a user pool. Refer [AWS Quota Documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html).
- Refer to the official [Cognito User Groups](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html) documentation to see generalized information related to User Groups.

2. Assign users to these groups. You can do this manually in the AWS Console, or **automatically** by checking an external "Admin List" (like a DynamoDB table) during sign-up.

#### Example: Automated Group Assignment via Lambda & DynamoDB (This is AI-Generated code example, need to verify before someone uses this exact code snippet)
This architecture assigns the "Admins" group only if the user's email exists in a specific DynamoDB table. Everyone else gets "GeneralUsers".
- prath's note: if I were making this lambda function, I'd have the logic in an actual .js file with a lambda handler. But this also seems like a valid approach i really don't know would have to confirm

```yaml
Resources:
  # 1. DynamoDB Table storing the list of Admin emails
  AdminWhitelistTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: "AdminWhitelist"
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: "email"
          AttributeType: "S"
      KeySchema:
        - AttributeName: "email"
          KeyType: "HASH"

  # 2. Permissions for Lambda to Read DB and Update Cognito
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: { Service: [ 'lambda.amazonaws.com' ] }
            Action: [ 'sts:AssumeRole' ]
      Policies:
        - PolicyName: AdminGroupLogicPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: ['dynamodb:GetItem']
                Resource: !GetAtt AdminWhitelistTable.Arn
              - Effect: Allow
                Action: ['cognito-idp:AdminAddUserToGroup']
                Resource: !Sub 'arn:aws:cognito-idp:${AWS::Region}:${AWS::AccountId}:userpool/${MyUserPool}'
              - Effect: Allow
                Action: ['logs:CreateLogGroup', 'logs:CreateLogStream', 'logs:PutLogEvents']
                Resource: '*'

  # 3. The Lambda Function that does the sorting
  AssignGroupFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: nodejs20.x
      Handler: index.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Environment:
        Variables:
          ADMIN_TABLE_NAME: !Ref AdminWhitelistTable
      Code:
        ZipFile: |
          const { CognitoIdentityProviderClient, AdminAddUserToGroupCommand } = require("@aws-sdk/client-cognito-identity-provider");
          const { DynamoDBClient, GetItemCommand } = require("@aws-sdk/client-dynamodb");
          
          const cognitoClient = new CognitoIdentityProviderClient();
          const ddbClient = new DynamoDBClient();

          exports.handler = async (event) => {
            const email = event.request.userAttributes.email;
            const userPoolId = event.userPoolId;
            const username = event.userName;
            
            // Default to General Group
            let targetGroup = "GeneralUsers"; 

            try {
              // Check if email exists in DynamoDB Admin Table
              const ddbResponse = await ddbClient.send(new GetItemCommand({
                TableName: process.env.ADMIN_TABLE_NAME,
                Key: { email: { S: email } }
              }));

              // If an item is found, upgrade them to Admins
              if (ddbResponse.Item) {
                targetGroup = "Admins";
              }

              // Execute the assignment
              await cognitoClient.send(new AdminAddUserToGroupCommand({
                GroupName: targetGroup,
                UserPoolId: userPoolId,
                Username: username,
              }));
              
              console.log(`Assigned ${email} to ${targetGroup}`);

            } catch (err) {
              console.error("Error assigning group:", err);
              // Fail open or closed depending on requirements. 
              // Sending event back usually allows login to proceed without group assignment.
            }
            
            return event;
          };

  # 4. Attach the Lambda to the User Pool "Post Confirmation" Trigger
  MyUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: "MyUniversityPool"
      LambdaConfig:
        PostConfirmation: !GetAtt AssignGroupFunction.Arn

  # 5. Permission for Cognito to invoke the Lambda
  CognitoLambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref AssignGroupFunction
      Principal: cognito-idp.amazonaws.com
      SourceArn: !GetAtt MyUserPool.Arn
```


3. **Frontend Check:** When a user logs in, Cognito includes a `cognito:groups` list in their ID Token. The frontend checks this list to grant permissions.

#### Example: React Logic (using AWS Amplify v6)
```javascript
import { useState, useEffect } from 'react';
import { fetchAuthSession } from 'aws-amplify/auth';

const Dashboard = () => {
  const [isAdmin, setIsAdmin] = useState(false);

  useEffect(() => {
    const checkUserRole = async () => {
      try {
        const session = await fetchAuthSession();
        // The groups are usually array of strings found in the ID Token payload
        const groups = session.tokens?.idToken?.payload['cognito:groups'] || [];
        
        if (groups.includes('Admins')) {
          setIsAdmin(true);
        }
      } catch (err) {
        console.error("Error checking auth session:", err);
      }
    };

    checkUserRole();
  }, []);

  return (
    <div>
      <h1>Welcome to the Application</h1>
      {isAdmin && <button>Delete Database (Admins Only)</button>}
    </div>
  );
};
```

**Pros:** simpler architecture, single login entry point for everyone.
**Cons:** requires frontend logic to parse tokens.

### Option 2: Two Different Application Stacks

Create two completely separate User Pools: one for General Population and one for Special Admins.

**How it works:**

1. **Stack A (Public/General):** Configured for lower security friction (e.g., allows straight-forward SSO Logging, etc).

2. **Stack B (Admin):** Configured for high security (e.g., invites only, MFA required).

3. **Frontend:** The app usually has separate login flows or buttons ("Staff Login" vs "Student Login").

**Pros:** Complete isolation. You can enforce strict security (like MFA) on admins without burdening regular users.
**Cons:** More infrastructure to manage; requires managing two sets of environment variables (Pool IDs) in your app.


