# XOAUTH2 flow for email

This library provides a basic XOauth2 flow for email which handles

- logging in a web browser
- saving refresh token
- getting an access token from the refresh token

I use it with offlineimap and Emacs.

Unlike various other offerings across the web, this is designed to be a single
source of truth for email authentication. Implementing a new provider is as
simple as subclassing and handling getting the refresh token.

The refresh token is stored in a file, with permissions set to 600.  *It can be
read by any process running as your user (and by root)*.  Personally this
doesn't bother me: email isn't secure anyway.  You might want to subclass/edit
and store in something like a keyring.

Microsoft accounts, for some annoying reason, use a 'display' email (frequently
`first.last@company.suffix`) but require authentication with a 'machine' email
(`asdf67@company.suffix`). Or at any rate they do in Durham. This information is
stored on the Class if provided, so we can generate the xoauth2 str reliably. We
could just as easily do this in the client (indeed, I did), but the idea is to
be the sole source of truth, not just another step in an already overcomplicated
process.

For the client id/secret we pretend to be thunderbird.

## Installation

```bash
pip install -r requirements.txt
```
Put `email.py` somewhere sensible.  Then create yourself a `oauth.py` following
the model in `oauth_example.py`.

## Initial Setup
Make sure your `.pass` folder exists and has the right permissions. The scripts 
will write some information inside, in a file (such as `~/.pass/Microsoft`). You
don't need to create or fill such a file beforehand.

Run 

```
python oauth.py USER@ACCOUNT --refresh
```
for every account defined in `oauth.py`.  This
will fire up a web browser (run with `BROWSER=/path/to/browser` to change) and
direct you to your usual company login portal, after which it will extract the
refresh token and save it to disk. 

You can pass all the accounts in at once, in which case they will be processed
sequentially.  Warning!  If your browser keeps you logged in — which it probably
will — and you have multiple accounts from the same provider, this may not do
what you think.

Note about Office365: the `redirect_uri` may need to use port `8080` instead of `5671` (line 128 of `email_auth.py`).

## Offlineimap (or library usage)

In my `~/.offlineimaprc` I have:

```python
[Respository RemoteGmail]
pythonfile = ~/code/email/oauth.py
remoteuser = ...
auth_mechanisms = XOAUTH2
oauth2_access_token_eval = Gmail1.authentication_token()
```

and so on.

You may need to add `oauth2_request_url = https://login.microsoftonline.com/common/oauth2/v2.0/token`, and/or to copy/paste the token from your file after `oauth2_refresh_token=`, instead of using `oauth2_access_token_eval`.

## Emacs (or cli usage)

In my `init.el` I have:

```elisp
(require 'oauth2)

(cl-defmethod smtpmail-try-auth-method
  (process (_mech (eql xoauth2)) user password)
  (smtpmail-command-or-throw
   process
   (concat "AUTH XOAUTH2 " (shell-command-to-string (concat "python ~/code/email/oauth.py --authstr " user)))))


(add-to-list 'smtpmail-auth-supported 'xoauth2)
```

The complete cli:

```bash
python oauth.py USER@ACCOUNT # get and print authentication token
python oauth.py --authstr USER@ACCOUNT # get and print base64 encoded str for xoauth2
python oauth.py --refresh USER@ACCOUNT USER2@ACCOUNT2 # update refresh token for user.
```

# Credits
The office365 was rewritten from [M-365](https://github.com/UvA-FNWI/M365-IMAP).
Although I would have used `bottle`... but bottle 0.13 *still* isn't out, so the
server can't cleanly be stopped from a request... The differences are mostly
stylistic. The gmail code is straight from the docs.
