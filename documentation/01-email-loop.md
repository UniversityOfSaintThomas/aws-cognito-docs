### 1. Open-To-All Authentication (Self-Registration):
```yaml
Resources:
  PublicUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "${ProjectId}-${StageId}-public-userpool"
      
      # --- REQUIRED FOR SIGN-UP FLOW ---
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

      # (Optional but recommended for user experience)
      Policies:
        PasswordPolicy:
          MinimumLength: 8
          RequireLowercase: true
          RequireNumbers: true
          RequireUppercase: true
```

### 2. Invite-Only (Admin Control):
```yaml
Resources:
  InviteOnlyUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "${ProjectId}-${StageId}-invite-userpool"
      
      # --- REQUIRED FOR INVITE-ONLY MODE ---
      AdminCreateUserConfig:
        # This is the single line that prevents public sign-ups
        AllowAdminCreateUserOnly: true 
      # ------------------------------------

      # --- REQUIRED FOR EMAIL VERIFICATION/INVITE ---
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
      # Note: The "Sign Up" link will still show on the Hosted UI (limitation)
      # but clicking it will result in the "error encountered" page.
```

### 3. Restricted by Domain / Whitelist:
- this is just AI Copypasta because I personally have had no experience with this
- it's there if anyone might need it

```yaml
Resources:
  # --- 1. The Pre-Sign-Up Lambda Function ---
  # This function checks the email domain (e.g., allows only @stthomas.edu)
  DomainCheckFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs20.x # Or Python runtime
      CodeUri: s3://source-bucket/domain-check-lambda.zip # Your compiled code package
      MemorySize: 128
      Timeout: 5
      # Define environment variables used by your Lambda code
      Environment:
        Variables:
          ALLOWED_DOMAIN: "stthomas.edu" 

  # --- 2. IAM Role for the Lambda ---
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
        - PolicyName: LogWriterPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: [ 'logs:CreateLogGroup', 'logs:CreateLogStream', 'logs:PutLogEvents' ]
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/*:log-stream:*'

  # --- 3. The User Pool with the Trigger Attached ---
  RestrictedUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "${ProjectId}-${StageId}-restricted-userpool"
      
      # --- KEY DIFFERENCE: Attaching the Lambda ---
      LambdaConfig:
        PreSignUp: !GetAtt DomainCheckFunction.Arn # Attaches the custom logic
      
      # --- REQUIRED FOR SIGN-UP FLOW ---
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
      # ... (rest of the basic config like Policies) ...
```