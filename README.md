# yes® Relying Party/Client Implementation in Python 3

This library implements a complete yes® relying party / client flow in Python.

> :warning: **Please read the [license](LICENSE) before using!** This is not an SDK for yes and not part of any contract you might have with yes.com.

Please refer to the [yes® Relying Party Developer Guide](https://www.yes.com/docs/rp-devguide/latest/index.html) for a description of the yes® flows.

## Implementation Status

 * ☒ [yes® Identity Flow](https://yes.com/docs/rp-devguide/latest/IDENTITY/index.html) 
 * ☒ [yes® Signing Flow](https://yes.com/docs/rp-devguide/latest/QES/index.html) for Qualified Electronic Signatures
 * ☒ [yes® Credit Transfer](https://yes.com/docs/pis/1.6/index.html)
 * ☐ future yes® Flows (AIS, ...)

## Examples

Please refer to [this repository](https://github.com/yescom/pyyes-examples) for example.

## How to Use (Identity Flow)

Please also refer to the example code in [`simple_identity.py`](examples/simple_identity.py).

**Step 1:** Acquire credentials to access the yes® ecosystem as described [here](https://www.yes.com/docs/rp-devguide/latest/ONBOARDING/index.html). Note that for testing, you can use the Sandbox Demo Client credentials published [here](https://www.yes.com/docs/rp-devguide/latest/ONBOARDING/index.html#_sandbox_demo_client).

**Step 2:** Put the `client_id`, the paths to the certificate and private key file, and other information into a `YesConfiguration` object.

```python
import yes 

yes_configuration = yes.YesConfiguration(
    environment=yes.YesEnvironment.SANDBOX, # or .PRODUCTION
    client_id="sandbox.yes.com:e85ff3bc-96f8-4ae7-b6b1-894d8dde9ebe",  # provided by yes®
    cert_file="cert.pem",  # see developer guide
    key_file="key.pem",  # see developer guide
    redirect_uri="http://localhost:3000/yes/oidccb",  # exactly as registered with yes®
)

```

**Step 3:** Decide which data you want to retrieve from the yes® IDP. See the [respective sections of the developer guide](https://yes.com/docs/rp-devguide/latest/IDENTITY/index.html#user_information) for details. If in doubt, request all data in the `id_token` and leave the `userinfo` dictionary empty. 

Note that this definition here is in native Python syntax and will be converted to JSON before being sent to the server. 

You also need to decide if you want the user to use their online banking second factor. See [here](https://yes.com/docs/rp-devguide/latest/IDENTITY/index.html#acr) for details.

```python
claims = {
    "id_token": {
        "verified_claims": {
            "claims": {"given_name": None,},
            "verification": {"trust_framework": None},
        }
    },
    "userinfo": {
        "verified_claims": {
            "claims": {"family_name": None,},
            "verification": {"trust_framework": None},
        }
    },
}
```
**Step 4:** Set up a starting point for the flow and two callback URIs, the account chooser callback URI and the OpenID Connect callback URI. Note that the two callback URIs have to be registered with yes for your client.

```python
import cherrypy
import yes

class YesExample:
    @cherrypy.expose
    def start(self):
        """
        Starting the yes® flow after the user clicked on the yes® button.
        """
        ...

    @cherrypy.expose
    def accb(self, state, issuer_url=None, error=None):
        """
        Account chooser callback. The user arrives here after selecting a bank.
        """
        ...

    @cherrypy.expose
    def oidccb(self, iss, code=None, error=None, error_description=None):
        """
        OpenID Connect callback endpoint. The user arrives here after going
        through the authentication/authorizaton steps at the bank.
        """
        ...
```

**Step 4:** Create a yes® session (`yes.YesIdentitySession`) and instantiate a yes® identity flow based on it (`yes.YesIdentityFlow`). You need to take care that this session information is persisted across all requests from the user - ideally, store it in the session mechanism provided by your web framework. *Do not store it in a user-accessible place (cookies, URL parameters, etc.)!*

```python
class YesExample:
    @cherrypy.expose
    def start(self):
        """
        Starting the yes® flow after the user clicked on the yes® button.
        """
        yessession = yes.YesIdentitySession(claims, request_second_factor=True)
        cherrypy.session["yes"] = yessession
        yesflow = yes.YesIdentityFlow(yes_configuration, cherrypy.session["yes"])
        ...

    @cherrypy.expose
    def accb(self, state, issuer_url=None, error=None):
        """
        Account chooser callback. The user arrives here after selecting a bank.
        """
        yesflow = yes.YesIdentityFlow(yes_configuration, cherrypy.session["yes"])
        ...

    @cherrypy.expose
    def oidccb(self, iss, code=None, error=None, error_description=None):
        """
        OpenID Connect callback endpoint. The user arrives here after going
        through the authentication/authorizaton steps at the bank.
        """
        yesflow = yes.YesIdentityFlow(yes_configuration, cherrypy.session["yes"])
        ...
```

**Step 5:** Finally, call the methods for starting the flow, handling the callback information, and retrieving the user data in the respective places in the flow. Make sure to handle error conditions properly, in particular `yes.YesAccountSelectionRequested` (see example).

```python

class YesExample:
    @cherrypy.expose
    def start(self):
        """
        Starting the yes® flow after the user clicked on the yes® button.
        """
        yessession = yes.YesIdentitySession(claims, request_second_factor=True)
        cherrypy.session["yes"] = yessession
        yesflow = yes.YesIdentityFlow(yes_configuration, cherrypy.session["yes"])
        ac_redirect = yesflow.start_yes_flow()

        cherrypy.log(f"Account chooser redirection to {ac_redirect}.")
        raise cherrypy.HTTPRedirect(ac_redirect)

    @cherrypy.expose
    def accb(self, state, issuer_url=None, error=None):
        """
        Account chooser callback. The user arrives here after selecting a bank.

        Note that the URL of this endpoint has to be registered with yes for
        your client. 
        """
        yesflow = yes.YesIdentityFlow(yes_configuration, cherrypy.session["yes"])

        try:
            authorization_endpoint_uri = yesflow.handle_ac_callback(
                state, issuer_url, error
            )
        except yes.YesUserCanceledError:
            cherrypy.HTTPRedirect("/")
        except yes.YesError as exception:
            # not implemented here: show nice error messages
            raise cherrypy.HTTPError(400, str(exception))

        raise cherrypy.HTTPRedirect(authorization_endpoint_uri)

    @cherrypy.expose
    def oidccb(self, iss, code=None, error=None, error_description=None):
        """
        OpenID Connect callback endpoint. The user arrives here after going
        through the authentication/authorizaton steps at the bank.

        Note that the URL of this endpoint has to be registered with yes for
        your client. 
        """
        yesflow = yes.YesIdentityFlow(yes_configuration, cherrypy.session["yes"])

        try:
            yesflow.handle_oidc_callback(iss, code, error, error_description)
        except yes.YesAccountSelectionRequested as exception:
            # user selected "select another bank" → must send user back to account chooser
            raise cherrypy.HTTPRedirect(exception.redirect_uri)
        except yes.YesError as exception:
            # not implemented here: show nice error messages
            raise cherrypy.HTTPError(400, str(exception))

        # id token and userinfo are alternative ways to retrieve user information - see developer guide
        user_data_id_token = yesflow.send_token_request()
        user_data_userinfo = yesflow.send_userinfo_request()

        return (
            "Got user data in the ID token: <pre>"
            + json.dumps(user_data_id_token, indent=4)
            + "</pre> ... and from the userinfo endpoint: <pre>"
            + json.dumps(user_data_userinfo, indent=4)
            + "</pre>"
        )

```
Example output:
```
Got user data in the ID token:
{
    "sub": "8b9f9588-4ad1-4069-9c66-7231f79d41c0",
    "aud": "sandbox.yes.com:e85ff3bc-96f8-4ae7-b6b1-894d8dde9ebe",
    "acr": "https://www.yes.com/acrs/online_banking_sca",
    "verified_claims": {
        "claims": {
            "given_name": "Peter"
        },
        "verification": {
            "trust_framework": "de_aml"
        }
    },
    "iss": "https://testidp.sandbox.yes.com/issuer/10000002",
    "exp": 1607523149,
    "iat": 1607522249,
    "nonce": "7l7Un_gVvYWfEUhfH8bfrgye_DV3rZTttdxam9QjoSg"
}
... and from the userinfo endpoint:
{
    "sub": "8b9f9588-4ad1-4069-9c66-7231f79d41c0",
    "verified_claims": {
        "claims": {
            "family_name": "Gottschalk"
        },
        "verification": {
            "trust_framework": "de_aml"
        }
    }
}
```

See the full example in [`simple_identity.py`](examples/simple_identity.py).

## How to Use (Signing Flow)

**Note:** Besides the same client data as described for the identity flow, for signing, you also need the identifier of the QTSP to create the signature with. Please contact yes® to receive the identifier. This identifier needs to be added to the configuration:

```python
...
yes_configuration = {
    ...
    "qtsp_id": "sp:yes.com:..."
    ...
}
...
```

Please see [`full_signing_flow.py`](examples/full_signing_flow.py) for a signing example.

This library can be used to sign the following types of documents:

 * Plain-text strings (`yes.TextSigningDocument`)
 * A default text defined by the QTSP if the QES is only used for identification purposes (QID feature) (`yes.DefaultSigningDocument`)
 * PDF documents (`PDFSigningDocument`)

Optionally, identity attributes can be embedded into the signature (QESID).
