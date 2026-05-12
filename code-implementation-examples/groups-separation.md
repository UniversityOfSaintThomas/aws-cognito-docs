# Separation of Groups (Handling Admins vs. General Users)

When you need to separate different types of users (e.g., Staff vs. Students, or Admins vs. Public), we have two recommended approaches:

### Option 1: Single Stack with Cognito Groups (Recommended for most apps)

Use one User Pool for everyone, but sort users into **Cognito Groups**.

**Table of Contents: Separation of Groups (What to add)**
- **1a.** User Pool and "Post Confirmation" Trigger
- **1b.** Admins Group
- **1c.** General Users Group
  - **(Optional) 1d.** Make more groups as needeed
- **2a.** DynamoDB to store Admin emails
- **2b.** Lambda Permissions to Read/Update
- **2c.** The Lambda Function that does the sorting
- **2d.** Permission for Cognito to invoke the Lambda
- **3a.** Enables a frontend to authenticate
- **3b.** Set Cognito Branding Version to Hosted UI (Classic)

In `application-infrastructure/template.yml` , under `Resources:` add the following code:
```yaml
Resources:
Resources:
	# ---------------------------------------------------------------------------
	# Separation of Groups
	
	# 1a. Create the User Pool (standard)
	CognitoUsers:
	  Type: AWS::Cognito::UserPool
	  DeletionPolicy: Retain
	  UpdateReplacePolicy: Retain
	  Properties: 
	    UserPoolName: !Sub "${Prefix}-${ProjectId}-${StageId}-UserPool"
	    LambdaConfig:
	      PostConfirmation: !GetAtt AssignGroupFunction.Arn
	
	# 1b. Define the 'Admins' Group
	AdminGroup:
	 Type: AWS::Cognito::UserPoolGroup
	 Properties:
		 GroupName: "Admins"
		 UserPoolId: !Ref CognitoUsers
		 Description: "University Staff and IT Admins"
		 Precedence: 0 # Higher precedence overrides lower ones if IAM roles conflict
	
	# 1c. Define the 'GeneralUsers' Group
	GeneralGroup:
	 Type: AWS::Cognito::UserPoolGroup
	 Properties:
		 GroupName: "GeneralUsers"
		 UserPoolId: !Ref CognitoUsers
		 Description: "Standard Student Access"

	# 1d. Define the more groups as needed (This Documentation will only use Admin and GeneralUsers)
	
	# 2a. DynamoDB Table storing the list of Admin emails
  AdminWhitelistTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub "${Prefix}-${ProjectId}-${StageId}-AdminWhitelist"
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: "email"
          AttributeType: "S"
      KeySchema:
        - AttributeName: "email"
          KeyType: "HASH"
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      TimeToLiveSpecification:
        AttributeName: "ttl"
        Enabled: true

  # 2b. Permissions for Lambda to Read DB and Update Cognito
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
	    RoleName: !Sub "${Prefix}-${ProjectId}-${StageId}-LambdaExecRole"
	    Description: "IAM Role that allows the Lambda permission to execute and access resources"
      Path: !Ref RolePath
      PermissionsBoundary: !If [HasPermissionsBoundaryArn, !Ref PermissionsBoundaryArn, !Ref 'AWS::NoValue' ]
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: 
	            Service: [ 'lambda.amazonaws.com' ]
            Action: sts:AssumeRole
      # These are the resources your Lambda function needs access to Logs, SSM Parameters, DynamoDb, S3, etc.
      # Define specific actions such as get/put (read/write). Work towards practicing the Principle of Least Privilege
      Policies:
			- PolicyName: !Sub "${Prefix}-${ProjectId}-${StageId}-ExecutionPolicy"
				PolicyDocument:
			    Statement:
			    
	          - Sid: LambdaAccessToWriteLogs
		            Action:
		            - logs:CreateLogGroup
		            - logs:CreateLogStream
		            - logs:PutLogEvents
		            Effect: Allow
		            Resource: !GetAtt AppLogGroup.Arn
            
            - Sid: DynamoDBReadAdminWhitelist
	            Effect: Allow
	            Action: ['dynamodb:GetItem']
	            Resource: !GetAtt AdminWhitelistTable.Arn
            
            - Sid: CognitoAssignGroup
	            Effect: Allow
	            Action: ['cognito-idp:AdminAddUserToGroup']
	            Resource: !Sub 'arn:aws:cognito-idp:${AWS::Region}:${AWS::AccountId}:userpool/*'
	            # The * above matches all user pools in this account/region. The condition below
	            # restricts access to only the user pool tagged with this deployment's ApplicationDeploymentId,
	            # preventing this Lambda from acting on unrelated user pools.
	            Condition:
	              StringEquals:
	                "aws:ResourceTag/atlantis:ApplicationDeploymentId": !Sub "${Prefix}-${ProjectId}-${StageId}"
		
		# 2c. The Lambda Function that does the sorting
	  AssignGroupFunction:
	    Type: AWS::Lambda::Function
	    Properties:
	      FunctionName: !Sub '${Prefix}-${ProjectId}-${StageId}-AssignGroupFunction'
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
	            let targetGroup = "GeneralUsers";
	            try {
	              const ddbResponse = await ddbClient.send(new GetItemCommand({
	                TableName: process.env.ADMIN_TABLE_NAME,
	                Key: { email: { S: email } }
	              }));
	              if (ddbResponse.Item) targetGroup = "Admins";
	              await cognitoClient.send(new AdminAddUserToGroupCommand({
	                GroupName: targetGroup,
	                UserPoolId: userPoolId,
	                Username: username,
	              }));
	              console.log(`Assigned ${email} to ${targetGroup}`);
	            } catch (err) {
	              console.error("Error assigning group:", err);
	            }
	            return event;
	          };
	
		# 2d. Permission for Cognito to invoke AssignGroupFunction
	  CognitoLambdaInvokePermission:
	    Type: AWS::Lambda::Permission
	    Properties:
	      Action: lambda:InvokeFunction
	      FunctionName: !Ref AssignGroupFunction
	      Principal: cognito-idp.amazonaws.com
	      SourceArn: !GetAtt CognitoUsers.Arn
	   
		
	  # 3a. Cognito App Client: OAuth2 client for the frontend to authenticate users via the hosted login UI
	  #     Uses Authorization Code flow (no client secret) with openid/email/profile scopes
	  CognitoAppClient:
	    Type: AWS::Cognito::UserPoolClient
	    Properties:
	      ClientName: !Sub "${Prefix}-${ProjectId}-${StageId}-AppClient"
	      UserPoolId: !Ref CognitoUsers
	      GenerateSecret: false
	      AllowedOAuthFlowsUserPoolClient: true
	      AllowedOAuthFlows:
	        - code
	      AllowedOAuthScopes:
	        - openid
	        - email
	        - profile
	      SupportedIdentityProviders:
	        - COGNITO
	      CallbackURLs:
	        - http://localhost:3000/callback
	      LogoutURLs:
	        - http://localhost:3000/
	 
	# 3b. Cognito Hosted UI Domain (Classic):
	#     provides a managed login/logout UI at <Prefix>-<ProjectId>.auth.<region>.amazoncognito.com
	CognitoUserPoolDomain:
		Type: AWS::Cognito::UserPoolDomain
		Properties:
		  Domain: !Sub "${Prefix}-${ProjectId}"
		  UserPoolId: !Ref CognitoUsers
		  ManagedLoginVersion: 1
	        
	#Rest of the code down here vvv (dont copy this or anything below this point)
	# ---------------------------------------------------------------------------
	# API Gateway Resources
	
	# -- API Gateway --
	#WebApi:
	#	.........
```

Note:
- There is a limit to the number of groups that can exist in a user pool. Refer [AWS Quota Documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html).
- Refer to the official [Cognito User Groups](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html) documentation to see generalized information related to User Groups.
- To assign users to these groups. You can do this manually in the AWS Console, or **automatically** by checking an external "Admin List" (like a DynamoDB table) during sign-up.
- This architecture assigns the "Admins" group only if the user's email exists in a specific DynamoDB table. Everyone else gets "GeneralUsers".
  - prath's note: if I were making this lambda function, I'd have the logic in an actual .js file with a lambda handler. But this also seems like a valid approach i really don't know would have to confirm

## Set Up Frontend

```bash
cd application-infrastructure/src
npx create-react-app frontend
```

You will be prompted:
```bash
Need to install the following packages:
create-react-app@5.1.0
Ok to proceed? (y): y
```
Once the frontend is created, cd into it and install AWS Amplify, React Router Dom, and React Scripts

```bash
cd frontend
npm install aws-amplify@^6.16.3
npm install react-router-dom@^7.14.1
npm install react-scripts@5.0.1
```
If there are any vulnerabilities, do `npm audit fix`
### In the frontend Directory:

### Create a file called `src/components/amplifyconfiguration.js`

```jsx
const amplifyConfig = {
  // Auth: {
  //   Cognito: {
  //     userPoolId: 'YOUR_USER_POOL_ID',
  //     userPoolClientId: 'YOUR_APP_CLIENT_ID',
  //   }
  // }
  Auth: {
    Cognito: {
      userPoolId:
        process.env.REACT_APP_USER_POOL_ID || 'YOUR_USER_POOL_ID',
      userPoolClientId:
        process.env.REACT_APP_CLIENT_ID || 'YOUR_APP_CLIENT_ID',
      loginWith: {
        oauth: {
          domain: process.env.REACT_APP_COGNITO_DOMAIN || 'PREFIX-PROJECTID.auth.us-east-1.amazoncognito.com',
          scopes: ['openid', 'email', 'profile'],
          redirectSignIn: [process.env.NODE_ENV === 'production' ? 'https://your-cloudfront-url/callback' : 'http://localhost:3000/callback'],
          redirectSignOut: [process.env.NODE_ENV === 'production' ? 'https://your-cloudfront-url/' : 'http://localhost:3000/'],
          responseType: 'code',
        },
      },
    },
  },
};

export default amplifyConfig;

```

Replace `YOUR_USER_POOL_ID`, `YOUR_APP_CLIENT_ID`, `PREFIX`, and `PROJECTID`  with the proper information. You can find your **User Pool ID** and the **App Client ID** in your CloudFormation Resources under **CognitoUsers** and **CognitoAppClient** respectively.

### Create a file called `src/components/CallbackPage.jsx`

```jsx
// src/components/CallbackPage.jsx
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { getCurrentUser } from 'aws-amplify/auth';

const CallbackPage = () => {
  const navigate = useNavigate();

  useEffect(() => {
    const check = async () => {
      try {
        await getCurrentUser();
        navigate('/');
      } catch {
        // still processing, Amplify handles the code exchange automatically
      }
    };
    const timer = setTimeout(check, 1000);
    return () => clearTimeout(timer);
  }, [navigate]);

  return (
    <div style={{ padding: '40px', textAlign: 'center' }}>
      <h2>Logging you in...</h2>
      <p>Please wait while we verify your credentials.</p>
    </div>
  );
};

export default CallbackPage;

```

### Create a file called `src/components/Dashboard.jsx`

```jsx
import { useState, useEffect } from 'react';
import { fetchAuthSession } from 'aws-amplify/auth';

const Dashboard = () => {
  const [isAdmin, setIsAdmin] = useState(false);

  useEffect(() => {
    const checkUserRole = async () => {
      try {
        // const session = await fetchAuthSession();
        const session = await fetchAuthSession({ forceRefresh: true });
        // The groups are usually array of strings found in the ID Token payload
        const groups = session.tokens?.idToken?.payload['cognito:groups'] || [];
        console.log("groups",groups);
        
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

export default Dashboard;
```

### Create a file called `src/components/LoginPage.jsx`

```jsx
import { signInWithRedirect } from 'aws-amplify/auth';

const LoginPage = () => {
  const handleLogin = () => signInWithRedirect();

  return (
    <div style={{ display: 'flex', justifyContent: 'center', alignItems: 'center', minHeight: '100vh', backgroundColor: '#f5f5f5' }}>
      <div style={{ backgroundColor: 'white', padding: '40px', borderRadius: '8px', boxShadow: '0 2px 10px rgba(0,0,0,0.1)', width: '100%', maxWidth: '400px' }}>
        <h1 style={{ textAlign: 'center', marginBottom: '30px' }}>My App</h1>
        <button onClick={handleLogin} style={{ width: '100%', padding: '12px', fontSize: '16px', backgroundColor: '#007bff', color: 'white', border: 'none', borderRadius: '4px', cursor: 'pointer' }}>
          Login / Sign Up
        </button>
      </div>
    </div>
  );
};

export default LoginPage;

```

### Modify frontend/src/index.js

```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { Amplify } from 'aws-amplify';
import amplifyConfig from './components/amplifyconfiguration';
import './index.css';
import App from './App';
import reportWebVitals from './reportWebVitals';

Amplify.configure(amplifyConfig);

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  // <React.StrictMode>
  <BrowserRouter>
    <App />
  </BrowserRouter>
  // </React.StrictMode>
);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();

```

### Modify frontend/src/App.js

```jsx
// src/App.js
import { useState, useEffect } from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import { getCurrentUser, signOut } from 'aws-amplify/auth';
import Dashboard from './components/Dashboard';
import LoginPage from './components/LoginPage';
import CallbackPage from './components/CallbackPage';
import './App.css';

function App() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getCurrentUser()
      .then(setUser)
      .catch(() => setUser(null))
      .finally(() => setLoading(false));
  }, []);

  const handleLogout = async () => {
    await signOut();
    setUser(null);
  };

  if (loading) return <p>Loading...</p>;

  return (
    <Routes>
      <Route path="/callback" element={<CallbackPage />} />
      <Route path="/login" element={user ? <Navigate to="/" /> : <LoginPage />} />
      <Route path="/" element={
        user ? (
          <div>
            <button onClick={handleLogout}>Logout</button>
            <Dashboard />
          </div>
        ) : (
          <Navigate to="/login" />
        )
      } />
    </Routes>
  );
}

export default App;

```

Push the changes and merge them to the `test` branch

## **Frontend Check:**

### Run local frontend

To test out cognito go into the frontend to test

```bash
cd application-infrastructure/src/frontend
npm run start
```
When a user logs in, Cognito includes a `cognito:groups` list in their ID Token. The frontend checks this list to grant permissions.

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
