### 1. CloudFormation Template for Entra ID Setup
```yaml
# =============================================================================
# PARAMETERS
# -----------------------------------------------------------------------------
Parameters:
  # --- ADFS/ENTRA ID PARAMETERS ---
  EnableADFS:
    Type: String
    Default: "true"
    AllowedValues: ["true", "false"]
    Description: "Set to 'true' to enable the Entra/SAML identity provider."

  ADFSProviderName:
    Type: String
    Description: "The name displayed for the SSO button (e.g., 'EntraID' or 'StThomasSSO')."
    Default: "EntraSSO"

  ADFSMetadataURL:
    Type: String
    Description: "The App Federation Metadata URL provided by the Entra ID team."
    Default: ""

# =============================================================================
# CONDITIONS
# -----------------------------------------------------------------------------
Conditions:
  # This condition ensures the Entra provider is only built if explicitly enabled.
  CreateADFSProvider: !Equals [!Ref EnableADFS, "true"]

# =============================================================================
# RESOURCES
# -----------------------------------------------------------------------------
Resources:
  # --- 1. The Shared User Pool (The core directory) ---
  SharedUserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UserPoolName: !Sub "${Prefix}-${ProjectId}-${StageId}-userpool"
            
      # Required schema attributes
      Schema:
        - Name: email
          AttributeDataType: String
          Mutable: true
          Required: true
            
      # Password policy are defined but only apply if using admin-created users
      Policies:
        PasswordPolicy:
          MinimumLength: 8
          RequireLowercase: true
          RequireNumbers: true
          RequireSymbols: false
          RequireUppercase: true
      
      # AutoVerifiedAttributes and EmailConfiguration are not strictly required 
      # for federated users, but kept for robustness (e.g., admin-created user invite flow).
      AutoVerifiedAttributes:
        - email
        
      EmailConfiguration:
        EmailSendingAccount: COGNITO_DEFAULT
      
      # IMPORTANT: Setting this to 'true' ensures the /signup page is disabled.
      # Users MUST come through a federated provider (like Entra ID).
      AdminCreateUserConfig:
        AllowAdminCreateUserOnly: true 
      
      # Allows password recovery for admin-created users
      AccountRecoverySetting:
        RecoveryMechanisms:
          - Priority: 1
            Name: verified_email

  # --- 2. The Entra ID Identity Provider (Conditional SSO Trust) ---
  ADFSProvider:
    Type: AWS::Cognito::UserPoolIdentityProvider
    Condition: CreateADFSProvider
    Properties:
      UserPoolId: !Ref SharedUserPool
      ProviderName: !Ref ADFSProviderName
      ProviderType: "SAML"
      
      # The Metadata URL handles the public key exchange and endpoint discovery
      ProviderDetails:
        MetadataURL: !Ref ADFSMetadataURL
        
      # Map the standard identity claims from the SAML response to Cognito attributes
      AttributeMapping:
        email: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
        name: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"

  # --- 3. The Shared Hosted UI Domain ---
  SharedUserPoolDomain:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      UserPoolId: !Ref SharedUserPool
      # Naming convention to ensure uniqueness across AWS accounts + Cognito is very strict about what you can name your domains
      Domain: !Sub "${StageId}-${Prefix}-${AWS::AccountId}"

# =============================================================================
# OUTPUTS
# -----------------------------------------------------------------------------
Outputs:
  UserPoolId:
    Description: "The ID of the shared Cognito User Pool (required by client apps)."
    Value: !Ref SharedUserPool
    Export:
      Name: !Sub "${AWS::StackName}-UserPoolId"

  UserPoolArn:
    Description: "The ARN of the shared Cognito User Pool (required by API Gateways)."
    Value: !GetAtt SharedUserPool.Arn
    Export:
      Name: !Sub "${AWS::StackName}-UserPoolArn"

  UserPoolDomain:
    Description: "The FQDN prefix for the Cognito Hosted UI."
    Value: !Ref SharedUserPoolDomain
    Export:
      Name: !Sub "${AWS::StackName}-UserPoolDomain"

  # Provides the name used to reference this provider in client apps
  ADFSProviderName:
    Description: "The name of the configured Entra SAML Provider (used by client apps for SSO button)."
    Condition: CreateADFSProvider
    Value: !Ref ADFSProviderName
    Export:
      Name: !Sub "${AWS::StackName}-ADFSProviderName"
```