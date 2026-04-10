# Frontend Implementation Guide

This document explains how to integrate your web application (both infrastructure and frontend code) with the Shared Cognito Auth Stack.

## 1. Updating your Application's Infrastructure (CloudFormation)

To use the Shared Auth Stack, your individual web app's cloudformation template must define its own `AWS::Cognito::UserPoolClient`. This client acts as the bridge between your specific application and the central shared user pool.

### Required Template Modifications

You will need to pass the name of your Shared Auth Stack as a parameter and use `Fn::ImportValue` to reference the exported `UserPoolId` from the shared stack that you created in the previous step.

**Example Application Template Snippet (If using Entra ID with OIDC):**

```yaml
Parameters:
  SharedAuthStackName:
    Type: String
    Default: "acme-shared-auth-prod"
    Description: "The name of the deployed shared Cognito stack to import values from."

Resources:
  # ... other resources like API Gateway, Lambda ...

  # The User Pool Client your React app will use to interact with Cognito.
  CognitoAppClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: !Sub "${Prefix}-${ProjectId}-${StageId}-CognitoClient"
      UserPoolId:
        Fn::ImportValue: !Sub "${SharedAuthStackName}-UserPoolId"
      GenerateSecret: false # IMPORTANT: Must be false for Single Page Applications (SPAs) like React
      SupportedIdentityProviders:
        - COGNITO # Standard Cognito email/password
        - Fn::ImportValue: !Sub "${SharedAuthStackName}-OIDCProviderName" # Import Entra ID Provider for OIDC

      # --- OAuth 2.0 Configuration ---
      AllowedOAuthFlowsUserPoolClient: true
      AllowedOAuthFlows:
        - code # Required for PKCE flow
      AllowedOAuthScopes:
        - openid # Required for OIDC, returns the ID token.
        - email # Allows access to the user's email claim.
        - profile # Allows access to profile claims.

      # --- IMPORTANT: These URIs must match your React app's URLs ---
      CallbackURLs:
        - http://localhost:3000/callback # Local development
        - https://d2bw0ydew9gatv.cloudfront.net/callback # Production URL
      LogoutURLs:
        - http://localhost:3000/ # Redirect after logout
        - https://d2bw0ydew9gatv.cloudfront.net/ # Production URL

Outputs:
  AppClientId:
    Description: "The ID of the Cognito App Client."
    Value: !Ref CognitoAppClient
```

**Example Application Template Snippet (If using Entra ID with SAML):**

```yaml
Parameters:
  SharedAuthStackName:
    Type: String
    Default: "acme-shared-auth-prod"
    Description: "The name of the deployed shared Cognito stack to import values from."

Resources:
  # ... other resources like API Gateway, Lambda ...

  # The User Pool Client your React app will use to interact with Cognito.
  CognitoAppClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      ClientName: !Sub "${Prefix}-${ProjectId}-${StageId}-CognitoClient"
      UserPoolId:
        Fn::ImportValue: !Sub "${SharedAuthStackName}-UserPoolId"
      GenerateSecret: false # IMPORTANT: Must be false for Single Page Applications (SPAs) like React
      SupportedIdentityProviders:
        - COGNITO # Standard Cognito email/password
        - Fn::ImportValue: !Sub "${SharedAuthStackName}-SAMLProviderName" # Import Entra ID Provider for SAML

      # --- OAuth 2.0 Configuration ---
      AllowedOAuthFlowsUserPoolClient: true
      AllowedOAuthFlows:
        - code # Required for PKCE flow
      AllowedOAuthScopes:
        - openid # Required for OIDC, returns the ID token.
        - email # Allows access to the user's email claim.
        - profile # Allows access to profile claims.

      # --- IMPORTANT: These URIs must match your React app's URLs ---
      CallbackURLs:
        - http://localhost:3000/callback # Local development
        - https://d2bw0ydew9gatv.cloudfront.net/callback # Production URL
      LogoutURLs:
        - http://localhost:3000/ # Redirect after logout
        - https://d2bw0ydew9gatv.cloudfront.net/ # Production URL

Outputs:
  AppClientId:
    Description: "The ID of the Cognito App Client."
    Value: !Ref CognitoAppClient
```

## 2. Setting Up Frontend Authentication (React)

For a React single page authentication, we use the **OAuth 2.0 Authorization Code Flow with PKCE (Proof Key for Code Exchange)**.

### How PKCE Works Behind the Scenes (Don't need to bother with it, it's there if you're curious) :

1. **User Clicks Login:** Your app generates a random string (`code_verifier`) and a hashed version of it (`code_challenge`).
2. **Redirect to Cognito Hosted UI:** The user is redirected to the Cognito-hosted login page, sending the `code_challenge` in the URL.
3. **User Authenticates:** The user logs in (via email/password or Entra ID). Cognito redirects them back to your app's `redirectUri` with a short-lived Authorization Code.
4. **Token Exchange:** Your frontend app takes this Auth Code and sends it back to Cognito along with the original unhashed `code_verifier`.
5. **Validation:** Cognito verifies that the `code_verifier` hashes into the original `code_challenge` (proving that the app requesting the tokens is the same app that initiated the login in Step 1). If successful, Cognito returns the Access Token and ID Token.
6. **Authenticated Session:** Your app uses the Access Token to access protected API Gateway endpoints.

We recommend using the [`react-oauth2-code-pkce`](https://www.npmjs.com/package/react-oauth2-code-pkce) library, which handles all this complex hashing, token exchange, and state management automatically.

### Frontend Implementation Examples

**Step 1: Install the dependency**

```bash
npm install react-oauth2-code-pkce
```

**Step 2: Cognito Configuration (cognito-config.js)**
Create a config file linking your cloud infrastructure outputs to the frontend.

```javascript
/**
 * Cognito Configuration
 *
 * These values come from your CloudFormation stack outputs.
 * - userPoolId: From Shared Stack output "UserPoolId"
 * - clientId: From Application Stack output "AppClientId"
 * - domain: From Shared Stack output "CognitoDomain"
 */

const cognitoConfig = {
  userPoolId: process.env.REACT_APP_COGNITO_USER_POOL_ID || "us-east-1_w0rGU3APG",
  clientId: process.env.REACT_APP_COGNITO_CLIENT_ID || "2p7m68guubvlt2v3ik6882vf9t",
  domain: process.env.REACT_APP_COGNITO_DOMAIN || "test-shared-730335558011",
  region: process.env.REACT_APP_AWS_REGION || "us-east-1",

  // Where Cognito sends users after successful login
  redirectUri: process.env.NODE_ENV === "production"
    ? "https://d2bw0ydew9gatv.cloudfront.net/callback"
    : "http://localhost:3000/callback",

  // Where Cognito sends users after logging out
  logoutRedirectUri: process.env.NODE_ENV === "production"
    ? "https://d2bw0ydew9gatv.cloudfront.net/"
    : "http://localhost:3000/",

  // Endpoint to clear the Cognito session out of the browser
  logoutEndpoint: () => {
    return `https://${cognitoConfig.domain}.auth.${cognitoConfig.region}.amazoncognito.com/logout?client_id=${cognitoConfig.clientId}&logout_uri=${encodeURIComponent(cognitoConfig.logoutRedirectUri)}`;
  },

  // Endpoint to direct users to the sign-up page directly (Hosted UI)
  signupEndpoint: () => {
    return `https://${cognitoConfig.domain}.auth.${cognitoConfig.region}.amazoncognito.com/signup?client_id=${cognitoConfig.clientId}&redirect_uri=${encodeURIComponent(cognitoConfig.redirectUri)}&response_type=code&scope=openid+email+profile`;
  },
};

export default cognitoConfig;
```

**Step 3: Auth Provider Context (AuthProvider.js)**

```javascript
import { AuthProvider, AuthContext } from "react-oauth2-code-pkce";
import React from "react";

// Re-export for convenience
export { AuthProvider, AuthContext };

// Custom hook to use auth context easily across your application components
export const useAuthContext = () => {
  return React.useContext(AuthContext);
};
```

*(Note: Wrap your application entry point in index.js or App.js with the imported \<AuthProvider> configuring it with standard PKCE inputs pulled from cognitoConfig.)*

**Step 4: Using Auth State & Logout in your App (App.js)**

```javascript
import React, { useState } from "react";
import { useAuthContext } from "./AuthProvider";
import cognitoConfig from "./cognito-config";

function App() {
  const { token, logOut, idToken } = useAuthContext();

  // Extract user email from the ID token provided by Cognito
  const getUserEmail = () => {
    if (!idToken) return null;
    try {
      // Decode the JWT Payload
      const payload = idToken.split(".")[1];
      const decoded = JSON.parse(
        atob(payload.replace(/-/g, "+").replace(/_/g, "/"))
      );
      return decoded.email;
    } catch (err) {
      console.error("Error decoding token:", err);
      return null;
    }
  };

  const userEmail = getUserEmail();

  // Proper logout requires clearing local tokens AND the remote Cognito session
  const handleLogout = () => {
    // 1. Clear local tokens managed by react-oauth2-code-pkce
    logOut();
    // 2. Redirect to Cognito logout endpoint to clear the remote session cookies
    window.location.href = cognitoConfig.logoutEndpoint();
  };

  return (
    <div className="App">
      <header className="App-header">
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center" }}>
          <div>
            <h1>Canvas Catalog Enrollment Tool</h1>
          </div>
          {token && (
            <div style={{ textAlign: "right", marginRight: "20px" }}>
              <p style={{ margin: "0 0 10px 0" }}>
                Welcome, {userEmail || "User"}
              </p>
              <button
                onClick={handleLogout}
                style={{
                  padding: "8px 16px",
                  backgroundColor: "#dc3545",
                  color: "white",
                  border: "none",
                  borderRadius: "4px",
                  cursor: "pointer",
                }}
              >
                Logout
              </button>
            </div>
          )}
        </div>
      </header>
      
      <main>
        {/* Protected App Logic goes here */}
        {token ? <p>You have access to the app!</p> : <p>Please Log in to continue.</p>}
      </main>
    </div>
  );
}

export default App;
```
