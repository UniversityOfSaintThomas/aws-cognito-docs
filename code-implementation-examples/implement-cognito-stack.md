# Step-By-Step: Creating a Shared Cognito Stack

Follow this guide to set up a production-ready Federated Authentication stack. This process involves an initial specific "prototyping" phase to allow developers to start working immediately while waiting on external security tickets.

### 1. Repository & Pipeline Setup
Our organization uses a standardized serverless structure managed by Atlantis.

1.  **Clone the Configuration Repository:**
    *   Retrieve the `samconfig` repository from the **AWS CodeCommit** service in the **DevSandbox** account.
    *   (Refer to the internal documentation: *63Klabs atlantis cfn configuration repo for serverless deployments*).

2.  **Initialize the Project:**
    *   Run the creation script to generate your project scaffold.
    *   Command: `./cli/create_repo.py your-webapp --profile yourprofile`
    *   **Important:** When prompted for starter code, select `basic-apigw-lambda-py`. This template is minimal and provides the cleanest starting point for custom Cognito resources.

3.  **Configure CI/CD Pipeline:**
    *   Set up the pipeline so changes deploy automatically.
    *   Command: `./cli/config.py pipeline acme your-webservice test --profile yourprofile`

---

### 2. Initial Infrastructure Deployment (Prototyping Phase)
**Goal:** Deploy a functional User Pool *immediately* so developers can build the frontend, without waiting for the Microsoft Security Team.

**Configuration:**
*   **EnableADFS:** Set to `false`.
*   **User Provisioning:** Configured as "Invite Only" (Admin Control). This allows you to manually create test users (e.g., `dev@stthomas.edu`) and send them invites.

**Add the following CloudFormation to your `template.yml`:**

```yaml
Parameters:
  EnableADFS:
    Type: String
    Default: "false"
    AllowedValues: ["true", "false"]
    Description: "Set to 'true' to enable the Entra/SAML identity provider."

  ADFSMetadataURL:
    Type: String
    Description: "The Metadata URL provided by the Entra ID team (Leave empty for initial deploy)."
    Default: ""

Conditions:
  CreateADFSProvider: !Equals [!Ref EnableADFS, "true"]

Resources:
  # 1. Shared User Pool
  SharedUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "${ProjectId}-${StageId}-userpool"
      
      # --- ADMIN ONLY / INVITE ONLY MODE ---
      # Prevents public sign-ups. Users must be created by Admin or come via Federation (later).
      AdminCreateUserConfig:
        AllowAdminCreateUserOnly: true 
      
      UsernameAttributes:
        - "email"
      Schema:
        - Name: email
          AttributeDataType: String
          Mutable: true
          Required: true
      AutoVerifiedAttributes:
        - email
      EmailConfiguration:
        EmailSendingAccount: COGNITO_DEFAULT

  # 2. User Pool Domain (Required for SSO later)
  SharedUserPoolDomain:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      UserPoolId: !Ref SharedUserPool
      Domain: !Sub "${StageId}-${ProjectId}-${AWS::AccountId}"

  # 3. Client App (for your website)
  UserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: !Sub "${ProjectId}-client"
      UserPoolId: !Ref SharedUserPool
      GenerateSecret: false
      # Allowed Flows
      AllowedOAuthFlows:
        - code
        - implicit
      AllowedOAuthScopes:
        - email
        - openid
        - profile
      CallbackURLs:
        - "http://localhost:3000" # Update with real URLs
      SupportedIdentityProviders:
        # conditionally add EntraID later, start with COGNITO
        - COGNITO 

  # 4. ADFS Provider (CONDITIONAL - Created in Step 6)
  ADFSProvider:
    Type: AWS::Cognito::UserPoolIdentityProvider
    Condition: CreateADFSProvider
    Properties:
      UserPoolId: !Ref SharedUserPool
      ProviderName: "EntraID"
      ProviderType: "SAML"
      ProviderDetails:
        MetadataURL: !Ref ADFSMetadataURL
      AttributeMapping:
        email: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
        name: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"

Outputs:
  UserPoolId:
    Value: !Ref SharedUserPool
  UserPoolDomain:
    Value: !Ref SharedUserPoolDomain
```

**Action:** Commit and push this code. The pipeline will deploy the stack.

---

### 3. Gather Cognito Outputs
Once the pipeline finishes, go to the AWS Cloud Console:
1.  Navigate to **CloudFormation** > Select your Stack.
2.  Go to the **Outputs** tab.
3.  Note down the following values (construct the Reply URL manually using the Domain):
    *   **Entity ID (Urn):** `urn:amazon:cognito:sp:{UserPoolId}`
    *   **Reply URL / ACS URL:** `https://{UserPoolDomain}.auth.{region}.amazoncognito.com/saml2/idpresponse`

---

### 4. Open Support Ticket
Send the **Entity ID** and **Reply URL** to your manager or the Security Team. ask them to generate the ADFS Metadata.

*Status: Blocked (Waiting for Security Team)*
*(Developers can continue working using the "Invite Only" flow detailed in Step 2)*

---

### 5. Receive SSO Configuration
The Security Team will provide:
1.  **Metadata URL:** An XML link (e.g., `https://login.microsoftonline.com/.../federationmetadata.xml`).
2.  **Attribute Mapping:** Confirmation of claim names (usually standard).

---

### 6. Finalize Production Deployment (Enable SSO)
Update your `template.yml` parameters to switch on the Federation.

**Changes:**
1.  Set `EnableADFS` to `true`.
2.  Paste the provided URL into `ADFSMetadataURL`.
3.  Update `UserPoolClient` -> `SupportedIdentityProviders` to include `EntraID` (if not done dynamically).

```yaml
Parameters:
  EnableADFS:
    Type: String
    Default: "true" # <--- CHANGED
  
  ADFSMetadataURL:
    Type: String
    Default: "https://login.microsoftonline.com/example/federationmetadata.xml" # <--- CHANGED
```

**Action:** Push changes.
**Result:** The pipeline updates the stack. Your login screen will now show the "Sign in with EntraID" button alongside the username/password fields.
