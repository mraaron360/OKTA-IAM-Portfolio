# SAML settings reference

Quick-copy values for the lab. Replace `<placeholders>` with your own org values.

## Okta app integration

| Setting | Value |
|---|---|
| App name | `Lab - SAML SSO` |
| Sign-in method | SAML 2.0 |
| Single sign-on URL | `https://sptest.iamshowcase.com/acs` |
| Use for Recipient and Destination URL | Yes |
| Audience URI (SP Entity ID) | `IAMShowcase` |
| Default RelayState | *(empty)* |
| Name ID format | `EmailAddress` |
| Application username | `Email` |
| Update application username on | `Create and update` |
| Response | Signed |
| Assertion signature | Signed |
| Signature algorithm | `RSA_SHA256` |
| Digest algorithm | `SHA256` |
| Assertion encryption | Unencrypted |
| Single logout | Disabled |
| Signed requests | Disabled |
| Authn context class | `PasswordProtectedTransport` |

## Attribute statements

| Name | Okta expression |
|---|---|
| `email` | `user.profile.email` |
| `firstName` | `user.profile.firstName` |
| `lastName` | `user.profile.lastName` |

## Service Provider (RSA SAML Test Service Provider)

| Item | Value |
|---|---|
| Entity ID | `IAMShowcase` |
| ACS URL | `https://sptest.iamshowcase.com/acs` |
| ACS binding | HTTP-POST |
| Supported Name ID formats | `unspecified`, `emailAddress` |
| Wants signed assertions | Yes |
| Metadata | <https://sptest.iamshowcase.com/testsp_metadata.xml> |
| Instructions | <https://sptest.iamshowcase.com/instructions> |

## Okta IdP values (yours will differ)

| Item | Pattern |
|---|---|
| IdP metadata URL | `https://<your-org>.okta.com/app/<app-key>/sso/saml/metadata` |
| IdP-initiated SSO URL | `https://<your-org>.okta.com/app/<app-name>/<app-key>/sso/saml` |
| Issuer / IdP Entity ID | `http://www.okta.com/<app-key>` |

`<app-key>` is the identifier starting with `exk` that Okta generates for the app. It appears in the metadata URL on the app's **Sign On** tab.

## Sample assertion (abridged, values anonymised)

```xml
<saml2:Assertion ID="id-..." IssueInstant="..." Version="2.0">
  <saml2:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity">http://www.okta.com/<app-key></saml2:Issuer>
  <saml2:Subject>
    <saml2:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">user@example.com</saml2:NameID>
    <saml2:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml2:SubjectConfirmationData Recipient="https://sptest.iamshowcase.com/acs" NotOnOrAfter="..."/>
    </saml2:SubjectConfirmation>
  </saml2:Subject>
  <saml2:Conditions NotBefore="..." NotOnOrAfter="...">
    <saml2:AudienceRestriction>
      <saml2:Audience>IAMShowcase</saml2:Audience>
    </saml2:AudienceRestriction>
  </saml2:Conditions>
  <saml2:AuthnStatement AuthnInstant="...">
    <saml2:AuthnContext>
      <saml2:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</saml2:AuthnContextClassRef>
    </saml2:AuthnContext>
  </saml2:AuthnStatement>
  <saml2:AttributeStatement>
    <saml2:Attribute Name="firstName"><saml2:AttributeValue>Jane</saml2:AttributeValue></saml2:Attribute>
    <saml2:Attribute Name="lastName"><saml2:AttributeValue>Doe</saml2:AttributeValue></saml2:Attribute>
    <saml2:Attribute Name="email"><saml2:AttributeValue>user@example.com</saml2:AttributeValue></saml2:Attribute>
  </saml2:AttributeStatement>
</saml2:Assertion>
```
