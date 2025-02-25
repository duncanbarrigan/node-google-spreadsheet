# Authentication

This module relies heavily on the [google-auth-library](https://github.com/googleapis/google-auth-library-nodejs) module to handle most authentication needs.

> _For those already familiar..._<br/>
> Follow [their instructions](https://github.com/googleapis/google-auth-library-nodejs#ways-to-authenticate)
> and pass a `JWT` | `OAuth2Client` | `GoogleAuth` object as the second arg when initializing your `GoogleSpreadsheet`.

Now as with the rest of Google's docs, it's a bit hard to just jump in and get started quickly, so hopefully this guide will get you going for most common scenarios...

You have several options for how you want to connect, but most projects connecting from a server/backend should use a service account.

- [Service Account (via JWT)](#service-account) - connects as a specific "bot" user generated by google for your application
- [OAuth 2](#oauth) - connect on behalf of a specific user using OAuth
- [Application Default Credentials](#adc) - auto-detect credentials, useful if running your app in google cloud
- [API-key](#api-key) - only identifies your application for metering, provides read-only access to public docs
- [Other (self-managed token)](#token) - set a token that you are managing some other way (not recommended)

Some of google-auth-library's more exotic auth methods are likely supported as well, as long as you pass in a GoogleAuth object that has a `getRequestHeaders()` method.

## Setting up your "Application"

Regardless of your needs and the auth method you use, before you can do anything, you'll need to set up a new "Google Cloud Project"

**👉 BUT FIRST -- Set up your google project & enable the sheets API 👈**
1. Go to the [Google Developers Console](https://console.developers.google.com/)
2. Select your project or create a new one (and then select it)
3. Enable the Sheets API for your project
  - In the sidebar on the left, select **Enabled APIs & Services**
  - Click the blue "Enable APIs and Services" button in the top bar
  - Search for "sheets"
  - Click on "Google Sheets API"
  - click the blue "Enable" button
4. (Optional) Enable the "Google Drive API" for your project - if you want to manage document permissions
  - same as above, but search for "drive" and enable the "Google Drive API"


## Auth Scopes

Aside from enabling the correct API(s) for your _project_, your specific auth method must be initialized with the correct scope(s).

(API key access does not need any scopes selected - as it is read only)

In most cases, I'd suggest using the following for your list of scopes:
`['https://www.googleapis.com/auth/spreadsheets', 'https://www.googleapis.com/auth/drive.file']`

But if you need more control, this is the full list of relevant scopes you might want to use:
- `https://www.googleapis.com/auth/spreadsheets` - read/write access to all Sheets docs
- `https://www.googleapis.com/auth/spreadsheets.readonly` - read-only access to all Sheets docs
- `https://www.googleapis.com/auth/drive` - read/write access to all Google Drive files
- `https://www.googleapis.com/auth/drive.readonly` - read-only access to all Google Drive files
- `https://www.googleapis.com/auth/drive.file` - read/write access to only the specific Google Drive files used with this "app"



## Authentication Methods

### 🤖 Service Account (recommended) :id=service-account
**Connect as a bot user that belongs to your app**

This is a 2-legged oauth method and designed to be "an account that belongs to your application instead of to an individual end user".
Use this for an app that needs to access a set of documents that you have full access to, or can at least be shared with your service account.
([read more](https://developers.google.com/identity/protocols/OAuth2ServiceAccount))

You may also grant your service account ["domain-wide delegation"](https://developers.google.com/identity/protocols/oauth2/service-account#delegatingauthority) which enables it to impersonate any user within your org. This can be helpful if you need to connect on behalf of users _only within your organization_.

__Setup Instructions__

1. Follow steps above to set up project and enable sheets API
2. Create a service account for your project
  - In the sidebar on the left, select **APIs & Services > Credentials**
  - Click blue "+ CREATE CREDENTIALS" and select "Service account" option
  - Enter name, description, click "CREATE"
  - You can skip permissions, click "CONTINUE"
  - Click "+ CREATE KEY" button
  - Select the "JSON" key type option
  - Click "Create" button
  - your JSON key file is generated and downloaded to your machine (__it is the only copy!__)
  - click "DONE"
  - note your service account's email address (also available in the JSON key file)
3. Share the doc (or docs) with your service account using the email noted above

!> Be careful - never check your API keys / secrets into version control (git)

You can now use this file in your project to authenticate as your service account. If you have a config setup involving environment variables, you only need to worry about the `client_email` and `private_key` from the JSON file. For example:

```javascript
import { JWT } from 'google-auth-library'

import creds from './config/myapp-1dd646d7c2af.json'; // the file saved above

const SCOPES = [
  'https://www.googleapis.com/auth/spreadsheets',
  'https://www.googleapis.com/auth/drive.file',
];

const jwt = new JWT({
  email: creds.client_email,
  key: creds.private_key,
  scopes: SCOPES,
});
const doc = new GoogleSpreadsheet('<YOUR-DOC-ID>', jwt);
```

Or preferably load the info from env vars, for example:
```javascript
const jwtFromEnv = new JWT({
  email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
  key: process.env.GOOGLE_PRIVATE_KEY,
  scopes: SCOPES,
});
```

And here is an example using impersonation - NOTE: your service account must have ["domain-wide delegation" enabled](https://developers.google.com/identity/protocols/oauth2/service-account#delegatingauthority).

```javascript
const jwtWithImpersonation = new JWT({
  email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
  key: process.env.GOOGLE_PRIVATE_KEY,
  subject: 'user.to.impersonate@mycompany.com',
  scopes: SCOPES,
});
```

**SPECIAL NOTE FOR HEROKU USERS (AND SIMILAR PLATFORMS)**

Depending on the platform you use, sometimes setting/retreiving env vars that include line breaks can get tricky as they get encoded weirdly while saving these values in their system.

Here's one way to get it to work (using heroku cli):
1. Save the private key to a new text file
2. Replace `\n` with actual line breaks
3. Replace `\u003d` with `=`
4. run `heroku config:add GOOGLE_PRIVATE_KEY="$(cat yourfile.txt)"` in your terminal

And in some other scenarios, replacing newlines in your code can be helpful, for example:
```js
await doc.useServiceAccountAuth({
  client_email: process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL,
  private_key: process.env.GOOGLE_PRIVATE_KEY.replace(/\\n/g, "\n"),
});
```

### 👨‍💻 OAuth 2.0 :id=oauth
**connect on behalf of a user with an Oauth token**

Use [OAuth2Client](https://github.com/googleapis/google-auth-library-nodejs#oauth2) to authenticate.

Handling Oauth and how it works is out of scope of this project - but the info you need can be found [here](https://developers.google.com/identity/protocols/oauth2).

Nevertheless, here is a rough outline of what to do with a few tips:
1. Follow steps above to set up project and enable sheets API
2. Create OAuth 2.0 credentials for your project (**these are not the same as the service account credentials described above**)
  - Navigate to the [credentials section of the google developer console](https://console.cloud.google.com/apis/credentials)
  - Click blue "+ CREATE CREDENTIALS" and select "Oauth Client ID" option
  - Select your application type and set up authorized domains / callback URIs
  - Record your client ID and secret
  - You will need to go through an Oauth Consent screen verification process to use these credentials for a production app with many users
3. For each user you want to connect on behalf of, you must get them to authorize your app which involves asking their permissions by redirecting them to a google-hosted URL
  - generate the oauth consent page url and redirect the user to it
    - there are many tools, [google provided](https://github.com/googleapis/google-api-nodejs-client#oauth2-client) and [more](https://www.npmjs.com/package/simple-oauth2) [generic](https://www.npmjs.com/package/hellojs) or you can even generate the URL yourself
    - make sure you use the credentials generated above
    - make sure you include the [appropriate scopes](https://developers.google.com/identity/protocols/oauth2/scopes#sheets) for your application
  - the callback URL (if successful) will include a short lived authorization code
  - you can then exchange this code for the user's oauth tokens which include:
    - an access token (that expires) which can be used to make API requests on behalf of the user, limited to the scopes requested and approved
    - a refresh token (that does not expire) which can be used to generate new access tokens
  - save these tokens somewhere secure like a database (ideally you should encrypt these before saving!)
4. Initialize an OAuth2Client with your apps oauth credentials and the user's tokens, and pass the client to your GoogleSpreadsheet object


```javascript
const { OAuth2Client } = require('google-auth-library');

// Initialize the OAuth2Client with your app's oauth credentials
const oauthClient = new OAuth2Client({
  clientId: process.env.GOOGLE_OAUTH_CLIENT_ID,
  clientSecret: process.env.GOOGLE_OAUTH_CLIENT_SECRET,
});

// Pre-configure the client with credentials you have stored in e.g. your database
// NOTE - `refresh_token` is required, `access_token` and `expiry_date` are optional
// (the refresh token is used to generate a missing/expired access token)
const { accessToken, refreshToken, expiryDate } = await fetchUserGoogleCredsFromDatabase();
oauthClient.credentials.access_token = accessToken;
oauthClient.credentials.refresh_token = refreshToken;
oauthClient.credentials.expiry_date = expiryDate; // Unix epoch milliseconds

// Listen in whenever a new access token is obtained, as you might want to store the new token in your database
// Note that the refresh_token never changes (unless it's revoked, in which case your end-user will
// need to go through the full authentication flow again), so storing the new access_token is optional
oauthClient.on('tokens', (credentials) => {
  console.log(credentials.access_token);
  console.log(credentials.scope);
  console.log(credentials.expiry_date);
  console.log(credentials.token_type); // will always be 'Bearer'
})

const doc = new GoogleSpreadsheet('<YOUR-DOC-ID>', oauthClient);
```



### 🪄 Application Default Credentials :id=adc
**Auto-detect credentials in google infrastructure**

Use [GoogleAuth](https://github.com/googleapis/google-auth-library-nodejs#application-default-credentials) to authenticate.


```js
const { GoogleAuth } = require('google-auth-library');

const adcAuth = new GoogleAuth({
  scopes: ['https://www.googleapis.com/auth/spreadsheets', 'https://www.googleapis.com/auth/drive.file'],
});

const doc = new GoogleSpreadsheet('<YOUR-DOC-ID>', adcAuth);
```


### 🔑 API Key :id=api-key
**read-only access of public docs**

Google requires this so they can at least meter your usage of their API.

!> Allowing only read-only access using an API key is a limitation of Google's API (see [issuetracker](https://issuetracker.google.com/issues/36755576#comment3))

__Setup Instructions__
1. Follow steps above to set up project and enable sheets API
2. Create an API key for your project
  - Navigate to the [credentials section of the google developer console](https://console.cloud.google.com/apis/credentials)
  - Click blue "+ CREATE CREDENTIALS" and select "API key" option
  - Copy the API key
3. OPTIONAL - click "Restrict key" on popup to set up restrictions
  - Click "API restrictions" > Restrict Key"
  - Check the "Google Sheets API" checkbox
  - Click "Save"

!> Be careful - never check your API keys / secrets into version control (git)

```javascript
const doc = new GoogleSpreadsheet('<YOUR-DOC-ID>', { apiKey: process.env.GOOGLE_API_KEY });
```

### 🗝️ Raw Token :id=token
**self managed token**

If for some reason you are not using google-auth-library and you are already managing auth tokens some other way, you can just pass that in directly.

```javascript
const doc = new GoogleSpreadsheet('<YOUR-DOC-ID>', { token: someTokenYouAreManaging });
```

