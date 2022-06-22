# apple-token-revoke-in-firebase

This document describes how to revoke the token of Sign in with Apple in the Firebase environment.<br>
In accordance with Apple's review guidelines, apps that do not take action by June 30, 2022 may be removed.<br>
A translator was used to create the document. Please forgive me if the sentence is weird.<br>
**This document uses Firebase's Functions, and if Firebase provides related functions in the future, use it.**

The whole process is as follows.
1. Get authorizationCode from App where user log in.
2. Get a refresh token with no expiry time using authorizationCode with expiry time.
3. After saving the refresh token, revoke it when the user leaves the service.

You can get a refresh token at https://appleid.apple.com/auth/token and revoke at https://appleid.apple.com/auth/revoke.

## Getting started

If you have implemented Apple Login using Firebase, you should have ASAuthorizationAppleIDCredential somewhere in your project.<br>
In my case, it is written in the form below.

```
  func authorizationController(controller: ASAuthorizationController, didCompleteWithAuthorization authorization: ASAuthorization) {
    if let appleIDCredential = authorization.credential as? ASAuthorizationAppleIDCredential {
      guard let nonce = currentNonce else {
        fatalError("Invalid state: A login callback was received, but no login request was sent.")
      }
      guard let appleIDToken = appleIDCredential.identityToken else {
        print("Unable to fetch identity token")
        return
      }
      guard let idTokenString = String(data: appleIDToken, encoding: .utf8) else {
        print("Unable to serialize token string from data: \(appleIDToken.debugDescription)")
        return
      }
      // Initialize a Firebase credential.
      let credential = OAuthProvider.credential(withProviderID: "apple.com",
                                                IDToken: idTokenString,
                                                rawNonce: nonce)
      // Sign in with Firebase.
      Auth.auth().signIn(with: credential) { (authResult, error) in
        if error {
          // Error. If error.code == .MissingOrInvalidNonce, make sure
          // you're sending the SHA256-hashed nonce as a hex string with
          // your request to Apple.
          print(error.localizedDescription)
          return
        }
        // User is signed in to Firebase with Apple.
        // ...
      }
    }
  }
  ```
What we need is the authorizationCode. Add the following code under guard where you get the idTokenString.

  ```
  ...
  
  guard let idTokenString = String(data: appleIDToken, encoding: .utf8) else {
    print("Unable to serialize token string from data: \(appleIDToken.debugDescription)")
    return
  }

  // Add new code below
  if let authorizationCode = appleIDCredential.authorizationCode,
     let codeString = String(data: authorizationCode, encoding: .utf8) {
      print(codeString)
  }
  
  ...
  
  ```
  
Once you get this far, you can get the authorizationCode when the user log in.<br>
However, we need to get a refresh token through authorizationCode, and this operation requires JWT, so let's implement it with Firebase functions.
Turn off XCode for a while and go to your code in Firebase functions.<br>
If you have never used functions, please refer to https://firebase.google.com/docs/functions.

  
